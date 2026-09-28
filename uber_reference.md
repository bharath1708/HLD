# Ride-Sharing (Uber) — Reference (Breadth Mock)

Study reference (~71%). New concept: geospatial (find nearby drivers fast). You drove
the geohash + firehose + expanding-radius yourself — the hard part. Gap: LLD indexes
not derived inline, and the location-ping firehose missed in requirements.

---

## 1. Requirements

**Functional**
- Rider requests a ride (from → to)
- System matches rider to a *nearby available* driver
- Driver accepts/declines; on decline or no-driver → offer next / expand radius
- Real-time location tracking (rider watches driver approach; both tracked through trip)
- Surge pricing, ratings out of v1

**Non-functional**
- Ride requests: 100M/day ≈ **~1,200/s**
- **THE dominant load — driver location pings:** ~1M active drivers × 1 ping / 4s
  = **~250,000 location updates/sec** (~200× the ride-request rate). *This defines the design.*
- Storage: 1KB/trip × 100M/day = 100 GB/day → 40 TB/year → **200 TB over 5 years**
  (200,000 GB = 200 TB — watch the GB→TB conversion, 1 TB = 1,000 GB)
- Latency: match + live-location must be low (seconds)
- High availability

---

## 2. High-Level Design

**Diagram:** see `uber_architecture.svg`.

**The core challenge: "find available drivers within ~2km of this point, fast."**
A normal B-tree index on (lat, lng) does NOT do 2D proximity well.

**Solution — Geohashing (the key insight):**
- Geohash encodes a 2D (lat, lng) into a **1D string where nearby points share a prefix**
  (e.g. `dr5ru7` and `dr5ru2` are close — shared `dr5ru`).
- This turns "find points near this 2D location" (hard) into "find keys with this prefix"
  (easy — a normal index does prefix matching).
- Store each driver's location in Redis keyed by geohash cell; on a request, compute the
  rider's geohash and look up drivers in the **same cell + its 8 neighbors** (drivers just
  across a cell boundary are still nearby — the classic geohash gotcha).
- Alternatives to name: **Redis GEO** (`GEOADD`/`GEOSEARCH`, native), quadtree, S2.

**The location firehose (~250k/s):**
- GPS pings → Kafka → consumed → update **current** location in **Redis GEO** (short TTL).
- You do NOT write every ping to SQL. Redis holds current position; trip *route history*
  is sampled/batched to durable storage async — not every ping.

**Match flow:**
```
Rider requests (API) → Matching service
  → geohash query in Redis: available drivers in rider's cell + 8 neighbors
  → offer to nearest driver → accept? remove from pool, allocate, start trip
                             → decline / timeout? offer next
  → none nearby? EXPAND the search radius (wider geohash prefix / more cells)
  → trip persisted to trips DB (durable); WebSockets stream live location during trip
```

**Separation to state:** ride request = API (transactional). WebSocket = live tracking.
Current location = Redis (ephemeral, TTL). Trip lifecycle/state = durable DB.

---

## 3. Schema (LLD) — query-first, index INLINE (derived from the query)

```
riders
  id PK, name, phone
  → unique index (phone)                  "authenticate rider by phone"

vehicles
  id PK, license_plate, make, model

drivers
  id PK, name, vehicle_id FK, status (AVAILABLE|ON_TRIP|OFFLINE)
  → index (status)                        "available drivers" (DB fallback;
                                           real geo-matching is in Redis, not here)

trips
  id PK, rider_id FK, driver_id FK,
  from_lat, from_lng, to_lat, to_lng,
  status, fare, created_at                ← created_at needed for "recent first"
  → index (rider_id, created_at DESC)     "a rider's trip history, recent first"
  → index (driver_id, created_at DESC)    "a driver's trips / earnings"
  → index (status)                        "find ongoing trips" (recovery/ops)
```

**LLD lesson (the gap):** the index is DERIVED from the killer query — the query names
the filter column AND the sort column, so the column and index come as a pair.
"rider's trips, recent first" → needs rider_id (filter) + created_at (sort) → both exist
because the query demanded them. Writing columns first / copying indexes later produces
mismatches (e.g. an index on a `created_at` column you forgot to add).
Store lat/lng on trips (exact pickup/dropoff for map/fare/receipt); geohash lives in the
Redis matching layer, not the trip record.

---

## 4. Failure handling
- **Driver goes offline / stops pinging** → their Redis location **TTL expires** → auto-removed
  from the available pool → no stale drivers offered to riders. (Short TTL = self-cleaning.)
- **Matching service crashes mid-trip** → trip state persists in the DB → recoverable
  (Redis holds only current *location*; trip *lifecycle* is durable).
- **Redis is critical infra** → run it clustered with replicas for HA.
- **Driver accepts then crashes before pickup** → timeout → reassign.

## 5. Observability
- **Monitor:** match latency (time to find a driver), location-ping ingestion lag,
  % requests with no driver found (supply gap), active-trip count, offer→accept rate.
- **Page on:** match latency spikes, no-driver-found rate grows, ingestion lag backs up
  (matching goes stale).

---

## Where YOUR attempt differed (fixes)
1. **Missed the location-ping firehose** — the ~250k/s of GPS updates is the DEFINING load,
   far bigger than ride requests. Surface it in requirements; it forces the Redis-GEO +
   Kafka design.
2. **Storage 10× slip** — 200,000 GB = 200 TB, not 20 TB. Smell test fired but the GB→TB
   conversion slipped.
3. **HLD — strong, yours.** Geohash, Kafka→Redis firehose, expanding radius, match flow —
   all driven. Add the geohash *explanation* (2D→1D prefix + neighbor cells) and untangle
   WebSocket (tracking) from ride request (API).
4. **LLD — the persistent gap.** Indexes not stated until forced, and not derived (indexed
   a `created_at` column that wasn't in the table). Derive index FROM the query, inline.

**One-line takeaway:** HLD is interview-ready and transfers to a brand-new concept
(geospatial). The single remaining gap is the query-first LLD reflex — derive the index
from the killer query, inline, table by table. That's a drilling fix, not a knowledge one.
