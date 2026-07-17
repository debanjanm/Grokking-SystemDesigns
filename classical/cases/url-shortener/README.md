# URL Shortener

Design a service like bit.ly that converts long URLs to short ones and redirects users.

---

## Requirements

### Functional Requirements
- Given a long URL, generate a unique short URL
- Redirect short URL → original URL
- Optional: custom aliases, link expiry, click analytics

### Non-Functional Requirements
- Latency: redirect < 10ms p99 (critical — on every click)
- Availability: 99.99% (redirect failures are user-visible)
- Consistency: eventual consistency acceptable — short URL resolves within seconds of creation
- Security & Privacy: prevent enumeration attacks; no user PII stored in URL records by default
- Scalability: 100M URLs created/day; 10:1 read/write ratio

### Out of Scope
- User accounts / auth
- Real-time analytics dashboard (async analytics pipeline)
- Link previews / safety scanning

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| Writes (URL creation) | 100M/day → ~1,200/sec |
| Reads (redirects) | 1B/day → ~12,000/sec |
| Read/write ratio | 10:1 |

### Storage Estimation
| Metric | Value |
|---|---|
| Avg record size | ~500 bytes |
| Records/day | 100M |
| Storage/day | ~50 GB |
| Storage (5yr) | ~90 TB |

### Memory Estimation
| Metric | Value |
|---|---|
| Hot URLs (20% drive 80% traffic) | ~20M records |
| Cache size (500B × 20M) | ~10 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Write throughput | ~600 KB/s |
| Read throughput | ~6 MB/s |

---

## High-Level Design

### Architecture Style
**Stateless Microservices** — read and write paths separated; redirect service scales independently from creation service. Read path is extremely hot; cache layer critical.

### Architecture Diagram
```
Client
  ↓
CDN (cache redirect responses)
  ↓
Load Balancer
  ├── Write Service (POST /shorten)
  │       ↓
  │   ID Generator (Snowflake)
  │       ↓
  │   DB Write (Cassandra) + Cache Warm
  │
  └── Redirect Service (GET /{code})
          ↓
      Cache (Redis) ──hit──→ 302 Redirect
          ↓ miss
      DB Read (Cassandra)
          ↓
      Cache Write + 302 Redirect
          ↓ async
      Analytics Event (Kafka)
```

### Core Components
| Component | Responsibility |
|---|---|
| Write Service | Validate URL, generate short code, persist, warm cache |
| Redirect Service | Resolve short code → long URL via cache → DB; issue redirect |
| ID Generator | Distributed unique ID generation (Snowflake) → Base62 encode |
| Cache (Redis) | Short code → long URL; hot path, sub-millisecond lookup |
| DB (Cassandra) | Persistent store; key-value access pattern; high read throughput |
| CDN | Cache redirect responses for popular short URLs at edge |
| Analytics Pipeline | Async click event processing (Kafka → consumer → data warehouse) |

### Data Flow

**URL Creation:**
1. Client POST `/shorten` with long URL
2. Write service validates URL (format, blocklist check)
3. ID generator produces unique integer → Base62 encode → 7-char short code
4. Write to Cassandra (short_code → record)
5. Warm Redis cache
6. Return short URL to client

**Redirect:**
1. Client GET `/{short_code}` → hits CDN first
2. CDN miss → Redirect service → Redis lookup
3. Redis hit → 302 response immediately
4. Redis miss → Cassandra read → populate Redis → 302 response
5. Async: publish click event to Kafka

### Key Design Decisions
- **302 over 301 redirect:** 301 is cached by browser permanently — no analytics, no way to update. 302 always hits server — enables click tracking, expiry enforcement, URL updates.
- **Base62 encoding of Snowflake ID:** Avoids collision risk of MD5/hash. Snowflake gives globally unique monotonic ID; Base62 gives short, URL-safe string.
- **Cassandra over SQL:** Access pattern is pure key-value (short_code lookup). Cassandra excels at high read throughput, horizontal scaling, no joins needed.
- **CDN for top URLs:** Popular links (viral content) cause thundering herd on redirect service. CDN caches 302 responses at edge — eliminates server load for hot URLs.

---

## Low-Level Design

### Data Models
```
urls (Cassandra):
  short_code    VARCHAR   PK
  long_url      TEXT
  user_id       VARCHAR   (nullable)
  created_at    TIMESTAMP
  expires_at    TIMESTAMP (nullable)
  custom_alias  BOOLEAN

analytics (Cassandra — separate table):
  short_code    VARCHAR   partition key
  clicked_at    TIMESTAMP clustering key
  user_agent    TEXT
  country       TEXT
  referrer      TEXT
```

### API Design
```
POST /shorten
Request:  { "long_url": "https://...", "custom_alias": "my-link", "expires_in_days": 30 }
Response: { "short_url": "https://sho.rt/aB3xYz7", "short_code": "aB3xYz7" }

GET /{short_code}
Response: HTTP 302 Location: <long_url>
          HTTP 404 if not found or expired

GET /analytics/{short_code}
Response: { "total_clicks": 12345, "clicks_by_country": {...}, "clicks_by_day": {...} }

DELETE /{short_code}
Response: HTTP 204
```

### Component Deep-Dives

#### ID Generator (Snowflake-based)
```
64-bit Snowflake ID:
  [41 bits timestamp][10 bits machine_id][12 bits sequence]

→ Base62 encode (a-z, A-Z, 0-9)
→ 7-character short code (62^7 = 3.5 trillion unique codes)
```

Why not hash (MD5/SHA): collision risk at scale; requires collision detection retry loop. Snowflake is collision-free by construction.

#### Cache Strategy (Redis)
```
Key:   short_code
Value: { long_url, expires_at }
TTL:   24 hours (refresh on access for active links)
```

On redirect:
1. GET short_code from Redis
2. Check expires_at in value (not just Redis TTL) — allows accurate expiry without relying on TTL precision
3. Cache miss → Cassandra → SET in Redis with 24hr TTL

#### Custom Alias
- User provides alias → check Redis + Cassandra for collision → reserve if free
- Rate limit: max 10 custom aliases per user per day (prevent squatting)

#### Guardrails / Safety Layer
- URL validation: must parse as valid HTTP/HTTPS URL
- Blocklist check: scan long URL against known phishing/malware domains (Google Safe Browsing API)
- Rate limiting: max 100 URL creations per IP per hour
- Short code enumeration prevention: 7-char Base62 space is 3.5T — brute force infeasible; no sequential exposure (Snowflake IDs are not exposed directly)

### Algorithms & Strategies
- **ID generation:** Snowflake (distributed, monotonic, no coordination needed). Each write service node has unique `machine_id` (assigned at startup from ZooKeeper or env config).
- **Caching strategy:** Cache-aside pattern. Redis TTL: 24hr with sliding expiry on access. Eviction: LRU (hot URLs stay, cold URLs evicted).
- **Expiry enforcement:** expires_at stored in Cassandra and Redis value. Redirect service checks at read time — expired link returns 410 Gone. Background job cleans up expired rows weekly.
- **Analytics:** Fire-and-forget Kafka publish on each redirect. Consumer aggregates into ClickHouse (OLAP). No sync dependency on analytics for redirect latency.

### Security Design
- AuthN/AuthZ: API key for URL creation; public for redirects (no auth needed)
- Short code space: 62^7 = 3.5T codes — enumeration attack impractical
- Blocklist: Google Safe Browsing API integration on creation; periodic re-scan of existing URLs
- Encryption at rest: Cassandra encrypted; Redis encrypted
- Encryption in transit: TLS 1.3 on all endpoints
- PII: user_id stored only if authenticated; click analytics anonymized (IP not stored, country derived then discarded)
- Rate limiting: per-IP on creation endpoint (prevent abuse/spam)

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Redirect latency (p99) | Histogram | > 10ms |
| Cache hit rate | Gauge | < 90% |
| URL creation rate | Counter | — |
| 404 rate | Counter | > 1% (possible enumeration attack) |
| DB read latency (p99) | Histogram | > 50ms |

### Logging
- Per redirect: short_code, cache_hit, latency, status_code, country (derived)
- Per creation: short_code, url_hash (not raw URL), user_id_hash, is_custom_alias
- Do NOT log: raw long_url (may contain auth tokens), full user IPs

### Tracing
- Trace spans: CDN → redirect_service → redis_lookup → cassandra_read
- Sample: 1% normal (extremely high volume); 100% on 404s and errors

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Redirect latency spike | p99 > 10ms | Check Redis hit rate, scale redirect service |
| Cache hit rate drop | < 85% | Check Redis memory, eviction rate |
| 404 rate spike | > 2% | Possible enumeration attack — rate limit by IP |
| DB latency spike | p99 > 100ms | Check Cassandra node health |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| 302 redirect | Analytics accuracy, expiry control | Browser-side caching (every click hits server) |
| Base62 + Snowflake ID | Zero collision, no retry needed | Need distributed ID generator infrastructure |
| Cassandra over MySQL | Horizontal scale, high read throughput | No ACID transactions (not needed for this use case) |
| CDN for hot URLs | Edge-cached redirects (0ms server load) | 302 cached at CDN — analytics undercounted for viral links |
| Async analytics (Kafka) | Zero impact on redirect latency | Analytics lag (minutes), not real-time |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Redis down | OOM or node failure | Fail open to Cassandra; pre-warm Redis on recovery |
| Cassandra node down | Hardware failure | Cassandra replication (RF=3); redirect degrades gracefully with remaining nodes |
| ID generator down | Snowflake node crash | Multiple ID generator instances; fallback to secondary |
| Hot URL cache miss storm | Viral link, cache cold | CDN absorbs; probabilistic early expiry (cache refresh before TTL) |
| Expired link redirect | expires_at not checked | Check expires_at in value, not just Redis TTL; return 410 Gone |

---

## Interview Tips

### What Interviewers Expect
- ID generation strategy — must explain Snowflake vs hash and why Snowflake wins
- 301 vs 302 tradeoff — always ask this; 302 is correct answer for analytics
- Cache hierarchy: CDN → Redis → Cassandra (three layers)
- Cassandra choice: justify key-value access pattern

### Common Follow-ups
- How to handle expiry at scale? TTL in Redis + check expires_at at read time + async cleanup job
- Hot URL problem (thundering herd)? CDN layer + probabilistic early cache refresh
- Custom aliases at scale? Separate lookup table; rate-limit to prevent squatting
- Analytics at scale? Kafka → ClickHouse OLAP pipeline; HyperLogLog for unique visitor counts
