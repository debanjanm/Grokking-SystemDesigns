# How to Solve Classical System Design Problems

Distributed systems, databases, caching, messaging, consistency. See [universal guide](../HOW_TO_SOLVE.md) first.

---

## What "Classical" Means Here

Systems where the core challenge is **scale, availability, and consistency** — not ML or AI. The building blocks are: services, databases, caches, queues, load balancers, CDNs.

Examples: URL shortener, social feed, ride-sharing, payment system, messaging app, search engine, file storage.

---

## Step 1 — Nail the Problem Type First

Before anything else, classify the system:

```
What is the dominant operation?

Read-heavy (feed, search, profile)?
  → Cache-first design. Read replicas. CDN.

Write-heavy (logging, analytics, IoT telemetry)?
  → Async ingestion. LSM-tree DB (Cassandra, RocksDB). Batch processing.

Mixed with strong consistency (payments, inventory, bookings)?
  → ACID DB (PostgreSQL). Transactions. Idempotency keys.

Real-time (chat, live tracking, gaming)?
  → WebSocket / SSE. In-memory state. Pub/sub.

Graph-heavy (social network, recommendations, fraud rings)?
  → Graph DB (Neo4j) or adjacency list in Cassandra.

Geo-spatial (maps, ride-sharing, delivery)?
  → Geohash or quad-tree. In-memory spatial index.
```

Knowing the type unlocks the right component vocabulary immediately.

---

## Step 2 — Clarifying Questions (Classical-Specific)

Beyond universal questions, ask these:

**Scale:**
- How many users? DAU vs MAU?
- Read:write ratio?
- Peak QPS vs average?
- Data retention period?

**Consistency:**
- Is stale data ever acceptable? For how long?
- Are there financial transactions? (strong consistency required)
- Is this multi-region? (distributed consistency problem)

**Availability:**
- What is the cost of 1 minute of downtime?
- Is this customer-facing or internal?

**Data:**
- Structured (SQL) or unstructured (documents, blobs)?
- Any media (images, video)? (changes storage estimates 10–100×)
- Hot data vs cold data split?

---

## Step 3 — Database Selection Decision Tree

```
Need ACID transactions (payments, bookings, inventory)?
  └── PostgreSQL / MySQL

Need massive write throughput + horizontal scale (logs, events, time-series)?
  └── Cassandra / DynamoDB / ScyllaDB

Need flexible schema + document model?
  └── MongoDB

Need global distribution + multi-region writes?
  └── DynamoDB (global tables) / CockroachDB / Spanner

Need full-text search?
  └── Elasticsearch (on top of primary DB)

Need graph traversal (social graph, fraud rings)?
  └── Neo4j / Amazon Neptune

Need key-value at < 1ms latency?
  └── Redis (cache) / DynamoDB (persistent)

Need blob storage (images, video, files)?
  └── S3 / GCS (always — never store blobs in DB)
```

> Never pick a DB first. Pick the access pattern, then pick the DB that fits it best.

---

## Step 4 — Component Vocabulary

Know these cold. Interviewers expect you to use correct names.

### Compute
| Component | Use |
|---|---|
| Load Balancer | Distribute traffic; health checks; SSL termination |
| API Gateway | Auth, rate limit, routing, request logging |
| Service Mesh | Service-to-service auth, observability (Istio, Envoy) |
| CDN | Edge-cache static + dynamic content; geo-distribute reads |

### Storage
| Component | Use |
|---|---|
| Primary DB | Source of truth; ACID or eventual |
| Read Replica | Scale reads; near-zero overhead for read-heavy |
| Cache (Redis) | Hot data; session; rate limit counters |
| Object Store (S3) | Blobs, backups, static assets, data lake |
| Message Queue (Kafka) | Async decoupling; event streaming; fan-out |
| Search Index (Elasticsearch) | Full-text search on top of primary DB |

### Patterns
| Pattern | Use |
|---|---|
| Sharding | Horizontal DB split by partition key (user_id, region) |
| Consistent Hashing | Shard without full remapping on node change |
| CQRS | Separate read model from write model |
| Saga | Distributed transaction across services (no 2PC) |
| Outbox Pattern | Reliable event publish without 2PC (write to DB + outbox table atomically) |
| Circuit Breaker | Stop calling failing downstream service; fail fast |
| Idempotency Key | Safe retries on write operations |

---

## Step 5 — HLD Skeleton for Classical Systems

```
[Client]
    ↓
[CDN] (static assets, cacheable API responses)
    ↓
[Load Balancer]
    ↓
[API Gateway] (auth, rate limit, routing)
    ↓
[Service Layer] (stateless, horizontally scalable)
    ├── [Cache - Redis] (hot reads, session, counters)
    │         ↓ miss
    ├── [Primary DB] (source of truth)
    │         ↓ async events
    └── [Message Queue - Kafka]
              ↓
         [Workers / Consumers]
              ↓
         [Object Store - S3] (media, backups)
```

This is the **default skeleton**. Deviate deliberately and explain why.

---

## Step 6 — Scale Each Layer

Work layer by layer. For each: "what breaks here first?"

**API layer:**
- Stateless → scale horizontally
- Session state → move to Redis (not in-process)
- Rate limiting → Redis token bucket at gateway

**Cache layer:**
- Hit rate < 80% → rethink what you're caching
- Cache stampede → mutex lock or probabilistic early refresh
- Memory full → LRU eviction; shard cache cluster

**DB layer:**
- Read-saturated → add read replicas
- Write-saturated → shard by partition key
- Joins across shards → denormalize or move to NoSQL
- Index bloat → analyze query patterns; partial indexes

**Queue layer:**
- Consumer lag → add consumer instances (Kafka partitions = max parallelism)
- Message ordering → single partition per entity_id
- At-least-once delivery → idempotent consumers

---

## Step 7 — Consistency Checklist

Ask yourself for every write operation:

```
1. Is this write idempotent?
   (safe to retry without duplicate effect?)

2. What happens if it succeeds but the response is lost?
   (client retries → duplicate? use idempotency key)

3. What if two services need to update atomically?
   (distributed transaction → use Saga + compensating transactions)

4. How stale can a read be?
   (financial data → 0s; feed → 5s fine; analytics → minutes fine)

5. Is there a race condition?
   (two users booking last ticket → optimistic locking or DB constraints)
```

---

## Step 8 — Common Sub-Problems and Solutions

### ID Generation
```
Need: globally unique, time-ordered, no coordination

Options:
  UUID v4          → random, no order, collision-safe, 128-bit
  Snowflake        → 64-bit, time-ordered, datacenter+sequence, no DB needed ← best
  DB auto-increment → simple, sequential, single-node bottleneck
  ULID             → URL-safe Snowflake alternative
```

### Pagination
```
Offset (LIMIT/OFFSET):
  Simple; breaks on inserts (items shift); slow on large offsets
  Use for: admin tools, small datasets

Cursor-based (WHERE id > last_seen_id):
  Stable; fast; no skipped items on inserts
  Use for: feeds, timelines, any user-facing pagination ← always prefer
```

### Rate Limiting
```
Token bucket  → allows burst, enforces average ← best general choice
Sliding window → no boundary burst, O(1) storage
Fixed window  → simple, boundary burst exploit
```

### Fan-out
```
Fan-out on write (push):
  Pre-compute on write → fast reads
  Problem: write amplification for users with many followers

Fan-out on read (pull):
  Compute at read time → no write amplification
  Problem: slow reads for users following many people

Hybrid:
  Write fan-out for normal users
  Read fan-out (merge at read time) for celebrities (> 1M followers)
```

### Distributed Locking
```
Use Redis SETNX (SET if Not eXists) with TTL:
  lock = SETNX "lock:{resource}" 1 EX 30s
  if lock acquired → do work → DEL lock
  if not → retry or return 409

Pitfall: don't use DB row locks for distributed systems (deadlock, connection saturation)
```

---

## Patterns by Problem Type

| Problem | Pattern | Key component |
|---|---|---|
| Read-heavy | Cache-aside + CDN | Redis, CloudFront |
| Write-heavy | Async ingestion + batch | Kafka, Spark |
| Real-time fan-out | Pub/sub + WebSocket | Kafka, WebSocket server |
| Geo-spatial | Geohash index | Redis GEOADD, custom quad-tree |
| Full-text search | Inverted index | Elasticsearch |
| Horizontal DB scale | Consistent hashing shards | Cassandra, DynamoDB |
| Distributed transaction | Saga pattern | Kafka + compensating events |
| Deduplication | Bloom filter + idempotency key | Redis, DB unique constraint |
| Scheduling / delayed jobs | Priority queue + scheduler | Redis sorted set, SQS delay |

---

## Interview Tips (Classical)

**Always mention:**
- Why you chose this specific DB (not "I'll use MySQL")
- How you shard (partition key choice matters — bad key = hotspot)
- Cache invalidation strategy (not just "we cache it")
- At least one failure scenario and how to recover

**Interviewers probe:**
- "What if that node goes down?"
- "How do you handle a hot partition?"
- "What if two users try to book the last seat simultaneously?"
- "How does the cache stay consistent with the DB?"
- "What if Kafka falls behind?"

**Quick answers:**
- Node down → replica takes over (replication); client retries with backoff
- Hot partition → compound partition key (user_id + date); virtual nodes
- Race condition → optimistic locking (`WHERE version = ?`); DB unique constraint
- Cache consistency → TTL + event-driven invalidation on write
- Kafka lag → add partitions + consumers; alert on consumer lag metric
