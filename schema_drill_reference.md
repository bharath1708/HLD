# Schema Drill — Reference Solutions

Four LLD schema reps. Each table lists columns, PK/FK, and — the scored part —
**the access pattern + the index that serves it.**

**The one rule to carry:** design the index FROM the killer query, not from the
entity. Say the query out loud first, then index exactly what it filters (equality)
and sorts/ranges on. Composite index order = equality columns first, then sort/range.

---

## Rep 1 — Todo / Task app (multi-tenant)

Every tenant-scoped table carries `org_id`. One-to-many = FK column (no join table).

```
organizations
  id PK
  name

users
  id PK
  org_id      FK
  name
  email
  → index (org_id)            "all users in an org"
  → unique index (email)

projects
  id PK
  org_id      FK
  name
  → index (org_id)            "all projects in my org"

tasks
  id PK
  org_id      FK              ← multi-tenancy
  project_id  FK              ← one project (no join table)
  assigned_to FK → users.id   ← one assignee (no join table)
  title
  status      (TODO | IN_PROGRESS | DONE)
  due_date
  created_at, updated_at, created_by
  → index (assigned_to, status)   "my tasks by status"
  → index (project_id, status)    "tasks in a project by status"
  → index (org_id, due_date)      "org tasks due soon"
```

Lesson: don't reach for join tables on one-to-many — use a FK column.
Join table only for genuine many-to-many.

---

## Rep 2 — Hotel / Room booking

Killer query: "find available rooms in hotel X for date range Y."

```
hotels
  id PK
  name
  address

rooms
  id PK
  hotel_id    FK
  room_number
  room_type
  floor_number
  → index (hotel_id, room_type)   "rooms in hotel X of type Y"

bookings
  id PK
  room_id     FK
  user_id     FK
  start_date
  end_date
  status      (CONFIRMED | CANCELLED)
  → index (room_id, start_date, end_date)   "overlap check for a room"
```

**The overlap condition (memorize — bookings, calendars, Meeting Rooms II):**
```
two ranges overlap  ⇔  existing.start < req_end  AND  existing.end > req_start
```
Derived by negating the only two NON-overlap cases:
- ends before you start:  existing.end <= req_start
- starts after you end:   existing.start >= req_end

Availability query:
```sql
SELECT r.* FROM rooms r
WHERE r.hotel_id = :hotel
AND NOT EXISTS (
    SELECT 1 FROM bookings b
    WHERE b.room_id = r.id
    AND b.start_date < :req_end
    AND b.end_date   > :req_start
    AND b.status = 'CONFIRMED'
);
```
Use strict `<`/`>` if checkout-day = checkin-day is allowed (no conflict);
use `<=`/`>=` if touching endpoints should conflict.

---

## Rep 3 — Chat / Messaging app

Key insight: **a 1-on-1 chat is just a group of 2.** One conversation model,
one membership table — NOT separate direct/group tables.

Killer query: "load the most recent 50 messages in a conversation, paginated."

```
users
  id PK
  name
  email

conversations
  id PK
  type        (DIRECT | GROUP)
  name        (null for DMs, set for groups)
  created_at

conversation_participants          ← handles BOTH DM and group
  id PK
  conversation_id  FK
  user_id          FK
  → index (user_id)                "which conversations am I in"
  → index (conversation_id)        "who's in this conversation"
  → unique (conversation_id, user_id)

messages                           ← the hot table
  id PK
  conversation_id  FK
  sender_id        FK → users.id
  content
  created_at
  → index (conversation_id, created_at DESC)   ← THE killer index
```

Why the membership table: a DM = 2 rows, a group = N rows, same schema.
`from_user`/`to_user` columns can't represent a 50-person group — membership can.

Killer index `(conversation_id, created_at DESC)`:
```sql
-- recent 50
SELECT * FROM messages
WHERE conversation_id = :cid
ORDER BY created_at DESC LIMIT 50;

-- load older (KEYSET pagination — not OFFSET)
SELECT * FROM messages
WHERE conversation_id = :cid AND created_at < :oldest_seen
ORDER BY created_at DESC LIMIT 50;
```
Use **keyset pagination** (`WHERE created_at < last_seen`), never `OFFSET 50000`
— keyset stays fast on a huge messages table forever.

---

## Rep 4 — Leaderboard / Gaming scores

Killer queries: "top 10 players for game X" and "what's my rank in game X."
This rep is entirely about the index + the serving layer.

```
users
  id PK
  name

games
  id PK
  name

game_participants                  ← durable source of truth
  id PK
  game_id     FK
  user_id     FK
  score
  → index (game_id, score DESC)    ← index the SORT column, not the FK
```

Index on `(game_id, score DESC)`, NOT `(game_id, user_id)`:
```sql
-- top 10: no sort, walk the front of the index
SELECT * FROM game_participants
WHERE game_id = :gid ORDER BY score DESC LIMIT 10;
```

**Rank at scale = Redis Sorted Set (ZSET)**, with SQL as source of truth:
- `ZADD game:X score user_id`      update ranking, O(log n)
- `ZREVRANGE game:X 0 9`           top 10, pre-sorted, instant
- `ZREVRANK game:X user_id`        my rank, O(log n), no counting

Computing rank by `COUNT(*) WHERE score > mine` works but doesn't scale to
millions of players — the sorted set maintains the ordering for you.
Pattern: durable SQL store + fast Redis serving layer (same shape as the
derived-balance cache and the URL-shortener cache).

---

## The through-line (today's fix)

Your instinct is **entity-first** (list tables, then columns). It needs to be
**query-first**: say the killer query, then design the table + index that serves it.

- Rep 1 retry: query-first → 3 correct indexes with correct column order. ✓
- Rep 3: entity-first → forgot the `messages` table (the whole point).
- Rep 4: entity-first → indexed the FK instead of the sort column.

**Before writing any index: state the killer query out loud, index what it
filters (equality first) and sorts/ranges on (last).**
