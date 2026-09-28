# Notification Service — Reference Architecture & Schema

A study reference for the HLD/LLD mock. Compare against your own attempt;
the "Where yours differed" notes at the end are the specific gaps to fix.

---

## 1. Requirements

**Functional**
- Send notifications across **PUSH, SMS, EMAIL**
- Caller (internal service) specifies **channel(s)** and **priority**
- **Priority tiers:** `TRANSACTIONAL` (OTP, payment confirm — urgent, bypass opt-out) vs `MARKETING` (promos — delay-tolerant, respect opt-out)
- **User preferences / opt-out** — filter channels the user disabled (legally required for marketing)
- **Deduplication** via client-supplied idempotency key
- **Retry** on delivery failure
- Users can **view notification history**
- No attachments in v1 (scoped out)

**Non-functional**
- Scale: ~10M DAU, ~3 notifications/user/day ≈ **30M/day ≈ ~350/s avg, ~3.5k/s peak** (10×)
  - (if 3 per channel: ~90M/day ≈ ~1k/s avg, ~10k/s peak — state which)
- Message size: **~1 KB** each (title + body + metadata)
- Storage: ~30 GB/day → ~60 TB over 5 years
- **Latency by priority:** transactional = seconds; marketing = minutes-tolerant
- **Availability:** high, especially for transactional (OTP down = users can't log in)
- **Delivery guarantee:** at-least-once (⇒ consumers must be idempotent)

---

## 2. High-Level Design

**Architecture diagram:** see `notification_service_architecture.svg` (same folder).

```
 Caller services
      │
 Load balancer
      │
 Notification API ──► notifications DB
   │  (dedup +        │  outbox table
   │   atomic write)  │
 Redis (dedup)        ▼
                   relay / CDC
                      │
             ┌────────┴────────┐
        Kafka: high       Kafka: normal
             └────────┬────────┘
                      ▼
            Processor / workers ──► prefs store (cached in Redis)
                      │  (prefs filter + retry)
             ┌────────┼────────┐
        Twilio SMS  FCM/APNs  SES email
                      │
        status callbacks → 2nd Kafka topic → update status (SENT → DELIVERED)
```

**Request flow (narrate this out loud):**

```
Caller → Load Balancer → Notification API
   API: check idempotency key in Redis (miss → check DB),
        ONE atomic DB txn: write notifications row + outbox row,
        store key in Redis, return 202 Accepted + notification_id
Relay / CDC: drains outbox → publishes to priority-split Kafka topics
Kafka: topic-high (drained first)  |  topic-normal
Workers/Processor: consume Kafka →
        filter channels vs preferences store (cached in Redis;
        TRANSACTIONAL bypasses opt-out) →
        call provider adapters with retry + backoff, dead-letter on repeat fail
Provider adapters: Twilio (SMS) | FCM/APNs (PUSH) | SES/SMTP (EMAIL)
Status callbacks: providers → 2nd Kafka topic → update status (SENT → DELIVERED)
```

**Components & why each is there**
- **Load balancer** — spread traffic across stateless API instances
- **Notification API** — dedup + atomic write; keeps ingestion fast, returns immediately
- **Redis** — idempotency-key dedup (TTL'd to 24h window) + preferences cache
- **notifications DB** — source of truth for the record + current status
- **outbox table** — atomic queue write (no dual-write bug)
- **relay / CDC** — drains outbox → Kafka (replaces DB polling; lower latency, scales)
- **Kafka (priority-split)** — high vs normal topics so OTP never waits behind a marketing blast
- **Workers/Processor** — preference filter + provider calls + retry
- **Provider adapters** — external, unreliable, rate-limited third parties (retry lives here)
- **Status callback path** — second topic to move status SENT → DELIVERED

---

## 3. Schema (LLD)

Each table: key columns + indexes + the **access pattern** it serves.

### notifications — core record
`id (PK)`, `user_id (FK)`, `channel`, `priority`, `header`, `body`,
`status` (INITIATED|SENT|DELIVERED|FAILED), `idempotency_key`, `created_at`
- **Index `(user_id, created_at DESC)`** → paginated history read (`getAllNotifications`)
- **Unique index `(idempotency_key)`** → dedup enforced at the DB (atomic insert fails on dup)
- **Index `(status)`** → find stuck/failed for retry sweeps

### outbox — transactional-outbox queue (written atomically with notifications)
`id (PK)`, `notification_id (FK)`, `payload`, `published (bool)`, `created_at`
- **Index `(published, created_at)`** → relay reads `WHERE published = false`

### users — recipient contact
`id (PK)`, `name`, `email`, `phone`, `push_token`
- PK lookup at send-time

### user_preferences — opt-out / channel prefs
`id (PK)`, `user_id (FK)`, `channel`, `opted_out (bool)`
- **Index `(user_id)`** → worker looks up prefs at send-time (cache in Redis)
- **Unique `(user_id, channel)`** → one row per user-channel

### notification_status_log — immutable audit trail (optional, strong)
`id (PK)`, `notification_id (FK)`, `status`, `provider_response`, `created_at`
- **Index `(notification_id, created_at)`** → full delivery history of one notification
- Same pattern as a ledger: current status on `notifications`, full history here

### Redis (data layer, not a table)
- `idempotency:{key}` → dedup, TTL'd to 24h window
- `prefs:{user_id}` → cached preferences (avoid DB hit at peak send rate)

**Two hot paths to name explicitly (scored):**
- **Write path:** atomic insert into `notifications` + `outbox` (one txn)
- **Read path:** `(user_id, created_at)` on `notifications` for history

---

## 4. Failure handling (name at least one + mitigation)
- **Provider (Twilio) down / times out** → retry with exponential backoff; after N tries → **dead-letter queue** for later replay/inspection. Status = FAILED, surfaced for retry.
- **Worker crashes mid-send** → at-least-once delivery may resend → **idempotent send** (provider idempotency key or dedup on notification_id) prevents double-send.
- **Kafka/outbox** → outbox row persists until relay confirms publish; nothing lost on crash.

## 5. Observability (cheap signal, don't skip)
- Monitor: **p99 latency per channel, error rate, throughput, queue depth, delivery success rate**
- Page on: delivery success rate drops below threshold, queue depth grows unbounded, provider error rate spikes

---

## Where YOUR attempt differed (the fixes to internalize)
1. **Preferences/opt-out filter** — you dropped it 3×. It lives in the worker at send-time; transactional bypasses it. Legal + stated requirement.
2. **Outbox table** — you designed the outbox *pattern* but left the table out of the schema. It's core.
3. **Priority queues** — added late; must be there from the start (OTP can't wait behind marketing).
4. **Provider adapters** — you had bare "SMS/PUSH" boxes; name them as external-provider adapters (that's where retry/rate-limiting live).
5. **Access pattern + index per table** — you listed tables with no indexes/access patterns. That's the scored 1.0. State the index and the query it serves for every table.
6. **DB-poll vs outbox-relay** — you had both, conflicting. Pick outbox→relay→Kafka (or justify polling explicitly).

**One-line takeaway:** your instincts on the hard parts (outbox, dedup, priority, status callbacks) were right — you lose points by dropping stated requirements and by naming tables/boxes without their access pattern or purpose. Same "state the mechanism, skip the consequence" gap, in design form.
