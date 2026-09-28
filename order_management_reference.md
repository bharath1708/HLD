# Order-Management System — Reference (Full Mock)

Study reference. Compare against your attempt; the "Where yours differed" notes
at the end are the specific fixes. This was your best mock (~74%) on the hardest
problem — the saga you drove yourself was a senior answer.

---

## 1. Requirements

**Functional**
- Place an order → moves through PAYMENT → INVENTORY RESERVE → FULFILLMENT → SHIPPING → DELIVERY
- **Order lifecycle / state machine** (the core requirement — name it upfront):
  `PLACED → INVENTORY_HELD → PAID → CONFIRMED → FULFILLING → SHIPPED → DELIVERED`
  (any stage can FAIL → roll back)
- **Reserve inventory so two customers can't buy the last item** (exclusive resource)
- Notify customer after each stage
- Product management, cancel/return out of v1 scope
- Idempotency handled by downstream payment provider (stated assumption)

**Non-functional**
- Scale: 10M DAU × 2 orders/day = **20M orders/day ≈ ~240/s avg, ~2,400/s peak** (10×)
  - reads (view orders/products) ~10× → ~2,400/s avg
- Storage: 500 B/order → ~10 GB/day → **~20 TB over 5 years**
- **CAP split (say this — don't say "HA and consistent" flat):**
  - **CP** for payment + inventory (no double-charge, no overselling)
  - **AP / eventual** for order status, tracking, notifications

---

## 2. High-Level Design

**Architecture diagram:** see `order_management_architecture.svg`.

**The core problem:** placing an order spans multiple services (payment, inventory,
fulfillment), each with its own DB — no shared ACID transaction. Coordinate with a
**SAGA**, orchestrated by the Order Manager, with **compensating transactions** on failure.

**The saga (orchestrated):**
```
Order Manager (orchestrator, owns sequence + state machine):

1. Reserve inventory   (atomic, held with TTL)
      fail (out of stock) → reject order  (nothing to compensate)
      success ↓
2. Charge payment
      fail → COMPENSATE: release inventory reservation
      success ↓
3. Confirm reservation (turn HOLD into committed stock decrement)
4. Persist order + write outbox row  (one atomic DB txn)
5. Relay → Kafka → Fulfillment → Shipping → Delivery
```

**Key mechanisms**
- **Order Manager = saga orchestrator** — owns the step sequence, the compensations,
  and the durable order state (so it can recover after a crash).
- **Atomic inventory reserve (CP):**
  `UPDATE inventory SET reserved = reserved + :qty
   WHERE product_id = :p AND (total_stock - reserved) >= :qty`
  Only one of two concurrent "last item" requests wins → no overselling.
- **Reservation TTL:** a hold auto-expires if the saga doesn't confirm in time
  → a crashed/abandoned order returns its stock (prevents stock leak).
- **Compensating transactions:** payment fails → release reservation; inventory
  fails after payment → refund. Undo-by-appending, never rollback across services.
- **Transactional outbox:** order write + fulfillment message committed atomically,
  relay publishes to Kafka → no lost fulfillment on a crash between DB and publish.

---

## 3. Schema (LLD) — query-first

```
orders
  id PK
  user_id       FK          ← whose order
  status        (PLACED | INVENTORY_HELD | PAID | CONFIRMED | SHIPPED | DELIVERED | FAILED)
  total_amount
  created_at
  → index (user_id, created_at DESC)   "a user's orders, recent first"
  → index (status)                     "find orders stuck mid-saga" (recovery/ops)

order_items                             ← FK lives HERE (one order, many items)
  id PK
  order_id      FK
  product_id    FK
  quantity
  price
  → index (order_id)                   "get all items for an order"

inventory
  product_id PK
  total_stock
  reserved                              ← held-but-not-confirmed (makes reserve atomic)
  → PK lookup + conditional UPDATE      "atomic reserve / confirm"
  (available = total_stock - reserved)
reservation:
  id, order_id, product_id, quantity,
  status        (HELD | CONFIRMED | RELEASED)
  expires_at    TIMESTAMP    ← e.g. now() + 15 minutes
outbox
  id PK
  order_id      FK
  payload
  published     BOOL
  created_at
  → index (published, created_at)       "relay reads unpublished"
```

**Killer queries → indexes (say these out loud — the scored part):**
- "get order with its items"     → orders PK + `order_items(order_id)`
- "a user's order history"        → `orders(user_id, created_at DESC)`
- "atomic inventory reserve"      → PK lookup + guarded conditional UPDATE
- "find stuck orders for recovery"→ `orders(status)`

---

## 4. Failure handling (name failure + mitigation)
- **Order Manager crashes mid-saga** (paid, not yet confirmed) → state machine persists
  each step; on restart, orchestrator finds orders stuck in intermediate states via
  `orders(status)` index and **resumes or compensates**.
- **Reservation held but order abandoned/crashed** → **TTL** auto-expires, stock returns.
- **Payment succeeds but fulfillment message lost** → **transactional outbox** guarantees
  the message committed with the order and eventually publishes.

## 5. Observability
- **Monitor:** order success rate (full-saga completion), payment failure rate,
  saga step latencies, **count of orders stuck in intermediate states** (key — jammed saga),
  reservation-expiry rate, Kafka/fulfillment queue depth.
- **Page on:** order success rate drops, stuck-order count grows, payment failure spikes,
  queue depth unbounded.

---

## Where YOUR attempt differed (fixes)
1. **HLD — strong.** You drove the saga + compensation (release reservation on payment
   fail) yourself. Add the **confirm step** (hold → committed) and the **reservation TTL**
   without a nudge.
2. **Requirements:** apply **peak** (×10), and split **CAP** explicitly (CP payment/inventory,
   AP status) instead of "HA and consistent." Name the **state machine** upfront.
3. **Schema:** FK goes on `order_items` (order_id), NOT `order_items_id` on orders
   (one-to-many → FK on the many side). Add `user_id` to orders. Add `reserved` to
   inventory (without it the atomic reserve can't work). And **lead with the access
   pattern + index** — query-first, like the schema drill.
4. **Observability ≠ product analytics.** Monitor system health (success rate, stuck
   orders, latencies), not "which pages users visit / fraud."

**One-line takeaway:** best mock yet, on the hardest problem — the saga was a senior
answer you drove yourself. The remaining gap is delivering the **LLD query-first** and
the right answers **first-try without a redirect**. That's reps, not learning.
