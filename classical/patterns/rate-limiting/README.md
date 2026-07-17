# Pattern: Rate Limiting

Control the rate of requests a client can make to a service — preventing abuse, ensuring fairness, and protecting downstream systems.

---

## What It Is

Rate limiting enforces an upper bound on how many requests a client (user, IP, API key, tenant) can make within a time window. It sits at the API gateway or service layer and rejects excess requests with HTTP 429 (Too Many Requests).

---

## When to Use

| Use | Avoid |
|---|---|
| Public API (prevent abuse, DDoS mitigation) | Internal trusted services with known load |
| Multi-tenant SaaS (enforce plan limits) | Batch processing systems (throughput ≠ abuse) |
| Expensive downstream resources (LLM, DB) | Systems where dropping requests is worse than overload |
| Third-party API calls (stay within vendor limits) | |
| Protect from runaway clients (bugs, loops) | |

---

## Core Concepts

### Limiting Dimensions
Choose what to limit on:
- **Per IP** — anonymous abuse prevention
- **Per API key / user** — authenticated usage enforcement
- **Per tenant** — multi-tenant fairness
- **Per endpoint** — protect expensive routes specifically
- **Global** — system-wide throughput cap

Often combine: per-user AND per-endpoint (e.g., 100 req/min global + 10 req/min for `/expensive`).

### Response on Limit
```
HTTP 429 Too Many Requests
Headers:
  Retry-After: 30              ← seconds until window resets
  X-RateLimit-Limit: 100       ← total allowed
  X-RateLimit-Remaining: 0     ← requests left this window
  X-RateLimit-Reset: 1721214000 ← UTC epoch when window resets
```

---

## Algorithms

### 1. Fixed Window Counter
```
Window: [0s–60s], [60s–120s], ...
Counter: increment per request in current window
Reject:  if counter > limit

Problems:
  - Boundary burst: 100 req at 0:59 + 100 req at 1:01 = 200 req in 2 seconds
  - Bursty by design
```

### 2. Sliding Window Log
```
Store: sorted set of request timestamps per client
On request:
  1. Remove timestamps older than window (ZREMRANGEBYSCORE)
  2. Count remaining (ZCARD)
  3. If count >= limit → reject
  4. Else add current timestamp (ZADD)

Accurate: no boundary burst
Cost: O(n) storage per client per window
```

### 3. Sliding Window Counter (approximate)
```
Hybrid: two fixed windows + weighted interpolation
  current_window_count = fixed_window[now]
  prev_window_count    = fixed_window[prev]
  overlap_fraction     = (window_size - elapsed_in_current) / window_size

  rate = prev_window_count × overlap_fraction + current_window_count

Accurate to ~0.1%; O(1) storage; best practical choice
```

### 4. Token Bucket ← recommended for most cases
```
Bucket holds up to CAPACITY tokens
Tokens added at REFILL_RATE per second
Request consumes 1 token (or N tokens for weighted requests)
No tokens → reject

Properties:
  - Allows bursts up to CAPACITY
  - Sustains average rate of REFILL_RATE
  - Most natural model for "allow some burst, enforce average"

Example: capacity=20, refill=10/sec
  → client can burst 20 requests instantly, then sustains 10/sec
```

### 5. Leaky Bucket
```
Requests enter bucket (queue)
Bucket drains at fixed rate (regardless of input rate)
Bucket full → drop request

Properties:
  - Smooths bursty traffic into steady stream
  - No bursts allowed (unlike token bucket)
  - Good for: protecting downstream that can't handle bursts
```

### Algorithm Comparison
| Algorithm | Burst | Accuracy | Storage | Use |
|---|---|---|---|---|
| Fixed Window | Yes (boundary) | Low | O(1) | Simple, not recommended |
| Sliding Log | No | Exact | O(n) | Low-volume, exact needed |
| Sliding Counter | Minimal | ~99.9% | O(1) | General purpose |
| Token Bucket | Yes (controlled) | Exact | O(1) | API rate limiting |
| Leaky Bucket | No | Exact | O(n queue) | Smoothing traffic |

---

## Architecture

### Centralized (Redis-based)
```
Client → API Gateway
              ↓
         Rate Limiter (checks Redis)
              ├── ALLOW → forward to service
              └── REJECT → 429

Redis stores counters:
  key: "rl:{user_id}:{endpoint}:{window}"
  value: integer counter or token bucket state
  TTL: window size
```

**Pros:** exact counting across all gateway instances  
**Cons:** Redis round-trip adds ~1ms; single point of failure

### Distributed (local + sync)
```
Each API gateway node:
  Local counter (in-process, ~µs lookup)
        ↓ periodically sync
  Shared Redis counter (every 100ms or N requests)

Trade-off: may allow small overage (local count not yet synced)
Good for: very high QPS where Redis round-trip is unacceptable
```

### Rate Limiting in the Stack
```
L1: CDN (Cloudflare) → block obvious DDoS, IP-level limits
L2: API Gateway (Kong / Nginx) → per API key, per route
L3: Service-level → per user per expensive operation
L4: DB connection pool → implicit rate limiting via queue depth
```

---

## Implementation Details

### Token Bucket in Redis (Lua script — atomic)
```lua
-- Atomic token bucket check-and-update
local key      = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill   = tonumber(ARGV[2])   -- tokens per second
local now      = tonumber(ARGV[3])   -- current timestamp (ms)
local cost     = tonumber(ARGV[4])   -- tokens this request costs

local bucket = redis.call("HMGET", key, "tokens", "last_refill")
local tokens     = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens based on elapsed time
local elapsed = (now - last_refill) / 1000.0   -- seconds
tokens = math.min(capacity, tokens + elapsed * refill)

if tokens >= cost then
    tokens = tokens - cost
    redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
    redis.call("EXPIRE", key, 3600)
    return 1   -- ALLOW
else
    redis.call("HMSET", key, "tokens", tokens, "last_refill", now)
    return 0   -- REJECT
end
```

Lua script is atomic in Redis — no race condition between check and update.

### Sliding Window Counter in Redis
```python
def is_allowed(user_id: str, limit: int, window_sec: int) -> bool:
    now = time.time()
    window_start = now - window_sec

    pipe = redis.pipeline()
    key = f"rl:{user_id}"

    # Remove old entries outside window
    pipe.zremrangebyscore(key, 0, window_start)
    # Count entries in window
    pipe.zcard(key)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Set TTL
    pipe.expire(key, window_sec)

    _, count, _, _ = pipe.execute()
    return count < limit
```

### Rate Limit Headers
```python
# Always return headers even on success — lets clients self-throttle
response.headers["X-RateLimit-Limit"]     = str(limit)
response.headers["X-RateLimit-Remaining"] = str(max(0, limit - count))
response.headers["X-RateLimit-Reset"]     = str(int(window_reset_epoch))

if count >= limit:
    response.headers["Retry-After"] = str(window_sec)
    return 429
```

### Tiered Limits (multi-tenant SaaS)
```python
PLANS = {
    "free":       { "rpm": 60,   "tpm": 10_000  },
    "pro":        { "rpm": 1000, "tpm": 100_000 },
    "enterprise": { "rpm": 10000, "tpm": 1_000_000 },
}

def get_limits(tenant_id):
    plan = db.get_plan(tenant_id)  # cached in Redis
    return PLANS[plan]
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Token bucket over fixed window | Controlled burst; no boundary exploit | Slightly more complex implementation |
| Centralized Redis | Exact counting across nodes | Redis round-trip latency (~1ms); Redis as dependency |
| Distributed local+sync | Lower latency | Small overage allowed during sync interval |
| Lua script (atomic) | No race condition | Redis CPU for script execution |
| Sliding window counter over log | O(1) storage; accurate | ~0.1% approximation error |
| Per-user + per-endpoint limits | Fine-grained protection | More Redis keys; higher memory usage |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Fixed window boundary burst | Use token bucket or sliding window |
| Race condition on check-then-set | Atomic Lua script or Redis INCR |
| No Retry-After header | Client can't back off intelligently |
| Limiting on mutable IP (proxies, NAT) | Also limit on API key / user_id |
| No bypass for internal services | Allowlist internal IPs / service accounts |
| Single Redis node (SPOF) | Redis Cluster or read replica with degraded fallback |
| Not alerting on high rejection rate | Rejection spike = possible attack or client bug |

---

## Used In
- [URL Shortener](../../cases/url-shortener/) — 100 URL creations/hr per IP; anti-spam
- [LLM Gateway](../../../genai-systems/cases/llm-gateway/) — token bucket per tenant (request + token level)
- [RAG Pipeline](../../../genai-systems/cases/rag-pipeline/) — per-tenant query rate limit
- [Twitter Feed](../../cases/twitter-feed/) — tweet posting limit; API rate limit for 3rd party clients
- [Uber](../../cases/uber/) — surge pricing request limiting; driver location update throttle

---

## Tools
| Tool | Notes |
|---|---|
| Redis | Token bucket / sliding window via Lua; industry standard |
| Kong | API gateway with rate limiting plugin |
| Nginx (limit_req) | Leaky bucket at nginx layer; simple config |
| Cloudflare | CDN-level rate limiting; blocks before origin |
| AWS API Gateway | Built-in throttling per stage/method |
| Envoy / Istio | Service mesh rate limiting (sidecar) |
| Custom middleware | Full control; implement token bucket in app layer |
