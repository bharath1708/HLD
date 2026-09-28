# Seat Booking — Where Does the HELD State Live? (Two Designs)

The key design decision in seat booking: when a user holds seats (before payment),
where does that transient HELD state live — the DB, or Redis? Both are valid; know the
tradeoff and be ready to defend your pick (interviewers score "name the fork + justify").

---

## Design A — DB write at hold time (traditional saga)

The Booking Orchestrator writes to the DB upfront, then transitions the status on payment.

```
1. User selects seats {J-1, J-2}
2. Orchestrator writes to DB:
     booking row           → status = HELD
     seat_allocation rows  → one per seat (J-1, J-2)
3. Update Redis seat status → HELD (for the live seat map)
4. Payment response:
     success → update booking status = BOOKED  (+ Redis → BOOKED)
     failure → update booking status = FAILED   (+ release Redis)
```

**Pros**
- Durable booking lifecycle from the start — a crash leaves a DB record ("was HELD,
  awaiting payment") → recovery is explicit: scan for stuck HELD bookings (status index).
- Full audit trail — failed/abandoned attempts are visible in the DB.
- One source of truth (DB) for booking state; Redis is just the fast seat-map layer.

**Con**
- DB write on EVERY hold — including holds that never pay. During a drop this is a lot of
  booking + seat_allocation writes (the spike the Redis-only design avoids).

**Defense against the write-spike concern:**
- The **waiting room** throttles admission (~1,000/s reach the booking flow), so the DB
  sees the admitted trickle, not the full 100K/s spike. This protects the DB writes too.
- Or make the hold-write async (write after the Redis claim succeeds) so the user's claim
  isn't blocked on the DB.

---

## Design B — Redis-only hold, DB only on confirm

HELD state lives ONLY in Redis (native TTL + atomic). The DB gets a row only on payment
success (BOOKED).

```
1. User selects {J-1, J-2}
2. Atomic Lua claim in Redis: both AVAILABLE → both HELD (TTL 5min)
     store pending booking in Redis: hold:{hold_id} → {user, event, seats, expires_at}
3. [between reserve and payment — everything lives in Redis]
4. Payment response:
     fail / TTL expire → Redis auto-releases seats + drops hold data → NOTHING in DB
     success → write durable booking + seat_allocation (BOOKED) + Redis → BOOKED
```

**Pros**
- DB write rate low — only confirmed bookings hit the DB → survives the thundering herd.
- Compensation is FREE — payment failure/timeout just lets the TTL expire; nothing was
  persisted, so there's nothing to roll back.
- Hot, short-lived HELD churn stays in Redis where it belongs (native TTL, atomic claims).

**Cons**
- HELD state is volatile (Redis-only) until confirmed.
- Less audit trail (only successful bookings persisted).
- Need a safety net: on Redis crash, rebuild live seat status from the DB's confirmed
  bookings (BOOKED is durable in DB; HELD seats just reset to AVAILABLE — fine).

---

## The tradeoff at a glance

| | A: DB write at hold | B: Redis-only hold |
|---|---|---|
| Booking row created | at hold (HELD) | at confirm (BOOKED) |
| DB write rate | high (every hold) | low (only confirmed) |
| Failure cleanup | update row → FAILED + release Redis | TTL auto-expires, nothing to clean |
| HELD durability | durable in DB | Redis-only (volatile) |
| Audit trail | full (all attempts) | successful only |
| Thundering-herd DB load | higher (mitigated by waiting room) | lower |
| Recovery | scan DB for stuck HELD | rebuild Redis from DB (BOOKED) |

---

## How to answer in the interview

Pick one and justify with the tradeoff:

> "I'd write the booking + seat_allocation at hold time with status HELD, update Redis for
> the live seat map, then transition to BOOKED or FAILED on the payment response. That
> gives a durable booking lifecycle and simple recovery — I can scan for stuck HELD
> bookings. The tradeoff is a DB write per hold, but the waiting room throttles admission
> so the DB only sees the admitted trickle, not the full spike. If write load were still a
> concern, I'd move the hold state to Redis-only and persist only on confirm — that makes
> the DB spike-free and compensation automatic via TTL, at the cost of a volatile HELD
> state I'd rebuild from the DB on a Redis crash."

That names BOTH designs, the tradeoff axis (durability/audit vs write-load), and a
condition for switching — which is what scores.

---

## Multi-seat booking note
Within ONE booking, the seats (J-1, J-2) are claimed atomically together (all-or-nothing)
and always succeed/fail together — so the booking record carries a SINGLE status covering
all its seats. Per-seat status lives in Redis (for the map + atomic claim); the booking
record needs only one status because the seats share the same fate. This only holds
because the multi-seat claim is atomic (one Lua op holds all or none) — if you held seats
one-by-one, they could diverge and one booking-status wouldn't work.

## Schema (Design A shape)
```
booking:  id PK, user_id FK, event_id FK, status (HELD|BOOKED|FAILED), created_at
          → index (user_id, created_at DESC)   "a user's bookings, recent first"
          → index (status)                      "find stuck HELD bookings (recovery)"

seat_allocation:  id PK, booking_id FK → booking.id, seat_id FK → seats.id
          → index (booking_id)                 "seats in a booking"
          (NOTE: FK must reference booking.id — keep table name consistent, not 'ticket' vs 'booking')
```
