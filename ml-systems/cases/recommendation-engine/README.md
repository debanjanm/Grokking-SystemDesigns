# Recommendation Engine

Design a personalized recommendation system (like Netflix, Amazon, or Spotify) that surfaces relevant items to users in real time.

---

## Requirements

### Functional Requirements
- Recommend items (movies, products, songs) personalized per user
- Support homepage feed, "similar items", and "users also bought" surfaces
- Update recommendations as user interacts (clicks, purchases, ratings)
- Handle new users with no history (cold start)
- Support business rules (boost promoted items, exclude out-of-stock)

### Non-Functional Requirements
- Latency: recommendation API < 100ms p99
- Availability: 99.99% — homepage must load even if ML system degrades
- Consistency: eventual — recommendations refresh within minutes of new interactions
- Security & Privacy: user interaction data PII-sensitive; GDPR-compliant deletion
- Scalability: 100M users, 10M items, 1B interactions/day

### Out of Scope
- Ad targeting (separate system)
- Search ranking (separate case)
- A/B experimentation platform (integrated but not designed here)

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| DAU | 50M |
| Recommendation requests/day | 500M (10 surfaces × 50M users) |
| Requests/sec (peak) | ~20,000 |
| Interaction events/day | 1B (clicks, watches, purchases) |

### Storage Estimation
| Metric | Value |
|---|---|
| User embedding (256-dim float32) | 1 KB/user |
| Item embedding (256-dim float32) | 1 KB/item |
| User embeddings (100M users) | ~100 GB |
| Item embeddings (10M items) | ~10 GB |
| Interaction log/day | ~200 GB |
| Interaction log (2yr retention) | ~150 TB |

### Memory Estimation
| Metric | Value |
|---|---|
| Item embeddings in-memory (ANN index) | ~10 GB |
| Feature cache (Redis) | ~50 GB |
| Pre-computed candidate cache | ~20 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Interaction event ingestion | ~2 MB/s |
| Recommendation response (20 items) | ~5 KB × 20K RPS = ~100 MB/s |

### Cost Estimation
| Metric | Value |
|---|---|
| Model training (daily) | GPU cluster, ~4hr batch job |
| Online inference | CPU-based (embedding lookup + ANN), cheap |
| Feature store reads | ~500M reads/day |

---

## High-Level Design

### Architecture Style
**Lambda Architecture** — batch layer (daily model retraining) + speed layer (real-time feature updates + online scoring). Pre-computed candidates for latency; online re-ranking for personalization.

### Architecture Diagram

**Offline Pipeline (batch):**
```
Interaction Logs (S3/HDFS)
    ↓
Feature Engineering (Spark)
    ↓
Model Training (collaborative filtering / two-tower)
    ↓
Embedding Export
    ├── User Embeddings → Feature Store
    ├── Item Embeddings → ANN Index (Faiss/Qdrant)
    └── Model Artifacts → Model Registry
```

**Online Pipeline (real-time):**
```
User Request
    ↓
Recommendation Service
    ├── Feature Fetcher (Feature Store → user features, context)
    ├── Candidate Generator (ANN search on item embeddings)
    ├── Ranker (lightweight ML model, scores candidates)
    ├── Business Rules Filter (exclude OOS, apply boosts)
    └── Response (top-N items)
         ↓
    Interaction Logger → Kafka → Feature Store (real-time update)
```

### Core Components
| Component | Responsibility |
|---|---|
| Feature Store | Serve precomputed user/item features; low-latency reads |
| ANN Index | Approximate nearest neighbor search over item embeddings |
| Candidate Generator | Retrieve top-K candidate items per user via ANN |
| Ranker | Score and re-rank candidates using personalization features |
| Business Rules Engine | Apply filters (OOS, geography, promotions) post-ranking |
| Interaction Logger | Capture click/view/purchase events → real-time + batch pipelines |
| Training Pipeline | Daily batch job: feature eng → model train → embedding export |

### Data Flow

**Online request:**
1. User hits recommendation surface → request arrives with user_id + context (device, time, surface)
2. Feature Fetcher pulls user embedding + recent activity features from Feature Store (Redis/Feast)
3. Candidate Generator: ANN search on item embedding index → top-500 candidates
4. Ranker: lightweight gradient-boosted model scores all 500 candidates using user + item + context features
5. Business Rules: filter OOS items, apply category diversity, boost promoted items
6. Return top-20 items to client
7. Async: log impression events to Kafka

**Offline model update:**
1. Kafka → S3: interaction events land in data lake hourly
2. Daily Spark job: join interactions + item metadata → training dataset
3. Two-tower model trains on user-item pairs → user/item embeddings
4. Embeddings exported → Feature Store (user) + ANN index (item) rebuilt
5. New model artifacts pushed to Model Registry → online ranker updated (canary → full rollout)

### Key Design Decisions
- **Two-stage: candidate generation + ranking:** ANN search is fast but coarse (cosine similarity only). Ranker uses richer features (recency, context, diversity) but on smaller candidate set. Two stages balance speed and quality.
- **Pre-computed user embeddings:** Computing embeddings at request time is too slow. Batch-compute daily; store in Feature Store. Real-time signals (last 5 clicks) added as separate features at serving time.
- **ANN over exact NN:** Exact nearest neighbor search on 10M items is ~1s. HNSW ANN gives top-500 in < 5ms with ~95% recall — acceptable loss for speed gain.
- **Lambda architecture:** Batch covers long-term preferences; speed layer adds recent signals. Pure streaming would miss long-range patterns; pure batch misses recency.
- **Model Selection:** Two-tower neural network for candidate generation (learns user/item embeddings jointly). LightGBM for ranking (fast inference, handles tabular features, interpretable).

---

## Low-Level Design

### Data Models
```
interactions (Cassandra / columnar store):
  user_id       UUID      partition key
  timestamp     TIMESTAMP clustering key (DESC)
  item_id       UUID
  event_type    ENUM (view, click, purchase, rating, skip)
  surface       VARCHAR   (homepage, similar_items, search)
  session_id    UUID
  rating        FLOAT     (nullable)

items (PostgreSQL):
  item_id       UUID      PK
  title         TEXT
  category      VARCHAR
  price         DECIMAL
  in_stock      BOOLEAN
  embedding_version INT
  created_at    TIMESTAMP

feature_store (Redis — online):
  key: "user:{user_id}:features"
  value: {
    embedding: float32[256],
    recent_categories: [cat1, cat2, ...],   # last 10 interactions
    avg_price_range: float,
    last_active_ts: timestamp
  }
  TTL: 24 hours

ann_index (Faiss HNSW — in-memory):
  vectors: item_id → float32[256]
  metadata: item_id, category, price_bucket
```

### API Design
```
GET /recommendations
Query params: user_id, surface (homepage|similar|also_bought), item_id (for similar), limit (default 20)
Response: {
  "items": [
    { "item_id": "...", "title": "...", "score": 0.92, "reason": "because you watched X" }
  ],
  "model_version": "v42",
  "served_at": "2026-07-17T10:00:00Z"
}

POST /interactions
Request: { "user_id": "...", "item_id": "...", "event_type": "click", "surface": "homepage" }
Response: 204 No Content

GET /items/{item_id}/similar
Response: same as /recommendations
```

### Component Deep-Dives

#### Two-Tower Model Architecture
```
User Tower:                    Item Tower:
  user_id embedding            item_id embedding
  + age_bucket                 + category embedding
  + genre_prefs                + price_bucket
  + recent_items (avg pool)    + popularity_score
       ↓                            ↓
  Dense layers (256-dim)      Dense layers (256-dim)
       ↓                            ↓
  User Embedding ─────── dot product ──── Score
                    (trained to maximize for positive pairs)
```

Training: in-batch negative sampling. Loss: softmax cross-entropy over batch.

#### Online Ranker Features
| Feature | Type | Source |
|---|---|---|
| user_item_embedding_sim | float | dot(user_emb, item_emb) |
| user_category_affinity | float | Feature Store |
| item_popularity_7d | float | Feature Store |
| item_recency | float | item.created_at |
| context_time_of_day | int | request context |
| context_device | category | request context |
| user_price_affinity | float | Feature Store |
| item_in_price_range | bool | user_price_affinity × item_price |

#### Cold Start Strategy
| Scenario | Strategy |
|---|---|
| New user, no history | Popularity-based recommendations; ask onboarding preferences |
| New user, onboarding prefs | Content-based: find items matching stated categories |
| New item, no interactions | Content-based embedding from item metadata; boost in explore slot |
| New user after 3 interactions | Switch from popularity → collaborative filtering (hybrid) |

#### Guardrails / Safety Layer
- **Filter layer:** remove OOS items, adult content (if not opted in), GDPR-deleted items
- **Diversity enforcement:** max 3 items from same category in top-20
- **Freshness boost:** items > 6 months old get slight score penalty (configurable)
- **Feedback loop prevention:** don't over-recommend same item user already purchased
- **Exposure bias:** down-rank items that appear in every recommendation (popularity trap)

### Algorithms & Strategies
- **ANN search:** HNSW index (Faiss). `ef_search=50`, `M=16`. Rebuilt nightly from new item embeddings. Served from memory — 10GB fits comfortably.
- **Ranking:** LightGBM with 200 trees. Training: offline on impression + click logs with click-through rate as label. Inference: ~1ms per candidate batch of 500.
- **Real-time feature update:** Interaction events → Kafka → Flink consumer → Redis INCR / list push. User embedding regenerated by lightweight online model every hour for active users.
- **Diversity:** Maximal Marginal Relevance (MMR) post-ranking: greedily select items that are relevant but dissimilar to already-selected items.
- **Business rules:** Rule DSL evaluated in order: filter → boost → diversity → cap. Rules stored in config, hot-reloadable without deploy.

### Security Design
- AuthN/AuthZ: recommendation API authenticated via user session token; user_id validated server-side
- PII: user_id in interaction logs pseudonymized; real user_id only in secure mapping table
- GDPR deletion: delete from interactions table + trigger embedding recomputation + purge Feature Store
- Data residency: interaction logs stored in region of user (EU/US separation)
- Encryption at rest: S3, Redis, Cassandra all encrypted
- Encryption in transit: TLS 1.3

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Recommendation latency (p99) | Histogram | > 100ms |
| ANN index latency | Histogram | > 10ms |
| Feature Store read latency | Histogram | > 5ms |
| Click-through rate (CTR) | Gauge | > 20% drop vs 7d avg |
| Coverage (% items ever recommended) | Gauge | < 10% (popularity trap) |
| Cold start user CTR | Gauge | Separate tracking |
| Model staleness | Gauge | > 25hr since last train |

### Logging
- Per request: user_id_hash, surface, candidate_count, latency, model_version, top_item_ids
- Per interaction: user_id_hash, item_id, event_type, surface, position_shown
- Do NOT log: raw user PII, full interaction history per request

### Tracing
- Trace spans: feature_fetch → candidate_gen → ranking → rules → response
- Annotate: ANN latency, feature cache hit/miss, ranker score distribution
- Sample: 1% normal (high volume); 100% on latency > 200ms or cold start users

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| CTR drop | > 20% below 7d avg | Investigate model freshness, feature drift |
| Recommendation latency spike | p99 > 100ms | Check ANN index, Feature Store health |
| Model not updated | > 25hr | Check training pipeline, alert ML team |
| Feature Store miss rate | > 5% | Check Redis memory, cold start surge |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Two-stage (generate + rank) | Speed (ANN fast) + quality (ranker rich) | Pipeline complexity |
| Pre-computed user embeddings | Low serving latency | Stale for inactive users (refreshed daily) |
| Lambda architecture | Long-term + short-term signals | Two systems to maintain (batch + stream) |
| ANN over exact NN | < 5ms candidate retrieval | ~5% recall loss |
| LightGBM ranker over neural | Fast inference, interpretable | Less expressive than deep ranking models |
| Diversity via MMR | Prevents category saturation | Slight relevance drop for niche users |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Stale recommendations | Training pipeline failure | Alert on model age > 25hr; fall back to previous model |
| Feature Store cold | Redis OOM or restart | Pre-warm from Feature Store snapshot; fall back to item popularity |
| ANN index OOM | Item catalog growth | Shard index by category; increase instance memory |
| Popularity trap | Model over-indexes on popular items | Coverage metric alert; diversity enforcement in rules |
| Feedback loop | Model trained on its own recommendations | Inject random exploration items (epsilon-greedy, 5%) |
| Cold start flood | New user surge (product launch) | Popularity fallback pre-cached; no Feature Store needed |

---

## Feedback Loop & Improvements
- Online A/B tests: 50/50 split new vs old model; measure CTR, watch-time, purchase rate
- Offline eval: precision@K, recall@K, NDCG on held-out interaction set before any online test
- Exploration: epsilon-greedy (5% random items) prevents filter bubble + gathers training signal for new items
- Continuous learning: high-signal events (purchase) trigger near-real-time embedding update for active users
- Eval cadence: offline metrics gated before A/B; A/B runs 2 weeks minimum for statistical significance

---

## Interview Tips

### What Interviewers Expect
- Two-stage pipeline (candidate gen → ranking) is industry standard — must explain why
- Cold start handling — interviewers always ask; have content-based + popularity fallback ready
- Feature Store concept — what it is, why low-latency matters, online vs offline store
- Training/serving skew — features used in training must exactly match features at serving time

### Common Follow-ups
- How to handle feedback loop (filter bubble)? Exploration + coverage metrics
- How to evaluate offline? NDCG, precision@K on held-out set; but online A/B is ground truth
- Real-time personalization within session? Flink consumer updates recent-activity features in Redis per event
- Multi-armed bandit vs full retraining? Bandit for exploration/exploitation balance on new items; full retrain for embedding updates
