# Pattern: Feature Store

A centralized repository for storing, sharing, and serving ML features — with consistent access for both offline training and online inference.

---

## What It Is

Feature Store solves the **training/serving skew** problem: features computed differently at training time (batch) vs serving time (real-time) cause silent model degradation.

It provides two access patterns over the same feature definitions:
- **Offline store:** batch reads for training dataset generation (historical, point-in-time correct)
- **Online store:** low-latency reads (< 5ms) for real-time inference

---

## When to Use

| Use | Avoid |
|---|---|
| Multiple models share same features | Single model, single team |
| Training/serving skew is a known problem | Features trivially computed at serving time |
| Features expensive to compute (joins, aggregations) | Prototype / experimental stage |
| Regulatory requirement for feature audit trail | Batch-only inference (no latency requirement) |
| Multiple teams contributing features | |

---

## Core Concepts

### Feature vs Raw Data
Raw data (events, transactions) → Feature engineering → Features (pre-aggregated, ready for model)

Example:
- Raw: list of transactions per user
- Feature: `user_txn_count_7d`, `user_avg_amount_30d` (pre-aggregated)

### Point-in-Time Correctness
Training datasets must use feature values **as they existed at the time of the label event** — not current values. Prevents data leakage.

```
Label event: purchase at T=100
user_txn_count_7d at T=100 = 12   ← correct
user_txn_count_7d at T=now = 47   ← leakage
```

### Online vs Offline Store
```
Offline Store (S3 / Data Warehouse):
  - Historical feature values with timestamps
  - Used for: training data generation, backfills
  - Latency: seconds to minutes (batch reads)

Online Store (Redis / DynamoDB):
  - Latest feature values per entity
  - Used for: real-time inference
  - Latency: < 5ms
```

---

## Architecture

```
Data Sources (events, DB, streams)
    ↓
Feature Pipelines (Spark / Flink)
    ├── Batch Pipeline (daily/hourly)    → Offline Store (S3 / Redshift)
    └── Stream Pipeline (Flink / Kafka)  → Online Store (Redis)
                                              ↓
Training:                           Serving:
  Offline Store                       Online Store
  + Point-in-time join                + Sub-5ms read
  → Training Dataset                  → Model Inference
```

---

## Implementation Details

### Feature Definition (as code)
```python
# Feast-style feature definition
user_stats = FeatureView(
    name="user_stats",
    entities=["user_id"],
    ttl=timedelta(days=7),
    features=[
        Feature("txn_count_7d", ValueType.INT64),
        Feature("avg_amount_30d", ValueType.DOUBLE),
        Feature("preferred_category", ValueType.STRING),
    ],
    batch_source=BigQuerySource(table="analytics.user_stats"),
    stream_source=KafkaSource(topic="user-events"),
)
```

### Point-in-Time Join
```sql
-- Training dataset: join features at label event time
SELECT
    label.user_id,
    label.event_time,
    label.label,
    feat.txn_count_7d,
    feat.avg_amount_30d
FROM labels
ASOF JOIN feature_history feat
  ON labels.user_id = feat.user_id
  AND feat.timestamp <= labels.event_time   -- ← point-in-time
ORDER BY feat.timestamp DESC
LIMIT 1
```

### Online Serving — Redis Schema
```
Key:    "feature:{entity_type}:{entity_id}"
Value:  { feature_name: value, ... }   (MessagePack / protobuf serialized)
TTL:    feature-specific (e.g., 24h for user features, 1h for real-time signals)

Batch read (for model serving — multiple entities):
  MGET feature:user:u1 feature:user:u2 ... (pipeline)
  → single round trip, ~1ms for 100 entities
```

### Feature Pipeline Types

| Type | Compute | Latency | Use for |
|---|---|---|---|
| Batch | Spark (daily/hourly) | Hours to minutes | Long-window aggregations (30d, 90d) |
| Near-real-time | Flink (seconds lag) | Seconds | Session features, hourly counts |
| Real-time | In-request computation | Milliseconds | Current cart, live context |
| On-demand | Computed at serving time | Milliseconds | Features needing request context |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Centralized feature store | Reuse, consistency, no skew | Infrastructure overhead |
| Redis for online store | Sub-5ms reads | Cost; limited to features that fit in memory |
| Pre-compute features | Fast serving | Storage cost; stale for fast-changing entities |
| Point-in-time joins (offline) | No data leakage | Expensive query (ASOF join on large tables) |
| Feature versioning | Safe updates | Schema migration complexity |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Training/serving skew | Define features once; same code for batch and stream |
| Missing TTL on online store | Features served after entity deleted; always set TTL |
| No point-in-time correctness | Silent data leakage; use ASOF join in training |
| Feature staleness alert missing | Add `feature_updated_at` metadata; alert if stale |
| Registering raw columns as features | Features should be engineered; don't just forward DB columns |

---

## Used In
- [Recommendation Engine](../../cases/recommendation-engine/) — user preferences, viewing history aggregations
- [Fraud Detection](../../cases/fraud-detection/) — velocity counters, device history, merchant risk scores
- [Search Ranking](../../cases/search-ranking/) — user query affinity, click history, category preferences

---

## Tools & Implementations
| Tool | Notes |
|---|---|
| Feast | Open-source; supports Redis online, S3/BigQuery offline |
| Tecton | Managed; strong streaming support |
| Hopsworks | Open-source; includes model registry |
| Vertex AI Feature Store | GCP-native |
| SageMaker Feature Store | AWS-native |
| Custom (Redis + Spark) | Common at scale; full control |
