# Search Ranking System

Design a search ranking system (like Google Shopping, Amazon Search, or LinkedIn) that retrieves and ranks relevant results for user queries.

---

## Requirements

### Functional Requirements
- Return ranked list of items matching a search query
- Personalize results based on user history and context
- Support filtering (category, price range, rating, availability)
- Handle typos, synonyms, and multi-language queries
- Support autocomplete / query suggestions
- Rank results by blend of relevance + personalization + business signals

### Non-Functional Requirements
- Latency: search results returned < 200ms p99
- Availability: 99.99% — search down = no revenue
- Consistency: index freshness within 1 hour for new/updated items
- Security & Privacy: personalization signals are user PII; query logs sensitive
- Scalability: 10B documents indexed, 100K QPS

### Out of Scope
- Query understanding (intent classification, NER) — assumed upstream
- Ad insertion (separate auction system)
- Image/voice search

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| QPS (avg) | 50,000 |
| QPS (peak) | 100,000 |
| Queries/day | 4B |
| Unique queries/day | ~400M (10% unique) |

### Storage Estimation
| Metric | Value |
|---|---|
| Documents indexed | 10B |
| Avg document size (indexed fields) | 1 KB |
| Inverted index size | ~10 TB |
| Dense vector index (256-dim) | ~10 TB |
| Query log/day | ~400 GB |

### Memory Estimation
| Metric | Value |
|---|---|
| Hot inverted index shards (in-memory) | ~500 GB |
| Dense vector index (ANN, in-memory) | ~5 TB (distributed) |
| Query result cache (Redis) | ~50 GB |
| User feature cache | ~100 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Query ingress | ~5 MB/s |
| Result egress (20 items × 2KB) | ~2 GB/s |

---

## High-Level Design

### Architecture Style
**Multi-stage ranking pipeline + Distributed inverted index** — query fan-out to sharded index; multi-stage (retrieval → L1 ranking → L2 ranking → rerank) balances coverage and quality within latency budget.

### Architecture Diagram

**Query path (online, < 200ms):**
```
User Query
    ↓
Query Service
    ├── Query Processor (spell-check, tokenize, expand synonyms)
    ├── Cache Check (Redis — popular queries)
    │         ↓ miss
    ├── Retrieval (parallel fan-out)
    │    ├── Inverted Index (BM25 — keyword match)   ← Elasticsearch / Lucene
    │    └── Dense Retrieval (ANN — semantic match)  ← Faiss / Qdrant
    │         ↓ merge (RRF fusion)
    ├── L1 Ranker (fast, lightweight — narrow to top-500)
    ├── L2 Ranker (ML model — score top-500, personalized)
    ├── Re-ranker / Business Logic (diversity, promotions, filters)
    └── Response (top-20 items + metadata)
```

**Indexing path (offline + near-real-time):**
```
Item Catalog Changes (Kafka)
    ↓
Document Processor
    ├── Tokenizer + Analyzer
    ├── Dense Encoder (bi-encoder for vector embedding)
    └── Index Writer
         ├── Inverted Index (Elasticsearch)
         └── Vector Index (Faiss — rebuilt or incremental update)
```

**Offline pipeline:**
```
Query Logs + Click Logs (S3)
    ↓
Feature Engineering
    ↓
Ranking Model Training (LTR — LambdaMART / LightGBM)
    ↓
Model Registry → Gradual Rollout
```

### Core Components
| Component | Responsibility |
|---|---|
| Query Processor | Normalize, spell-correct, tokenize, synonym expand |
| Inverted Index (Elasticsearch) | BM25 keyword retrieval; filtered by facets |
| Dense Retrieval Index (Faiss) | Semantic ANN retrieval for conceptual matches |
| L1 Ranker | Fast scoring on top-1000 candidates (heuristic + simple features) |
| L2 Ranker | Full Learning-to-Rank model; personalized scoring on top-500 |
| Business Rules Engine | Diversity, promotions, geolocation boosts, OOS filter |
| Query Cache (Redis) | Cache results for popular/repeated queries |
| Feature Store | User features (click history, preferences) for personalization |

### Data Flow

**Query (online):**
1. User query arrives → spell-check, tokenize, synonym expansion
2. Cache check (Redis) — popular query hit → return immediately
3. Parallel retrieval: BM25 (Elasticsearch) + Dense ANN (Faiss) → merge top-1000 via RRF
4. L1 Ranker: filter to top-500 using BM25 score + item popularity + in-stock signal (< 5ms)
5. L2 Ranker: load user features from Feature Store; score all 500 with LTR model → ordered list (< 50ms)
6. Business rules: apply filters (price range, rating), diversity (max 3 same brand), promoted item boosts
7. Return top-20 items with metadata + facet counts

**Indexing (near-real-time):**
1. Item created/updated → event on Kafka topic `catalog.changes`
2. Document Processor: extract indexed fields, generate dense embedding via bi-encoder
3. Write to Elasticsearch (inverted index) — available within minutes
4. Batch update Faiss index (rebuilt nightly, or incremental IVFPQ update)

### Key Design Decisions
- **Multi-stage ranking:** Single-stage ranking all 10B docs is impossible at 200ms. Stage 1 (retrieval) uses fast approximate methods to get to ~1000 candidates. Stage 2 (ranking) uses expensive ML only on those 1000. Cascading quality improvement with latency control.
- **Hybrid retrieval (BM25 + dense):** BM25 is strong for exact keyword matches ("iPhone 14 Pro case"). Dense retrieval catches semantics ("phone protection" → cases). Hybrid via RRF consistently outperforms either alone.
- **L1 + L2 separation:** L1 is cheap (heuristic score, no user features) — narrows 1000 → 500 fast. L2 is expensive (LTR model with 200 features) — applied to 500 only. Saves L2 compute by 2×.
- **Cache for popular queries:** Top 5% queries account for 60% of traffic (power law). Caching these covers majority of traffic with < 50ms response. Cache invalidated on catalog updates to affected items.
- **Model Selection:** LambdaMART (gradient boosted trees with NDCG optimization) for L2 ranker — industry standard for LTR, interpretable, fast inference. Bi-encoder (BERT-based) for dense retrieval — trained on query-item click pairs.

---

## Low-Level Design

### Data Models
```
items (Elasticsearch index):
  item_id         keyword   (PK)
  title           text      (analyzed: tokenized, stemmed)
  description     text
  category        keyword   (facet)
  brand           keyword   (facet)
  price           float     (range filter)
  rating          float
  review_count    integer
  in_stock        boolean
  embedding       dense_vector[256]  ← dense retrieval
  popularity_7d   float     (updated daily)
  created_at      date

query_logs (Parquet on S3):
  query_id        UUID
  user_id         UUID (hashed)
  query_text      TEXT
  timestamp       TIMESTAMP
  result_item_ids ARRAY<UUID>
  clicked_item_id UUID (nullable)
  position_clicked INTEGER (nullable)

user_features (Feature Store / Redis):
  user_id         UUID
  embedding       float32[256]      ← query-user embedding
  preferred_categories ARRAY<VARCHAR>
  avg_price_range { min, max }
  recent_queries  ARRAY<STRING>     (last 10)
  last_active     TIMESTAMP
```

### API Design
```
GET /search
Query params:
  q=running+shoes
  &user_id=...         (optional, for personalization)
  &filters=category:footwear,price:50-200,rating:4+
  &sort=relevance      (relevance | price_asc | price_desc | newest)
  &limit=20
  &offset=0

Response: {
  "query": "running shoes",
  "corrected_query": null,
  "total_results": 45231,
  "results": [
    {
      "item_id": "...",
      "title": "Nike Air Zoom Pegasus 40",
      "price": 129.99,
      "rating": 4.7,
      "rank_score": 0.94,
      "debug": { "bm25": 0.82, "semantic": 0.91, "personalization": 0.96 }
    }
  ],
  "facets": {
    "category": { "footwear": 45231 },
    "brand": { "Nike": 12000, "Adidas": 8000 }
  },
  "model_version": "v38"
}

GET /autocomplete?q=runn&user_id=...
Response: { "suggestions": ["running shoes", "running shorts", "running watch"] }
```

### Component Deep-Dives

#### Multi-Stage Ranking Pipeline

**Stage 1 — Retrieval (parallel):**
```
Query → BM25 (Elasticsearch)     → top-500 (keyword match)
      → ANN Dense (Faiss HNSW)   → top-500 (semantic match)
                  ↓
           RRF Fusion
           score = Σ 1/(k + rank_i)   k=60
                  ↓
           top-1000 candidates
```

**Stage 2 — L1 Ranker (< 5ms):**
- Heuristic score: `0.4×bm25 + 0.3×popularity + 0.2×in_stock + 0.1×freshness`
- Filter: remove OOS if filter applied
- Output: top-500

**Stage 3 — L2 Ranker / LTR (< 50ms):**
- Load user features from Feature Store
- Build feature vector per (query, item) pair
- Score all 500 with LambdaMART model
- Output: ranked list

**Stage 4 — Business Rules + Re-rank (< 10ms):**
- Apply: diversity (max 3 same brand), promoted item boosts, geo availability
- Apply user filters (price range, rating)
- Output: final top-20

#### Learning to Rank (LTR) — Feature Set
| Feature | Type | Source |
|---|---|---|
| bm25_score | float | Elasticsearch |
| semantic_sim | float | dot(query_emb, item_emb) |
| item_ctr_7d | float | Query log aggregation |
| item_popularity_7d | float | Sales/view count |
| user_category_affinity | float | Feature Store |
| brand_user_affinity | float | Feature Store |
| price_in_user_range | bool | Feature Store |
| position_bias_correction | float | IPS weight |
| item_freshness_days | int | item.created_at |
| query_item_historical_ctr | float | Query-item click table |

Training: pairwise LTR with click data. Label: clicked > shown-but-not-clicked. Position bias corrected via Inverse Propensity Scoring (IPS).

#### Query Cache Strategy
```
Cache key: hash(normalized_query + filters + sort)
Cache value: { items: [...], facets: {...}, model_version }
TTL: 5 minutes (balance freshness vs hit rate)

Invalidation: on catalog update → check which cached queries contain affected item_id → evict
```

Popular query detection: track query frequency in Redis counter; if count > 100/hr → pre-warm cache.

#### Autocomplete
```
Trie or FST (Finite State Transducer) on popular queries:
  - Build from query logs: top 10M queries by frequency
  - Weighted by: query_count × recency_decay × personalization_boost
  - Rebuild nightly
  - Serve from memory (< 1ms)
  - Prefix match: "runn" → ["running shoes", "running watch", ...]
```

Personalized: user's recent search history boosts matching suggestions.

#### Guardrails / Safety Layer
- **Query sanitization:** escape special characters, max query length 512 chars
- **Result safety:** filter items flagged as counterfeit, recalled, or policy-violating before serving
- **Diversity enforcement:** max N items from same seller/brand to prevent monopolization
- **Position bias:** LTR model trained with IPS to correct for top-position click bias
- **Query logging consent:** query logs filtered of PII; user_id hashed; comply with GDPR right-to-erasure

### Algorithms & Strategies
- **BM25 tuning:** `k1=1.2, b=0.75` (Elasticsearch defaults). Boost title field 3× over description.
- **Dense retrieval:** Bi-encoder (DistilBERT fine-tuned on query-item click pairs). Query encoded at request time (< 10ms). Item encoded offline (nightly). HNSW index in Faiss (`ef_search=100, M=32`).
- **LTR model:** LambdaMART (LightGBM with rank objective). 200 trees, max_depth=6. NDCG@10 as optimization metric. Retrained weekly on 90d click logs.
- **Position bias correction:** IPS weights = `1 / P(click | position)`. P(click|position) estimated from randomized ranking experiments (periodically shuffle results to calibrate).
- **Query expansion:** WordNet synonyms + fine-tuned word2vec on product corpus. Example: "sneakers" → also search "trainers", "athletic shoes".
- **Caching strategy:** LRU cache in Redis. Cache key normalized (lowercase, stemmed). Pre-warm top-1000 queries at startup. TTL 5 min. Invalidate on affected item updates.

### Security Design
- AuthN/AuthZ: search API public (no auth); personalization requires authenticated session token
- Query logs: user_id hashed (SHA-256 + salt); raw queries retained 30 days then deleted
- GDPR: right-to-erasure deletes user features + purges hashed user_id from query logs
- Index access: index write path restricted to indexing service (internal only)
- Rate limiting: 100 RPS per IP (anonymous); 1000 RPS per authenticated user

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Search latency (p99) | Histogram | > 200ms |
| Cache hit rate | Gauge | < 50% |
| NDCG@10 (online) | Gauge | > 5% drop vs 7d avg |
| Zero-result rate | Gauge | > 1% (possible index issue) |
| Index freshness lag | Gauge | > 1hr since last update |
| L2 ranker latency | Histogram | > 50ms |
| ANN query latency | Histogram | > 20ms |

### Logging
- Per query: query_hash, user_id_hash, result_count, cache_hit, latency, model_version, top_item_ids
- Per click: query_hash, item_id, position, timestamp
- Per index update: item_id, operation (upsert/delete), lag_ms, index_version

### Tracing
- Trace spans: query_proc → cache_check → retrieval_bm25 → retrieval_dense → fusion → l1_rank → l2_rank → rules → response
- Key: retrieval latency split between BM25 and ANN
- Sample: 5% normal; 100% on latency > 500ms

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Search latency spike | p99 > 200ms | Check index shard health, cache hit rate |
| Zero-result rate spike | > 2% | Possible indexing failure |
| NDCG drop | > 10% vs 7d avg | Investigate model drift or index corruption |
| Cache hit rate drop | < 40% | Check Redis memory, query distribution shift |
| Index lag | > 1hr | Check Kafka consumer, indexing service |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Multi-stage ranking | Latency budget control per stage | Pipeline complexity; cascading errors |
| Hybrid retrieval (BM25 + ANN) | Recall for both keyword and semantic | Two retrieval systems to maintain |
| LambdaMART over neural ranker | Fast inference, interpretable | Marginal quality on complex query-item relationships |
| Query result cache | < 50ms for popular queries | Stale results (mitigated by 5min TTL + invalidation) |
| Offline LTR retraining (weekly) | Stable, high-quality model | Slow to adapt to new item catalog or trends |
| IPS position bias correction | Unbiased training signal | Requires randomization experiments (revenue risk) |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Index shard down | Elasticsearch node failure | Replicated shards (RF=2); reroute to replica |
| ANN index stale | Nightly rebuild failed | Alert; serve BM25-only fallback (disable dense retrieval) |
| Feature Store down | Redis failure | Serve non-personalized results; degrade gracefully |
| Cache stampede | Popular query cache expires | Mutex lock on cache rebuild; probabilistic early refresh |
| Zero results | Index out of sync | Query expansion fallback (drop filters progressively) |
| Model drift | Catalog changes, seasonal shift | Weekly retraining + NDCG monitoring alert |

---

## Feedback Loop & Improvements
- Click-through data is the training signal — collected continuously, used in weekly LTR retraining
- Implicit feedback: dwell time, add-to-cart, purchase stronger signal than clicks
- Online eval: interleaving experiment (blend two rankers in single SERP) for faster A/B signal than full experiment
- Continuous index updates: item popularity scores updated daily from sales/view logs → re-indexed
- Query clustering: group rare queries with similar popular queries for better coverage on tail queries

---

## Interview Tips

### What Interviewers Expect
- Multi-stage pipeline with clear latency budget allocation (retrieval < 20ms, L1 < 5ms, L2 < 50ms, rules < 10ms = total < 200ms)
- Hybrid retrieval — BM25 + dense — is industry standard; explain RRF fusion
- LTR (Learning to Rank) concept — pairwise training, NDCG optimization
- Position bias correction — critical for unbiased training data

### Common Follow-ups
- How to handle tail queries (never-before-seen)? Query expansion + semantic retrieval covers them; fallback to popularity
- Personalization without user history? Context-only (device, time, location) features
- How to A/B test ranking models? Interleaving experiments (faster signal); or full 50/50 split
- Index at 100B documents? Shard inverted index across 1000 nodes; hierarchical ANN index (IVF partition)
