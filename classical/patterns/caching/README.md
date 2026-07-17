# Pattern: Caching

Store frequently accessed data in a fast, temporary layer to reduce latency and backend load.

---

## What It Is

Caching intercepts repeated reads and serves them from a fast store (memory) instead of recomputing or re-fetching from the origin (DB, API, ML model). Effective because most systems exhibit **temporal locality** — recently accessed data is likely to be accessed again soon.

---

## When to Use

| Use | Avoid |
|---|---|
| Read-heavy workload (read:write > 5:1) | Write-heavy (cache constantly invalidated) |
| Expensive computation (DB join, ML inference) | Data must always be fresh (financial balances) |
| Repeated identical queries | Unique queries every time (tail traffic) |
| Latency SLA tight (< 20ms) | Cache poisoning risk outweighs benefit |
| Static or slow-changing data | Dataset too large to fit in memory |

---

## Core Concepts

### Cache Hit vs Miss
```
Request → Cache lookup
  ├── HIT:  return cached value (fast, ~1ms)
  └── MISS: fetch from origin → store in cache → return value (slow, ~50-200ms)

Hit rate = hits / (hits + misses)
Target: > 80% for meaningful benefit
```

### Eviction Policies
| Policy | Logic | Best for |
|---|---|---|
| **LRU** (Least Recently Used) | Evict item not accessed longest | General purpose — assumes recency = relevance |
| **LFU** (Least Frequently Used) | Evict item accessed fewest times | Hot item preservation; skewed access patterns |
| **TTL** (Time-to-Live) | Evict after fixed time regardless of access | Time-sensitive data (sessions, rate limit windows) |
| **FIFO** | Evict oldest inserted | Simple; rarely optimal |
| **Random** | Evict random item | Approximates LRU with less overhead |

### Invalidation Strategies
| Strategy | When | Tradeoff |
|---|---|---|
| **TTL expiry** | Auto-expire after N seconds | Simple; may serve stale data up to TTL |
| **Write-through** | Update cache on every write | Always fresh; write latency doubles |
| **Write-behind (lazy)** | Write to cache; async write to DB | Fast writes; risk of data loss on crash |
| **Cache-aside** | App manages: miss → fetch → store | Flexible; app controls logic |
| **Event-driven invalidation** | Publish event on write → cache listener invalidates | Fresh; complex; good for shared caches |

---

## Architecture

### Cache-Aside (most common)
```
Client
  ↓
Application
  ├── READ:
  │    1. Check cache (Redis GET key)
  │    2. HIT → return
  │    3. MISS → query DB → SET cache → return
  │
  └── WRITE:
       1. Write to DB
       2. DELETE cache key (or let TTL expire)
```

### Write-Through
```
Client → Application → Cache (write) → DB (write, sync)
                            ↓
               Read: always hits cache (fresh)
```
Good for: read-heavy, strong consistency needed.

### Cache Hierarchy (multi-layer)
```
Client → L1: Browser/CDN cache (edge, ms)
              ↓ miss
         L2: Application cache (in-process, µs)
              ↓ miss
         L3: Distributed cache (Redis, ~1ms)
              ↓ miss
         L4: Database (50-200ms)
```

Each layer has higher latency but larger capacity.

---

## Implementation Details

### Redis Key Design
```
Patterns:
  Simple:      "user:{user_id}"           → serialized user object
  Namespaced:  "session:{session_id}"     → session data
  Composite:   "feed:{user_id}:{page}"    → paginated feed
  Counters:    "rate:{user_id}:{window}"  → integer counter

Avoid:
  - Very long keys (memory overhead)
  - Unpredictable keys (random UUIDs as values → can't invalidate by pattern)
  - JSON in keys (use structured prefix:id format)
```

### TTL Strategy
```
Static data (config, lookup tables):    TTL = 1 hour – 24 hours
User profile / preferences:            TTL = 15 min – 1 hour
Feed / recommendation:                 TTL = 1 – 5 min
Session data:                          TTL = sliding (reset on access), 30 min idle
Rate limit counters:                   TTL = window size (60s, 3600s)
Search results:                        TTL = 5 min (with targeted invalidation on catalog update)
```

### Cache Stampede Prevention
Problem: popular cache entry expires → thousands of requests hit DB simultaneously.

```
Solutions:

1. Mutex lock (Redis SET NX):
   if cache miss:
     lock = SETNX "lock:{key}" 1 EX 5s
     if lock acquired:
       value = fetch_from_db()
       SET key value EX ttl
       DEL lock
     else:
       wait 50ms → retry cache lookup

2. Probabilistic early refresh:
   remaining_ttl = TTL(key)
   if random() < exp(-beta × remaining_ttl):
     refresh cache now (before expiry)
   # Starts refreshing slightly before TTL — avoids hard expiry storm

3. Background refresh:
   Async job refreshes cache before TTL expires
   Serves stale value during refresh (stale-while-revalidate)
```

### Cache Warming
```
On cold start or after flush:
  - Pre-populate cache with top-N hot keys from DB
  - Identify hot keys: query logs, analytics
  - Warm before serving traffic (deploy → warm → route traffic)
  - Or: gradual traffic ramp (small % → cache warms naturally)
```

### Distributed Cache — Sharding
```
Multiple Redis nodes:
  hash_slot = CRC16(key) % 16384   ← Redis Cluster
  node = slot_to_node[hash_slot]

Consistent hashing (see consistent-hashing pattern):
  Adding/removing nodes → only ~1/N keys remapped
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Cache-aside over write-through | App controls invalidation; simpler infra | Risk of stale reads between write and TTL expiry |
| TTL-based invalidation | Simple, no event bus needed | Data stale up to TTL duration |
| LRU eviction | Hot data stays; simple | LFU better for extremely skewed access patterns |
| Distributed cache (Redis) | Shared across app instances | Network hop (~1ms); single point of failure if not clustered |
| High TTL | Better hit rate | Stale data longer |
| Low TTL | Fresh data | More DB load; lower hit rate |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Cache thundering herd | Mutex lock or probabilistic early refresh |
| Caching mutable data without invalidation | TTL or event-driven invalidation on write |
| Storing sensitive data (PII, passwords) in cache | Encrypt at rest; don't cache secrets |
| Cache key collision | Use namespaced, typed key schema |
| No cache hit rate monitoring | Alert on hit rate < 80%; investigate |
| Caching errors / null values | Don't cache exceptions; short TTL for "not found" (negative caching) |
| Over-caching (caching everything) | Only cache expensive or repeated reads |

### Negative Caching
```
# Cache "not found" to prevent repeated DB lookups for non-existent keys
if db_result is None:
    SET "user:{id}" "__NOT_FOUND__" EX 60   ← short TTL
    return 404

if cache_value == "__NOT_FOUND__":
    return 404
```

---

## Used In
- [URL Shortener](../../cases/url-shortener/) — Redis cache for short_code → long_url; 10:1 read:write ratio
- [Twitter Feed](../../cases/twitter-feed/) — timeline cache, tweet cache, user profile cache
- [Uber](../../cases/uber/) — driver location cache, surge pricing cache, ETA cache
- [ML Model Serving](../../../ml-systems/patterns/model-serving/) — prediction cache for repeated inputs
- [LLM Gateway](../../../genai-systems/cases/llm-gateway/) — semantic response cache (vector similarity)
- [Search Ranking](../../../ml-systems/cases/search-ranking/) — popular query result cache (Redis, TTL 5min)

---

## Tools
| Tool | Notes |
|---|---|
| Redis | Industry standard; rich data structures; Cluster for distribution |
| Memcached | Simpler; pure key-value; slightly faster for simple get/set |
| CDN (Cloudflare, CloudFront) | HTTP-level caching; static assets + API responses |
| Varnish | HTTP reverse proxy cache; complex invalidation rules |
| Caffeine (Java) | In-process L1 cache; no network hop |
| In-process dict/LRU | Simplest; per-instance; no sharing across replicas |
