# Rate Limiter — Reference (HLD Mock)

Study reference. You drove the ALGORITHM half well (token bucket, burst problem,
sliding window). The gap: the DISTRIBUTED half (global counter in shared Redis +
atomic op) is the crux and should be LED with, not arrived at.

---

## 1. Requirements

**Functional**
- Limit requests per client per window (e.g. 100/min per user); configurable
- Exceed → reject with **429 Too Many Requests**
- Sits in front of services; every request passes through

**Non-functional**
- **THE key number is THROUGHPUT, not storage:** every request checks the limiter →
  the limiter handles your FULL request volume. 100M/day ≈ ~1,200/s avg, ~12,000/s peak.
  That's 12K counter-checks/sec against the shared store — the number that matters.
- Latency: the check adds to EVERY API call → must be sub-ms (why Redis)
- High availability (it's in every request path)

---

## 2. Algorithms (know 2-3 + tradeoffs)

| Algorithm | How | Note |
|-----------|-----|------|
| **Fixed window** | count per calendar minute, reset each window | **Boundary bug:** 10K in last sec of min 1 + 10K in first sec of min 2 = 20K in 2 sec |
| **Sliding window log** | store a timestamp per request, count last 60s | precise, but memory-heavy (stores every timestamp) |
| **Sliding window counter** | weight previous window by overlap with the rolling window | practical approximation, fixes the boundary bug cheaply — usual pick for a strict limit |
| **Token bucket** | tokens refill at a steady rate; request needs a token; bucket has max capacity | **allows controlled bursts by design** (a feature, not a bug); smooth rate + burst tolerance |
| **Leaky bucket** | requests queue, processed at a fixed rate | smooths output, adds queuing |

**The burst / boundary problem** is the thing to explain: fixed window allows a 2× burst
across the window boundary. Sliding window (rolling 60s) fixes it. Token bucket *permits*
bounded bursts on purpose.

**Refill:** prefer **lazy refill** (compute tokens on read:
`tokens = min(capacity, last + elapsed × rate)`) over a background job filling millions of
buckets — the job approach is wasteful.

**Pick:** sliding-window-counter for a strict rolling limit; token bucket if you want to
allow controlled bursts. Name it as a tradeoff.

---

## 3. HLD — the DISTRIBUTED design (the crux — LEAD with this)

**The core problem:** the limiter runs on MANY gateway servers, but the limit is GLOBAL
("100/min per user" across ALL servers, not per-server). If each server counts locally, a
user hitting 5 servers gets 5× their limit.

**The answer:** the counter/bucket lives in a **shared Redis**, keyed per user
(`ratelimit:{user_id}`). Every gateway, on every request, does an **atomic check-and-
decrement** against the SAME Redis → the limit is global.

```
Request → LB → Gateway server (any of N)
  → atomic check-and-decrement in Redis  ratelimit:{user_id}
        token available? → decrement, forward request
        none?            → reject 429
  → shared Redis = GLOBAL limit across all gateways
```

**The atomic race (must name):** two requests from one user hit two gateways at once, both
read "1 token left," both allow → over limit. Fix: the check-and-decrement is ONE atomic
Redis operation — a **Lua script** (Redis runs Lua atomically) or atomic `INCR` + TTL.
No interleaving → no double-allow.

**The performance concern (must raise):** every request makes a network hop to Redis before
being served. Redis handles 12K/s easily (~100K ops/s ceiling), and it's sub-ms, so usually
fine. If not: each gateway holds a small local allowance synced periodically with Redis
(trades accuracy for fewer calls).

---

## 4. Failure — fail open vs fail closed (name it as a TRADEOFF)
- **Fail OPEN** (Redis down → allow all): the product stays up; you lose rate-limiting
  temporarily. **Usual industry default** — a protective layer shouldn't take down the whole
  API when its dependency fails.
- **Fail CLOSED** (Redis down → reject all): protects a fragile/expensive downstream that
  would collapse under unlimited load — but turns "limiter down" into "whole API down."
- **The answer is conditional:** robust backend that can absorb a burst → fail open. Fragile
  downstream that would collapse → fail closed. Depends on what you're protecting.
- Redis clustered with replicas for HA (reduces how often this even comes up).

## 5. Observability
- **Monitor:** 429 rate (too high = limits too tight or an attack), **Redis latency**
  (it's in EVERY request path — degradation slows all API calls), per-user request rate
  (spot abusers), Redis availability.
- **Page on:** 429 rate spikes, Redis latency breaches, Redis unavailable.

---

## Where YOUR attempt differed (fixes)
1. **Lead with the distributed counter.** You drove the algorithm well but reached the
   shared-Redis-global-counter only after a push. For a rate limiter, the distributed
   counter IS the interesting part — the algorithm is table stakes. Open with
   "counter in shared Redis so the limit is global, atomic check-and-decrement."
2. **Throughput, not storage, is the key number.** You computed storage (~1TB) but the
   metric that matters is 12K checks/sec against the shared store.
3. **Token bucket ALLOWS bursts** (a feature) — you called it a burst problem. Restate.
4. **Fail open vs closed is a TRADEOFF** — you picked fail-closed with one reason; name both
   and the condition (fail open usually; fail closed only for fragile downstreams).
5. **Lazy refill** over a fill-job for millions of buckets.

**One-line takeaway:** algorithm half is solid and yours; the distributed half (global Redis
counter + atomic op + fail-open/closed tradeoff) is the crux to lead with. Same pattern as
always — name the fork and the tradeoff, don't just pick.
