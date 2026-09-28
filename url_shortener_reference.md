# URL Shortener — Reference Architecture & Schema

Study reference for the HLD/LLD mock. Compare against your own attempt;
the "Where yours differed" notes at the end are the specific gaps to fix.

---

## 1. Requirements

**Functional**
- Shorten a long URL → short URL
- Redirect short URL → original long URL
- View **analytics** (click counts per link)
- No user registration in v1 (scoped out)
- No custom aliases in v1 (scoped out)
- No rate limiter in v1 (scoped out)
- Links are permanent (no expiry in v1)

**Non-functional**
- **Read:write ratio ≈ 10:1 → read-heavy** (redirects ≫ creations). *This drives the whole design.*
- Scale: ~100M new URLs/day, ~10 redirects per URL
  - Writes: 100M/day ≈ **~1,200/s avg**, ~12k/s peak (10×)
  - Reads: 1,000M/day ≈ **~12,000/s avg**, ~120k/s peak (10×)
- **Redirect latency: < 10ms** → at 120k/s peak, this CANNOT be DB-served → **redirects must come from cache**
- **Availability: high** — if it's down, every short link everywhere breaks
- Storage: 500 B/record → 50 GB/day → ~60 TB over 3 years

**The two requirements that shape everything:** 10:1 read-heavy + <10ms latency
→ **redirect path is cache-served, not DB-served.** Say this connection out loud.

---

## 2. High-Level Design

**Architecture diagram:** see `url_shortener_architecture.svg` (same folder).

```
            Client
              │
        Load balancer
              │
      URL-shortener service
        │            │
   (write path)  (read path)
        │            │
   Snowflake ID   Redis cache ──(miss)──► DB
   → base62       (short_code → long_url)
   → store DB              │
   + cache            302 redirect
```

**Write path (create short URL):**
1. Long URL comes in
2. **Snowflake** generates a unique 64-bit ID (timestamp + machine-id + sequence)
   — collision-free by construction, no lookup/check needed
3. **base62-encode** the ID → short code
4. Store `(short_code → long_url)` in DB, populate cache
5. Return short URL

**Read path (redirect):**
1. Short code comes in
2. Check **Redis** first (cache-first — justified by 10ms + read-heavy requirement)
3. Miss → check DB → populate cache
4. Not found → **404**
5. Found → **302 redirect** to long URL

**Key design decisions & why**
- **Short-code generation — Snowflake + base62.** Collision-free by design → fast,
  lookup-free writes. Tradeoffs: codes ~11 chars (vs 7 strictly needed), somewhat
  time-ordered/guessable, depends on machine clocks.
  - Alternatives: **hash(url) + collision check** (shorter, non-sequential, dedups
    naturally, but must handle collisions); **global counter + base62** (7-char codes,
    but a single counter is a bottleneck → needs range allocation).
- **Sizing:** 100M/day × 3yr ≈ 100B URLs. 62^7 ≈ 3.5 trillion → **7 base62 chars suffice.**
  (Snowflake gives ~11; state that you know 7 is enough.)
- **301 vs 302 → use 302 (temporary).** 301 lets the browser cache the redirect and
  skip your server → you LOSE analytics + can't change target. 302 = every click hits
  your server → **analytics work + updatable target.** Since analytics is a requirement,
  302 is correct. *Justify it via the analytics requirement.*
- **Dedup vs Snowflake tension** (good insight): Snowflake IDs are content-independent
  → same URL gets different codes → **Snowflake can't dedup naturally.** Hashing dedups
  naturally but has collisions. Default: don't dedup (keep writes lookup-free). If storage
  matters: add an index on `hash(long_url)` and return existing code (read on each write).

---

## 3. Schema (LLD) — the part to nail

Each table: columns + **access pattern + index it serves**.

### urls — the core mapping
```
short_code    PK   (base62 of Snowflake id) — or id BIGINT PK, short_code unique
long_url           TEXT
created_at         TIMESTAMP
created_by         (nullable — client/user id if auth added later)
```
- **PK = short_code** (or Snowflake id with a unique index on short_code)
- **Redirect lookup: point lookup by short_code** — the hottest read path, cache-first (Redis),
  DB fallback on PK. This is O(1).
- Optional: index on `hash(long_url)` ONLY if you dedup identical URLs

### click_events — analytics (SEPARATE table, not the urls row)
```
id            PK
short_code    FK → urls.short_code
clicked_at    TIMESTAMP
ip / country / referrer / user_agent   (whatever analytics you track)
```
- **Why separate:** you write a click on EVERY redirect (~120k/s peak). You do NOT want
  to update the `urls` row you're reading 120k/s — that's write contention on your hot
  read path. Append click events to a separate table/stream instead.
- Better at scale: don't even write synchronously — **emit the click to a queue/Kafka**,
  aggregate asynchronously into counts. Redirect path stays fast (just cache read + 302).
- Access pattern: aggregate by short_code + time window → index `(short_code, clicked_at)`

### Redis (data layer)
- `short_code → long_url` — serves the redirect at <10ms (the whole point)
- Optionally cache hot analytics counters

**Two hot paths to name:**
- **Write path:** Snowflake → base62 → insert urls + cache (lookup-free)
- **Read path:** point lookup by short_code, cache-first (Redis → DB)

---

## 4. Failure handling (name at least one + mitigation)
- **Redis (cache) down** → redirect path falls back to DB. Slower, but still works
  (degrades, doesn't fail). DB must be able to absorb the read burst → read replicas.
- **DB down** → redirects for cache-hits still work (served from Redis); cache-misses 503.
  → high replication, read replicas for the read-heavy load.
- **Snowflake clock skew / clock moves backwards** → generator refuses to issue IDs until
  clock catches up (prevents duplicate IDs).
- **Hot key** (one viral link) → Redis handles it; consider replicating hot keys.

## 5. Observability
- Monitor: **redirect p99 latency, cache hit ratio, error rate (404/503), redirect QPS,
  DB replica lag**
- Page on: cache hit ratio drops (cache issue), p99 latency breaches 10ms SLO,
  error rate spikes, DB replica lag grows

---

## Where YOUR attempt differed (fixes to internalize)
1. **Schema not delivered** — you circled it and stopped. This is 1.0 of 5.0, and the same
   section you skipped in Mock 1. THE hole to close.
2. **Analytics table** — you'd need to decide same-row vs separate; separate (or async via
   queue) is correct because you write a click on every redirect at 120k/s.
3. **7-base62-char sizing** — you reached for Snowflake but didn't do the "62^7 ≈ 3.5T,
   so 7 chars" math out loud. That calc is the estimation signal.
4. **Connect latency + read-heavy → cache** — you stated both facts but didn't say the
   implication out loud. The connection is what scores.
5. **302-via-analytics justification** — you picked 302 (right!) but justify it with the
   analytics requirement, not just "temporary."
6. **Named failure + mitigation, and observability** — not reached (stopped early).

**One-line takeaway:** strong top half (requirements asserted + smell-tested, HLD coherent,
Snowflake reached-for and defended, dedup-tension spotted yourself). The score gap is
entirely the **schema you avoid** and the second-half connections you leave implicit.
Same pattern as Mock 1 — drilling problem, not a knowledge problem.
