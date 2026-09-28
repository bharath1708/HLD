# YouTube Comment Section — HLD Reference

Defining trait: MASSIVELY READ-HEAVY (millions read a viral video's comments, few write).
The design is driven by that asymmetry + the reply tree + top/newest sorting.

---

## 1. Requirements

**Functional**
- Comment on a video; reply to a comment (nested); delete own comment (cascade deletes children)
- Sort by TOP (likes) or NEWEST
- Pagination (can't load 100K comments at once)
- Out of v1: moderation/admin controls, abuse/word filtering, non-text content

**Non-functional**
- 10M DAU; comment ~500 bytes
- Writes ~120 TPS avg; **reads ≫ writes** — the defining asymmetry (100x+ → ~12k+ QPS,
  concentrated on hot videos). THIS drives the design (cache + read replicas).
- Storage: 500B × 10M/day = 5 GB/day → ~2 TB/year → **~10 TB / 5 years**
  (units consistent: 500 bytes, 5 GB/day — smell test agrees)
- **CAP = AP** (NOT CP). A comment isn't an exclusive resource — no "two people can't
  have the same comment" conflict. Availability > strict consistency: a comment showing
  up a second late everywhere is fine; a "comments unavailable" error is very visible.
  Contrast seat booking (CP — can't double-book). "Don't lose comments" = durability
  (replication), NOT consistency. Ordering = the sort key, not CAP.

---

## 2. API Contract

```
POST /videos/{video_id}/comments
  { "parent_comment_id": null | "comment-123", "text": "..." }
  → 201 { "comment_id": "comment-456", "status": "CREATED" }   (return the new id)
  → 404 { "error": "VIDEO_NOT_FOUND" }
  (parent_comment_id null = top-level, set = reply — one endpoint handles both)

GET /videos/{video_id}/comments?sort=top|newest&limit=20&after=<cursor>
  → top-level comments: [{ comment_id, text, author, like_count, reply_count }]
  → next_cursor
  (return reply_COUNT, not the replies themselves)

GET /comments/{comment_id}/replies?limit=10&after=<cursor>
  → paginated replies for one comment (loaded on demand when user expands)
```

**The tree-pagination insight:** you do NOT return the whole nested tree in one response
(a viral comment could have 100K replies). Two-level pagination:
- top-level comments paginated, each with a reply COUNT
- replies loaded separately, on demand ("View 47 replies" → separate paginated call)
This is how YouTube actually works.

**Other API points:**
- `sort=top|newest` param (a functional requirement — don't omit it)
- CURSOR-based pagination (`after=<cursor>`), not offset (`page=`) — avoids the
  shifting-window problem as new comments arrive in a live feed
- YouTube flattens to ~2 levels (top-level + replies); arbitrary nesting is complex to
  paginate/render — capping depth is simpler and usually enough (name the tradeoff)

---

## 3. High-Level Design

```
Actor → LB → API Gateway
                 │
     ┌───────────┴───────────┐
   WRITE                    READ
     │                        │
Comment Service          Comment Service
  write directly           check Redis (cache-aside)
  to Cassandra              miss → Cassandra → populate cache
     │                        │
     ├─ emit event → Kafka   └─ return
        (async fan-out:
         notify owner,
         update counts,
         spam check [v1-])
```

**Write path — SIMPLE, direct (not a saga):**
Posting a comment is ONE write — no multi-step coordination. So:
- API Gateway → Comment Service → **write directly to Cassandra** (synchronous, immediate —
  the user sees their comment right away).
- THEN emit an event to Kafka for **async side-effects**: notify video owner, update
  comment/reply counts, run spam/abuse filters, analytics.
- Do NOT route the core write through an orchestrator + Kafka — there's no saga here.
  Orchestrator/Kafka-for-the-write is over-engineering (unlike order/booking, which DO
  need a saga). Kafka's right role here is the async fan-out AFTER the write.

**Read path — the main challenge (read-heavy):**
- **Cache-aside with Redis** — hot videos' comment pages cached; check Redis first, miss →
  Cassandra → populate. Most reads hit cache.
- **Read replicas** for Cassandra to spread the read load.
- Cache the first page(s) of top/newest for hot videos (the 99% case — most people read
  page 1).

**Datastore — Cassandra:**
- Write-scalable, handles huge comment volume, horizontal scale — good fit for read+write
  at this scale.
- **Cassandra is query-first modeled:** partition by video_id (a video's comments
  co-located), cluster by created_at (newest) — and a separate table/clustering for
  top-by-likes. NOT relational indexes bolted on.
- **Hot-partition risk:** a viral video = one huge partition → bucket it (e.g.
  video_id + time-bucket as the partition key) to spread load.

---

## 4. Data model

```
videos:
  id PK, title, description, url, channel_id FK, created_at
  → index (channel_id, created_at DESC)   "a channel's videos"

comment:
  id PK
  video_id          FK → videos.id           ← the comment belongs to a video (don't omit!)
  parent_comment_id FK → comment.id (nullable: null=top-level, set=reply)
  text
  commented_by      FK → users.id
  like_count        (denormalized counter — for the "top" sort)
  created_at
  → index (video_id, created_at DESC)      "top-level comments for a video, newest"  ← MAIN query
  → index (video_id, like_count DESC)      "top comments for a video, by likes"       ← "top" sort
  → index (parent_comment_id, created_at)  "replies for a comment"

comment_likes:   (if tracking who liked)
  id PK, comment_id FK, user_id FK
  → unique (comment_id, user_id)           "prevent double-like"
```
Cassandra equivalent: partition on video_id, cluster on created_at (or like_count) — model
per query, one table per access pattern.

---

## 5. Deep-dive points (if asked)
- **"Top" freshness:** like_count changes constantly on a viral comment → sorting by top
  can't re-sort the whole set per read. Options: periodically recompute top-N and cache it,
  or approximate (top is "good enough," not exact real-time). Exact real-time top-sort at
  scale is expensive — cache a periodically-refreshed top list.
- **Cascade delete:** deleting a comment deletes its subtree. In Cassandra (no FK cascade),
  handle in the service — delete the comment and its descendants (or soft-delete/tombstone).
- **Hot video:** viral video's page read millions of times → cache page 1 aggressively;
  bucket the partition to avoid a single hot Cassandra partition.
- **Counts:** reply_count / like_count denormalized (updated async via Kafka) so reads don't
  aggregate on the fly.

---

## Key takeaways
- Read-heavy → cache-aside + read replicas; cache hot videos' first pages.
- **AP, not CP** — comments aren't exclusive resources; availability > strict consistency.
- **Simple direct write** (no saga); Kafka only for async fan-out after the write.
- **Tree pagination** — paginate top-level with reply counts; load replies on demand.
- Cursor pagination (not offset) for a live feed.
- comment table MUST have video_id + (video_id, created_at/like_count) indexes.
