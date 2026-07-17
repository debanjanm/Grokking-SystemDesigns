# Twitter Feed (Social Media Timeline)

Design a system that generates and serves a personalized timeline of tweets for 300M daily active users.

---

## Requirements

### Functional Requirements
- Users can post tweets (text, images, links)
- Users see a home timeline: tweets from people they follow, reverse-chronological (or ranked)
- Users can follow/unfollow other users
- Tweets support likes, retweets, replies
- Push new tweets to followers in near real-time
- Support notifications (mention, like, retweet)

### Non-Functional Requirements
- Latency: timeline load < 200ms p99
- Availability: 99.99% — timeline must load even during spikes
- Consistency: eventual — timeline lag up to 5s acceptable
- Security & Privacy: private accounts; blocked users cannot see tweets; DMs encrypted
- Scalability: 300M DAU, 500M tweets/day, 150M timeline requests/day

### Out of Scope
- Search (separate Elasticsearch service)
- Ads insertion
- DM system
- Trending topics

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| DAU | 300M |
| Tweets posted/day | 500M (~6,000/sec) |
| Timeline reads/day | 150M (~1,750/sec avg, 10K/sec peak) |
| Read:write ratio | ~300:1 |

### Storage Estimation
| Metric | Value |
|---|---|
| Avg tweet size | 300 bytes |
| Tweets/day | 500M → 150 GB/day |
| Tweets (5yr) | ~270 TB |
| Media (images) | ~10× text → ~1.5 PB/5yr (on object store) |

### Memory Estimation
| Metric | Value |
|---|---|
| Timeline cache per user (800 tweet IDs × 8B) | ~6 KB |
| Cache for 30M active users | ~180 GB |
| Hot tweet cache | ~10 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Write throughput | ~1.8 MB/s |
| Read throughput (timeline, 20 tweets × 300B) | ~100 MB/s |

---

## High-Level Design

### Architecture Style
**Event-driven fan-out + Pre-computed timelines** — tweets fan out to followers' timeline caches asynchronously; reads served from cache (not computed on the fly). Hybrid approach for celebrity accounts (fan-out-on-read for users with > 1M followers).

### Architecture Diagram

**Write path (tweet post):**
```
User → Tweet Service → DB (Cassandra) → Kafka (tweet.created)
                                              ↓
                                       Fan-out Service
                                       ├── Fetch follower list
                                       └── Push tweet_id to each follower's timeline cache (Redis)
                                              ↓ async
                                       Notification Service
```

**Read path (timeline load):**
```
User → Timeline Service
            ├── Redis: GET timeline:{user_id} → list of tweet_ids
            ├── Tweet Fetcher: MGET tweet:{id} for each tweet_id (Redis)
            │         ↓ cache miss
            │    Cassandra (tweet data)
            └── Hydrate: user profiles, media URLs, like counts
                    ↓
              Response (20 tweets)
```

### Core Components
| Component | Responsibility |
|---|---|
| Tweet Service | Validate + persist tweets; publish to Kafka |
| Fan-out Service | Consume tweet events; push to follower timeline caches |
| Timeline Service | Serve pre-computed timeline from Redis; hydrate tweet data |
| Follow Service | Manage follow/unfollow graph; invalidate caches |
| Notification Service | Push notifications for mentions, likes, retweets |
| User Graph Store | Follower/following lists (Cassandra or graph DB) |
| Media Service | Upload images/video to object store (S3); return CDN URLs |

### Data Flow

**Post tweet:**
1. User POSTs tweet → Tweet Service validates (length, spam)
2. Tweet written to Cassandra; tweet_id generated (Snowflake)
3. Event published to Kafka: `{ tweet_id, author_id, timestamp }`
4. Fan-out Service consumes event:
   - Fetch author's follower list (up to 75K followers → batch)
   - For each follower: `LPUSH timeline:{follower_id} tweet_id` + `LTRIM` to 800
5. Notification Service sends push notification to mentioned users

**Load timeline:**
1. User requests timeline → Timeline Service
2. `LRANGE timeline:{user_id} 0 19` → 20 tweet_ids from Redis
3. `MGET tweet:{id1} tweet:{id2} ...` → tweet data from Redis (or Cassandra on miss)
4. Hydrate: author profile, media URL from CDN, like/retweet counts
5. Return to client; client paginates via cursor

### Key Design Decisions
- **Fan-out on write (push model):** Pre-compute timelines at write time. Reads are fast (pure cache lookup). Tradeoff: write amplification — 1 tweet from user with 10K followers → 10K Redis writes.
- **Fan-out on read for celebrities:** User with 10M followers → 10M Redis writes per tweet is too expensive. Instead: skip fan-out for users > 1M followers; at read time, merge celebrity tweets into timeline. Hybrid approach used by Twitter.
- **Timeline as list of IDs (not full tweets):** Store only tweet_ids in timeline cache. Tweet data fetched separately — deduplicated across timelines, updatable (like count changes without re-fanning).
- **Cassandra for tweets:** Write-heavy, no complex queries, time-series access (newest first). Partition by author_id, cluster by timestamp.
- **Redis for timeline cache:** LPUSH/LRANGE for timeline list. Fixed size (800 entries) via LTRIM — users rarely scroll past 800.

---

## Low-Level Design

### Data Models
```
tweets (Cassandra):
  tweet_id      BIGINT    PK (Snowflake — time-ordered)
  author_id     UUID
  content       TEXT      (max 280 chars)
  media_urls    LIST<TEXT>
  reply_to_id   BIGINT    (nullable)
  retweet_of_id BIGINT    (nullable)
  created_at    TIMESTAMP
  like_count    COUNTER
  retweet_count COUNTER

follows (Cassandra — two tables for bidirectional lookup):
  followers_of (user_id → follower list):
    user_id     UUID      PK
    follower_id UUID      clustering key
    followed_at TIMESTAMP

  following_of (user_id → following list):
    user_id     UUID      PK
    followee_id UUID      clustering key

timeline_cache (Redis):
  key:   "timeline:{user_id}"
  type:  List (tweet_ids, newest first)
  max:   800 entries (LTRIM on every LPUSH)
  TTL:   7 days (inactive users not cached)

tweet_cache (Redis):
  key:   "tweet:{tweet_id}"
  value: serialized tweet object (MessagePack)
  TTL:   24 hours (hot tweets stay; old tweets evicted)
```

### API Design
```
POST /tweets
Request:  { "content": "Hello world", "media_ids": [], "reply_to": null }
Response: { "tweet_id": "...", "created_at": "..." }

GET /timeline?cursor=&limit=20
Response: {
  "tweets": [ { tweet_id, content, author, like_count, created_at, ... } ],
  "next_cursor": "..."
}

POST /follow/{user_id}
Response: 204

DELETE /follow/{user_id}
Response: 204

POST /tweets/{tweet_id}/like
Response: { "like_count": 1042 }
```

### Component Deep-Dives

#### Fan-out Service — Celebrity Handling
```
On tweet.created event:
  follower_count = get_follower_count(author_id)

  if follower_count <= 1_000_000:
    # Fan-out on write (push)
    followers = get_followers(author_id)          # paginated from Cassandra
    for batch in chunks(followers, 100):
      pipeline = redis.pipeline()
      for follower_id in batch:
        pipeline.lpush(f"timeline:{follower_id}", tweet_id)
        pipeline.ltrim(f"timeline:{follower_id}", 0, 799)
      pipeline.execute()

  else:
    # Skip fan-out; mark as celebrity tweet
    redis.sadd("celebrity_tweets:{author_id}", tweet_id)
    redis.expire(f"celebrity_tweets:{author_id}", 86400)
```

At read time — merge celebrity tweets:
```
def get_timeline(user_id):
    tweet_ids = redis.lrange(f"timeline:{user_id}", 0, 99)   # own feed

    # Merge celebrity tweets for celebrities user follows
    followed_celebrities = get_followed_celebrities(user_id)  # cached
    for celebrity_id in followed_celebrities:
        celeb_ids = redis.lrange(f"celebrity_tweets:{celebrity_id}", 0, 19)
        tweet_ids.extend(celeb_ids)

    # Sort by tweet_id (Snowflake → time-ordered), return top 20
    return sorted(tweet_ids, reverse=True)[:20]
```

#### Timeline Pagination (Cursor-based)
```
Cursor = last seen tweet_id (not page number)
  → Stable: new tweets inserted at top don't shift page 2
  → GET /timeline?cursor=1234567890

Server:
  idx = redis.lpos("timeline:{user_id}", cursor)
  tweet_ids = redis.lrange("timeline:{user_id}", idx+1, idx+20)
```

#### Like Counter
```
Likes are high-write (viral tweet → millions of likes/hr)
Do NOT update Cassandra synchronously per like:

Solution: Redis INCR counter + async flush
  INCR "like_count:{tweet_id}"
  Async job: every 30s → flush Redis counters to Cassandra
  Read: always from Redis counter (not Cassandra)
```

#### Guardrails / Safety Layer
- **Spam detection:** content analysis on post; rate limit 100 tweets/day per user
- **Blocked users:** block list checked at fan-out (don't push to blocked users) and at read (filter from timeline)
- **Private accounts:** fan-out only to approved followers; access check at read
- **Media validation:** virus scan + NSFW classifier on upload before CDN distribution
- **Character limit:** enforced server-side (not just client) — 280 chars hard limit

### Algorithms & Strategies
- **Tweet ID generation:** Snowflake (41-bit timestamp + datacenter + sequence) — time-ordered, no coordination, fits in BIGINT
- **Timeline ordering:** tweet_ids are Snowflake → naturally time-ordered; sort by id DESC for reverse-chronological
- **Ranked timeline (optional):** after fetching 100 tweet_ids from cache → ML ranking model scores → return top 20. Cache stores more than displayed to allow reranking.
- **Caching strategy:** write-through on tweet creation (tweet cached immediately); LRU eviction; hot tweets (viral) never evicted (TTL refreshed on read)
- **Fanout batching:** Redis pipeline per batch of 100 followers → single round-trip; Kafka consumer parallelized by author_id hash

### Security Design
- AuthN: JWT session tokens; 24hr expiry with refresh
- AuthZ: private account follow requests require approval; block list enforced at DB + cache
- DMs: end-to-end encrypted (separate key pair per user, managed by key service)
- Media: signed CDN URLs (expire 1hr); direct S3 upload blocked
- Rate limiting: 100 tweets/day, 1000 API calls/hr per user (token bucket, Redis)
- PII: email/phone stored encrypted; GDPR deletion cascades to tweets + timeline caches

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Timeline load latency (p99) | Histogram | > 200ms |
| Fan-out lag | Gauge | > 5s from tweet post |
| Timeline cache hit rate | Gauge | < 95% |
| Fan-out queue depth (Kafka) | Gauge | > 100K (fan-out falling behind) |
| Like counter flush lag | Gauge | > 60s |
| Tweet write latency | Histogram | > 500ms |

### Logging
- Per tweet: tweet_id, author_id_hash, char_count, media_count, latency
- Per timeline load: user_id_hash, tweet_count_returned, cache_hit, latency
- Per fan-out: tweet_id, follower_count, fan-out_duration_ms, batches

### Tracing
- Trace: post_tweet → kafka_publish → fan-out → redis_write
- Trace: timeline_load → redis_read → tweet_hydrate → profile_fetch
- Sample: 1% normal; 100% on latency > 500ms

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Fan-out lag > 5s | Kafka consumer lag | Scale fan-out workers |
| Timeline cache miss spike | < 90% hit rate | Check Redis memory/eviction |
| Celebrity tweet not appearing | Fan-out skipped but merge broken | Alert timeline team |
| Tweet write failure | > 0.1% error rate | Page on-call |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Fan-out on write (push) | Fast reads (~1ms) | Write amplification; storage per follower |
| Celebrity fan-out on read | No write amplification for celebrities | Merge logic at read time; slight latency |
| Tweet IDs in timeline (not full tweet) | Deduplication; updatable counts | Extra lookup per tweet at read |
| Cassandra over MySQL | Write throughput, horizontal scale | No joins; eventual consistency |
| Redis timeline list (fixed 800) | Bounded memory | Users can't load tweets older than 800 |
| Eventual consistency (fan-out lag) | High write throughput | New tweet may take up to 5s to appear |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Fan-out lag spike | Viral tweet from large account | Auto-scale Kafka consumers; rate-limit fan-out per tweet |
| Timeline cache cold | Redis restart | Pre-warm from Cassandra for active users on startup |
| Celebrity tweets missing | Fan-out-on-read merge fails | Fallback: include celebrity tweets even if merge service down (degrade gracefully) |
| Like counter lost | Redis crash before flush | Redis AOF persistence; reconcile from Kafka event log |
| Follower list too large | User with 100M followers | Paginate follower fetch; fan-out uses multiple Kafka partitions |

---

## Feedback Loop & Improvements
- Ranked timeline: collect dwell time, likes, retweets → LTR model ranks top 100 cached tweets → return top 20
- A/B: reverse-chronological vs ranked — measure engagement rate
- Notification tuning: ML model scores notification worth sending → reduces notification fatigue

---

## Interview Tips

### What Interviewers Expect
- Fan-out on write vs read — must explain both and when to use which (celebrity problem)
- Timeline stored as list of IDs (not full tweet objects) — explain why
- Snowflake for tweet ID — time-ordered, no coordination
- Cassandra for tweets — justify write-heavy, no-join access pattern

### Common Follow-ups
- How to handle a user with 100M followers? Hybrid fan-out: write for normal, read for celebrities
- How to implement ranked timeline? Fetch 100 from cache → ML ranker → return top 20
- How to paginate without skipping new tweets? Cursor-based (last tweet_id), not page-number
- Hot tweet (viral)? Redis handles reads; like counter via INCR not per-write DB update
