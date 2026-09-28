# News Feed (Twitter/Instagram) — Reference (Full Mock)

Study reference (~75%, your best). HLD was excellent — you drove the fan-out hybrid
(the hardest part) yourself. Gap: LLD indexes not on all tables + phantom sort column.

---

## 1. Requirements

**Functional**
- User creates a post (with optional file → **S3**, DB stores a reference)
- User opens feed → recent posts from people they follow, most recent first
- Follow/unfollow (IN scope — the relationship is core to the whole design)
- On post, followers' feeds are updated
- Users may have millions of followers (celebrity case)
- Login/privacy out of v1

**Non-functional**
- 100M DAU; ~10M posts/day ≈ **~115 writes/s**
- **Read-heavy ~100:1** — each feed-open assembles many posts, users open often.
  *This ratio is WHY fan-out-on-write exists — say it: reads are the thing to optimize.*
- **Write amplification is the real cost:** 115 posts/s, but each post fans out to N
  followers → actual writes = 115 × avg_followers. The celebrity problem in disguise.
- Storage: 10M × 1KB = 10 GB/day → **~28 TB over 7 years** (files in S3, separate)
- AP / eventual consistency is fine (a post appearing 2s late in a feed is OK)
- Low latency on feed reads

---

## 2. High-Level Design — the hybrid fan-out (the heart of the problem)

**Diagram:** see `news_feed_architecture.svg`.

**The question:** feed-open = "recent posts from everyone I follow, sorted." Assemble fast?

**Two approaches:**
- **Fan-out on READ (pull):** at feed-open, query posts from all N people you follow,
  merge, sort. Cost is at READ time → slow for users following many people, and reads
  dominate → bad default.
- **Fan-out on WRITE (push):** when you post, push the post into every follower's
  precomputed feed (Redis `feed:{user}`). Feed-open = just read your precomputed feed →
  FAST reads. Cost moves to WRITE time. Good, because reads dominate 100:1.

**The celebrity problem (what interviewers push on):**
Fan-out-on-write breaks for a user with 50M followers — one post = 50M feed writes
(a write storm). So:

**THE HYBRID (the senior answer):**
- **Normal users (< threshold, e.g. 100K–1M followers)** → **fan-out-on-write** (push to
  followers' precomputed feeds).
- **Celebrities (> threshold)** → **NOT** fanned out (would storm) → their posts are
  **pulled at read time**.
- **Feed-open = read precomputed feed (push) + live-pull recent celebrity posts + MERGE +
  sort by time.** 99% comes from the fast precomputed feed; a small live pull covers the
  handful of celebrities you follow. Fast reads, no write storm.

**Details to state:**
- Precomputed feed = per-user Redis list/sorted-set `feed:{user}`, **capped** at ~recent
  few hundred post refs (older → paginate from posts DB, don't precompute infinite history).
- "Celebrity" = a follower-count threshold; a tuning knob, set where fan-out cost gets painful.
- Fan-out is **async** (post returns immediately; fan-out happens in the background via a queue).

---

## 3. Schema (LLD) — index derived from the query, on EVERY table

```
users
  id PK, name, phone, email
  → unique index (phone)              "authenticate by phone"

follows                               ← queried BOTH directions → TWO indexes
  id PK, follower_id FK, followee_id FK, created_at
  → index (follower_id)               "who do I follow" (build my feed)
  → index (followee_id)               "who follows me" (fan-out on post)
  → unique (follower_id, followee_id) prevent duplicate follows

posts
  id PK, post_by FK, content, file_url, created_at   ← created_at needed for sort
  → index (post_by, created_at DESC)  "recent posts by a user"
```

**Two LLD lessons (the gaps):**
1. **Index EVERY table, not just the star table.** `follows` is the important one here —
   it's queried two ways (who-I-follow for feed-building, who-follows-me for fan-out) →
   needs BOTH `(follower_id)` and `(followee_id)`. Don't skip the "supporting" tables.
2. **The sort column in an index must be a REAL column.** `index (post_by, created_at DESC)`
   requires a `created_at` column to exist. Derive index from query → the query names the
   filter (post_by) AND the sort (created_at) → BOTH become columns. (Phantom-column bug:
   writing the index but forgetting to add the column it sorts on.)

---

## 4. Failure handling
- **Fan-out partially fails** (service crashes mid-fan-out to 500K followers → 200K got it,
  300K didn't) → make fan-out **resumable + idempotent** (track progress, retry from where
  it stopped; idempotent writes so re-run doesn't duplicate). The **read-time celebrity/pull
  path also covers gaps** — a missing post can be caught on live pull.
- **Redis feed cache lost** → rebuild `feed:{user}` from the posts DB (source of truth).
- Redis clustered with replicas for HA.

## 5. Observability
- **Monitor:** feed-load latency (read path), **fan-out lag / backlog** (how long until a
  post reaches followers' feeds), post-creation rate, feed cache hit ratio.
- **Page on:** fan-out lag grows unbounded (a celebrity post backing up the queue),
  feed-load latency breaches SLO.

---

## Where YOUR attempt differed (fixes)
1. **HLD — excellent, yours.** Fan-out-on-write, celebrity problem, AND the hybrid — driven
   yourself. Add: state the read-time **merge** (precomputed + celebrity pull), keep
   follow/unfollow IN scope (it's core), and the capped-Redis-feed structure.
2. **Requirements:** draw the conclusion from your numbers — reads dominate ~100:1 (that's
   WHY fan-out-on-write), and the write amplification (post × followers) is the real cost.
3. **LLD — improving but not complete.** You derived the `posts` index (new — good!), but
   left `users` and `follows` unindexed, and `follows` is where the interesting
   bidirectional indexing lives. And the phantom `created_at` recurred. Fix: EVERY table
   gets a derived index, and the index's sort column must be a real column.
4. **Failure:** name the news-feed-specific failure (fan-out partial failure), not just the
   generic "Redis cluster."

**One-line takeaway:** best mock yet, hardest HLD section fully yours. Trajectory across
six mocks: HLD rock-solid every time; the ceiling is the LLD index reflex, which is
visibly closing — now down to "index ALL tables" + "sort column must exist."
