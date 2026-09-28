# Rate Limiter — Lua Scripts + Spring Boot Wiring

All algorithms run as ONE atomic Lua script in SHARED Redis (global limit across all
gateways). Called from the gateway via EVALSHA (script cached by hash).

Why Lua: Redis runs a Lua script atomically (uninterruptible), so two concurrent
requests from the same user can't both read the same count and both slip through.

---

## 1. Fixed Window Counter (simplest)

```lua
-- Fixed Window Counter (corrected)
-- KEYS[1] = "ratelimit:{user}:{window}"
-- ARGV[1] = limit
-- ARGV[2] = window_size_seconds
-- Returns: 1 = allowed, 0 = rejected

local count = tonumber(redis.call('GET', KEYS[1]) or '0')

if count >= tonumber(ARGV[1]) then
    return 0                                      -- limit hit -> reject
end

local new_count = redis.call('INCR', KEYS[1])     -- INCR returns the new value

if new_count == 1 then
    -- first request in this window -> set TTL ONCE
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
end

return 1
```

**How it works:** one counter per user per window (e.g. per minute). Increment each
request; reject if at the limit. EXPIRE makes the window's key vanish after it ends,
so the next window starts at 0.

**Flaw — boundary burst:** 100 requests at 12:00:59 + 100 at 12:01:00 = 200 in 2 sec
(different windows count separately). Simple but allows 2x bursts at boundaries.
Rarely the final answer.

---

## 2. Sliding Window Counter (practical default)

```lua
-- KEYS[1] = current window key
-- KEYS[2] = previous window key
-- ARGV[1] = limit
-- ARGV[2] = elapsed_fraction (0.0-1.0, how far into current window)
-- ARGV[3] = window_size_seconds (TTL)
-- Returns: 1 = allowed, 0 = rejected

local current  = tonumber(redis.call('GET', KEYS[1]) or '0')
local previous = tonumber(redis.call('GET', KEYS[2]) or '0')

local limit   = tonumber(ARGV[1])
local elapsed = tonumber(ARGV[2])

-- weight previous window by the portion still inside the rolling window
local estimate = current + previous * (1 - elapsed)

if estimate >= limit then
    return 0                                      -- over the rolling limit -> reject
end

redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[3]) * 2)   -- keep ~2 windows
return 1
```

**How it works:** two counters (current + previous window). Estimate the true rolling
count by weighting the previous window by (1 - elapsed) — how much of it still overlaps
the rolling 60s window. At 25% into the window, previous counts at 75%. Smooths out the
fixed-window boundary burst.

**elapsed** = (now % window_size) / window_size = fraction of current window already
passed (0.0-1.0). (1 - elapsed) = how much of the previous window still counts.

**Why EXPIRE * 2:** the current key must survive one extra window because next window it
becomes the "previous" that gets read, then it auto-expires.

**Trade-off:** slightly approximate (assumes even distribution in previous window), but
cheap (two counters) and no boundary bug. Good default for a STRICT limit.

---

## 3. Token Bucket (allows controlled bursts)

```lua
-- KEYS[1] = "ratelimit:{user}"  (a hash: tokens + last_refill)
-- ARGV[1] = capacity      (max tokens, e.g. 100)
-- ARGV[2] = refill_rate    (tokens per second, e.g. 10)
-- ARGV[3] = now            (current time, seconds)
-- ARGV[4] = ttl_seconds    (cleanup)
-- Returns: 1 = allowed, 0 = rejected

local capacity    = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now         = tonumber(ARGV[3])

local data        = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill')
local tokens      = tonumber(data[1])
local last_refill = tonumber(data[2])

if tokens == nil then
    tokens = capacity                             -- new bucket starts full
    last_refill = now
end

-- LAZY REFILL: add tokens accrued since last check, capped at capacity
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens < 1 then
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[4]))
    return 0                                       -- no token -> reject
end

tokens = tokens - 1                               -- consume one token
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[4]))
return 1
```

**How it works:** a bucket holds up to `capacity` tokens, refilling at `refill_rate`/sec.
Each request needs one token. **Lazy refill** is the trick — no background job adds
tokens; you compute `tokens + elapsed * refill_rate` at check time. A bucket unused for
5s at 10/sec "catches up" 50 tokens (capped) in one calculation.

**Why a hash (HMGET/HMSET):** two values per bucket — token count AND last-refill
timestamp — held under one key.

**Property:** token bucket ALLOWS bursts — a full bucket lets a user fire `capacity`
requests instantly, then throttles to the refill rate. Often desirable (real traffic is
bursty). Contrast: sliding window is a strict rolling limit, no bursts. Stripe/AWS use
token-bucket variants.

---

## Spring Boot wiring (identical for all three — just different KEYS/ARGV)

### 1. Load the script as a bean
```java
@Configuration
public class RateLimiterConfig {
    @Bean
    public DefaultRedisScript<Long> rateLimitScript() {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setLocation(new ClassPathResource("scripts/sliding_window.lua"));
        script.setResultType(Long.class);   // script returns 1 or 0
        return script;
    }
}
```
Put the .lua in src/main/resources/scripts/. DefaultRedisScript uses EVALSHA under the
hood (sends the hash, falls back to full text only on cache miss) — free script caching.

### 2. The service
```java
@Service
public class RateLimiterService {
    private final StringRedisTemplate redis;      // String template: values as
    private final DefaultRedisScript<Long> script; // readable strings for tonumber()

    private static final long WINDOW_SIZE = 60;
    private static final long LIMIT = 100;

    public RateLimiterService(StringRedisTemplate redis, DefaultRedisScript<Long> script) {
        this.redis = redis; this.script = script;
    }

    public boolean isAllowed(String userId) {
        try {
            long now = Instant.now().getEpochSecond();
            long currentWindow  = now / WINDOW_SIZE;
            long previousWindow = currentWindow - 1;
            double elapsed = (double) (now % WINDOW_SIZE) / WINDOW_SIZE;

            String curKey  = "ratelimit:" + userId + ":" + currentWindow;
            String prevKey = "ratelimit:" + userId + ":" + previousWindow;

            Long allowed = redis.execute(script,
                List.of(curKey, prevKey),                         // KEYS
                String.valueOf(LIMIT),                            // ARGV[1]
                String.valueOf(elapsed),                          // ARGV[2]
                String.valueOf(WINDOW_SIZE));                     // ARGV[3]

            return allowed != null && allowed == 1L;
        } catch (Exception e) {
            // FAIL OPEN: don't take down the API if the limiter's Redis is down
            log.warn("Rate limiter Redis unavailable, failing open", e);
            return true;
        }
    }
}
```

### 3. Apply as a filter (runs before controllers — cross-cutting concern)
```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    private final RateLimiterService rateLimiter;
    public RateLimitFilter(RateLimiterService r) { this.rateLimiter = r; }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String userId = extractUserId(req);   // from JWT / session
        if (!rateLimiter.isAllowed(userId)) {
            res.setStatus(429);
            res.getWriter().write("{\"error\":\"RATE_LIMITED\",\"message\":\"Too many requests\"}");
            return;                            // stop — don't hit the controller
        }
        chain.doFilter(req, res);              // under limit -> proceed
    }
}
```

---

## Interview talking points
- **Lua = atomicity** — read + compute + increment as one uninterruptible op; prevents the
  two-servers-both-allow race.
- **Shared Redis = global limit** across all gateways (not per-server).
- **DefaultRedisScript** does EVALSHA caching for you; use StringRedisTemplate so values
  are plain strings (for tonumber()).
- **Filter placement** — OncePerRequestFilter runs before controllers; rate limiting is a
  cross-cutting concern (Single Responsibility — not in the controller).
- **Fail open** on Redis down (usually) — a protective layer shouldn't take down the API.
- **Which algorithm:** fixed window (simple, boundary burst) / sliding window counter
  (strict, cheap, no bug — default) / token bucket (allows bursts — user-facing APIs).
