# Fraud Detection System

Design a real-time fraud detection system for financial transactions (payments, card transactions, account takeover).

---

## Requirements

### Functional Requirements
- Score every transaction in real time before it is approved or declined
- Flag suspicious transactions for manual review queue
- Block high-confidence fraud immediately (< 100ms decision)
- Support rule-based overrides on top of ML scores
- Provide explainability for declined transactions (regulatory requirement)
- Handle account takeover (ATO) detection, not just transaction fraud

### Non-Functional Requirements
- Latency: scoring decision < 100ms p99 (payment rails require fast response)
- Availability: 99.999% — false unavailability = missed fraud or blocked legitimate transactions
- Consistency: strong for blocking decisions; eventual for analytics/reporting
- Security & Privacy: transaction data highly sensitive; PCI-DSS compliance required
- Scalability: 10,000 TPS (transactions per second) peak

### Out of Scope
- Manual review investigator tooling (integrated but not designed)
- Chargeback dispute resolution
- KYC (Know Your Customer) onboarding

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| Transactions/day | 500M |
| Peak TPS | 10,000 |
| Fraud rate | ~0.1% (~500K fraudulent/day) |
| Manual review volume | ~50K/day (10% of flagged) |

### Storage Estimation
| Metric | Value |
|---|---|
| Transaction record size | ~2 KB |
| Transactions/day | 500M → ~1 TB/day |
| Retention (2yr) | ~700 TB |
| Feature vectors per transaction | ~500 bytes |

### Memory Estimation
| Metric | Value |
|---|---|
| User velocity features (Redis) | ~100 GB (100M users × 1 KB) |
| Merchant risk scores cache | ~5 GB |
| Active model in memory | ~2 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Ingress (transaction events) | ~20 MB/s |
| Egress (scoring decisions) | ~5 MB/s |

---

## High-Level Design

### Architecture Style
**Event-driven + Synchronous Decision Path** — transaction arrives as event; synchronous scoring pipeline must complete before payment gateway timeout (100ms); async pipelines handle feature updates and model retraining separately.

### Architecture Diagram

**Real-time Scoring Path (synchronous, < 100ms):**
```
Payment Gateway
    ↓
Transaction Event
    ↓
Fraud Detection Service
    ├── Feature Extractor (Redis feature store)
    │    ├── User velocity features (last 1hr, 24hr, 7d counts/amounts)
    │    ├── Merchant risk score
    │    ├── Device fingerprint
    │    └── Network features (IP, geolocation)
    ├── Rule Engine (fast pre-filter)
    │    └── hard rules: block if impossible travel, known bad IPs, etc.
    ├── ML Scorer (gradient boosted model)
    │    └── fraud_score ∈ [0, 1]
    ├── Decision Engine
    │    ├── score > 0.9 → BLOCK
    │    ├── score 0.5–0.9 → REVIEW (step-up auth)
    │    └── score < 0.5 → APPROVE
    └── Decision → Payment Gateway (approve/decline/step-up)
         ↓ async
    Event Bus (Kafka)
    ├── Feature Updater (update velocity counters in Redis)
    ├── Training Data Sink (S3)
    └── Alert Service (high-fraud-score notifications)
```

**Offline Pipeline (batch):**
```
Labeled Transactions (S3)
    ↓
Feature Engineering (Spark)
    ↓
Model Training (XGBoost / LightGBM)
    ↓
Evaluation (precision, recall, AUC-PR on imbalanced set)
    ↓
Model Registry → Shadow Mode → Canary → Full Rollout
```

### Core Components
| Component | Responsibility |
|---|---|
| Feature Store (Redis) | Sub-millisecond feature reads; velocity counters; merchant scores |
| Rule Engine | Hard rules for known fraud patterns; zero ML latency; human-interpretable |
| ML Scorer | Gradient boosted model; outputs fraud probability |
| Decision Engine | Thresholds + business policy → approve/decline/step-up |
| Feature Updater | Async: update velocity counters after each transaction |
| Training Pipeline | Weekly/daily batch retraining on labeled data |
| Review Queue | Flagged transactions → human investigators |

### Data Flow

**Synchronous path (must complete < 100ms):**
1. Payment gateway sends transaction to Fraud Detection Service
2. Feature Extractor reads from Redis: user velocity (count/amount in 1hr/24hr/7d windows), merchant risk, device hash, IP reputation — all < 5ms
3. Rule Engine evaluates hard rules — fast path: impossible travel, known fraudster, card on blocklist → instant BLOCK
4. ML Scorer: extract feature vector → GBM inference → fraud_score — < 10ms
5. Decision Engine applies thresholds → APPROVE / DECLINE / STEP-UP (3DS / OTP)
6. Decision returned to payment gateway

**Async path (post-decision):**
7. Transaction + decision published to Kafka
8. Feature Updater: increment velocity counters in Redis (sliding window using sorted sets)
9. Training data sink: write to S3 with label (ground truth arrives via chargeback feedback, days later)
10. Alert service: notify risk team on high-score clusters

### Key Design Decisions
- **Rule engine before ML:** Hard rules catch known patterns instantly and are explainable (regulatory requirement). ML handles novel patterns rules miss. Layer order: rules first (cheap, fast, explainable) → ML (expensive, probabilistic, handles unknown patterns).
- **Redis for velocity features:** Velocity counters (# transactions in last hour, total amount in 24hr) are the strongest fraud signals. Must be sub-millisecond. Redis sorted sets enable sliding window counts efficiently.
- **GBM over neural network:** Gradient boosted trees (XGBoost/LightGBM) offer: fast inference (< 10ms), good performance on tabular data, built-in feature importance (explainability). Neural nets offer marginal lift but 10× slower inference and black-box.
- **Three-tier decision (approve/review/block):** Binary approve/decline loses value in the middle. Step-up auth (OTP/3DS) lets legitimate users verify, reducing false positives.
- **Async feature updates:** Writing velocity counters synchronously in the critical path adds latency. Publish to Kafka post-decision; Flink consumer updates Redis < 1s later — acceptable lag.

---

## Low-Level Design

### Data Models
```
transactions (Cassandra):
  transaction_id  UUID      PK
  user_id         UUID
  merchant_id     UUID
  amount          DECIMAL
  currency        VARCHAR
  timestamp       TIMESTAMP
  device_id       VARCHAR
  ip_address      VARCHAR   (hashed for PCI)
  fraud_score     FLOAT
  decision        ENUM (approve, decline, review)
  label           ENUM (legitimate, fraud, unknown)  ← ground truth, arrives later
  model_version   VARCHAR

velocity_counters (Redis — sorted sets):
  key: "vel:{user_id}:1h"   → sorted set of (timestamp, amount)
  key: "vel:{user_id}:24h"  → sorted set of (timestamp, amount)
  key: "vel:{user_id}:7d"   → sorted set of (timestamp, amount)
  key: "vel:{merchant_id}:fraud_rate_7d" → float

device_fingerprints (Redis):
  key: "device:{device_hash}" → { user_ids: set, first_seen, last_seen, fraud_count }

blocklist (Redis):
  key: "block:card:{card_hash}" → reason
  key: "block:ip:{ip_hash}"    → reason
  key: "block:user:{user_id}"  → { reason, expires_at }

model_artifacts (S3 + local cache):
  /models/fraud/{version}/model.bin
  /models/fraud/{version}/feature_schema.json
  /models/fraud/{version}/thresholds.json
```

### API Design
```
POST /score
Request: {
  "transaction_id": "...",
  "user_id": "...",
  "merchant_id": "...",
  "amount": 149.99,
  "currency": "USD",
  "device_id": "...",
  "ip_address": "...",
  "card_last4": "1234",
  "timestamp": "2026-07-17T10:00:00Z"
}
Response: {
  "decision": "approve",          # approve | decline | step_up
  "fraud_score": 0.12,
  "rule_triggered": null,         # or "impossible_travel"
  "reason_codes": ["low_risk_merchant", "normal_velocity"],
  "latency_ms": 23
}

POST /feedback
Request: { "transaction_id": "...", "label": "fraud", "source": "chargeback" }
Response: 204

GET /rules
Response: { "rules": [...] }

PUT /rules/{rule_id}
Request: { "enabled": false }
Response: 204
```

### Component Deep-Dives

#### Velocity Feature Extraction (Redis Sorted Sets)
```
# Count transactions in last 1 hour for user
ZREMRANGEBYSCORE vel:{user_id}:1h -inf (now - 3600)
ZADD vel:{user_id}:1h now:{txn_id} now amount
ZCARD vel:{user_id}:1h   → count
ZRANGEBYSCORE ... -inf +inf WITHSCORES → sum amounts

# All via Redis pipeline → single round trip ~1ms
```

Sliding window (not fixed bucket) avoids boundary effects at window edges.

#### Rule Engine
Rules evaluated in priority order, short-circuit on first match:

| Priority | Rule | Action |
|---|---|---|
| 1 | Card on blocklist | BLOCK |
| 2 | User on blocklist | BLOCK |
| 3 | Known fraudulent IP | BLOCK |
| 4 | Impossible travel (< 30min between cities > 500km) | BLOCK + FLAG |
| 5 | Amount > 5× user 90d avg in single transaction | REVIEW |
| 6 | New device + high amount (> $500) | STEP_UP |
| 7 | Merchant fraud rate > 5% (7d) | REVIEW |

Rules stored as config (JSON DSL), hot-reloadable via config service.

#### ML Model — Feature Set
| Feature | Engineering |
|---|---|
| txn_amount | raw |
| amount_vs_user_avg_30d | amount / user_avg_30d |
| user_txn_count_1h | velocity counter |
| user_amount_sum_24h | velocity counter |
| merchant_fraud_rate_7d | merchant risk cache |
| time_since_last_txn | now - last_txn_ts |
| is_new_merchant | bool (first transaction at merchant) |
| is_new_device | bool |
| ip_country_matches_card_country | bool |
| hour_of_day | int (0–23) |
| day_of_week | int (0–6) |
| distance_from_last_txn_km | geolocation delta |

#### Guardrails / Safety Layer
- **False positive cap:** business rule — if model decline rate exceeds 2% of volume, alert risk team (likely model drift or data issue)
- **Explainability:** SHAP values computed for declined transactions → `reason_codes` in API response (regulatory requirement in EU/US)
- **PCI compliance:** card numbers never stored raw; hashed immediately at ingestion; IP addresses hashed before storage
- **Canary safety:** new model deployed to 1% traffic first; if recall drops > 5% vs baseline → auto-rollback

### Algorithms & Strategies
- **Model:** XGBoost (100 trees, max_depth=6). Trained on 90d of labeled transactions. Labels: chargeback = fraud, no chargeback within 90d = legitimate (noisy label — some fraud not disputed).
- **Class imbalance:** 0.1% fraud rate. Strategies: SMOTE oversampling on minority class during training; `scale_pos_weight=999` in XGBoost; optimize for AUC-PR not AUC-ROC (better for imbalanced).
- **Threshold tuning:** Operating point set to maximize F-beta (beta=0.5, precision-weighted) — cost of false positive (blocking legitimate) ~= 5× cost of false negative for low-value transactions. Separate thresholds per transaction amount tier.
- **Concept drift detection:** Monitor feature distribution (KL divergence) weekly. If drift detected → trigger retraining. Model retrained weekly; fast retrain (2hr on Spark) triggered on drift.
- **Account Takeover (ATO):** Separate model trained on login events: failed login rate, impossible travel on login, new device after password reset. Same scoring infrastructure, separate feature set.

### Security Design
- AuthN/AuthZ: internal service-to-service mTLS; payment gateway pre-authorized IP allowlist
- PCI-DSS: no PANs (Primary Account Numbers) stored; tokenized card reference only; quarterly pen test
- Encryption at rest: all transaction data AES-256; key rotation annually
- Encryption in transit: TLS 1.3; no plaintext on internal network
- Audit log: every decision immutably logged with model_version, feature_values_hash, rule_triggered
- Data access: analyst access to data warehouse only (not raw Redis/Cassandra); row-level security by region

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Scoring latency (p99) | Histogram | > 100ms |
| Feature Store latency | Histogram | > 5ms |
| Decline rate | Gauge | > 2% (possible model issue) |
| False positive rate (estimated) | Gauge | > 0.5% |
| Fraud caught rate (recall) | Gauge | < 85% (on labeled set) |
| Model staleness | Gauge | > 8 days since retrain |
| Feature drift (KL divergence) | Gauge | > 0.1 on key features |

### Logging
- Per transaction: transaction_id_hash, user_id_hash, amount_bucket, decision, fraud_score, rule_triggered, model_version, latency
- Per feature: feature vector hash (not raw values — PCI compliance)
- Per label update: transaction_id, label, source (chargeback/investigator), delay_days

### Tracing
- Trace spans: feature_fetch → rule_eval → ml_score → decision → kafka_publish
- Critical: feature fetch latency broken down per feature group (velocity, merchant, device)
- Sample: 100% for DECLINE and STEP_UP decisions; 0.1% for APPROVE

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Scoring latency spike | p99 > 100ms | Immediate page — payment blocking risk |
| Decline rate spike | > 3× baseline | Page risk team — possible model issue |
| Feature Store down | Read error rate > 0.1% | Failover to cached features, page on-call |
| Fraud cluster detected | 5+ high-score txns same merchant in 1min | Alert fraud ops team |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| GBM over deep learning | Fast inference, explainability | Marginal accuracy on complex patterns |
| Async feature updates | Low scoring latency | 1-2s lag on velocity counters (tiny fraud window) |
| Three-tier decision | Reduces false positives via step-up | UX friction for legitimate users hitting STEP_UP |
| Rules + ML layered | Explainability + novel pattern coverage | Rule engine needs maintenance as fraud evolves |
| Sliding window velocity | Accurate counts, no boundary effects | Redis sorted set storage (manageable) |
| Weekly retraining | Fresh model, catches drift | Daily fine-tune possible but high infra cost |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Feature Store down | Redis failure | Failover to degraded mode: score with model using defaults for missing features; escalate thresholds |
| Model serving failure | OOM or corrupt artifact | Auto-rollback to previous model version; rule-only fallback |
| Velocity counter lag | Kafka consumer backlog | Alert; temporary rule: flag users with >10 txns/hr manually |
| Label noise | Friendly fraud (dispute not fraud) | Use chargeback reason codes to filter; separate model for friendly fraud |
| Concept drift | New fraud pattern emerges | Drift detector triggers fast retrain; manual rule added while model updates |
| High false positive burst | Model degradation or bad feature | Kill switch: raise decision threshold to 0.95; alert risk team |

---

## Feedback Loop & Improvements
- Ground truth labels arrive via chargebacks (days to weeks later) — standard lag in fraud
- Investigator feedback: human review decisions label borderline cases → high-quality training signal
- Champion/challenger: new model shadows production, compares recall/precision before promotion
- Online learning: for fast-moving fraud patterns, incremental gradient updates on recent data (Vowpal Wabbit style) — bridges gap between weekly batch retrains
- Eval cadence: AUC-PR, recall@precision=0.95 reported weekly; compared to previous 4 weeks rolling

---

## Interview Tips

### What Interviewers Expect
- Class imbalance is the first ML question — have SMOTE + scale_pos_weight + AUC-PR answer ready
- Latency constraint is the system design crux — explain why Redis, why async updates, why GBM not neural net
- Explainability for regulatory compliance — SHAP values, reason codes
- Concept drift — fraud patterns change weekly; model needs frequent retraining

### Common Follow-ups
- How to handle label delay? Chargebacks take weeks — use proxy labels (investigator decisions) + delayed retraining loop
- Account takeover vs transaction fraud? Separate models, same infrastructure; ATO uses login event features
- How to prevent adversarial fraud? Feature obfuscation (don't reveal exact thresholds); rule diversity; ensemble models
- Graph-based fraud detection? Build transaction graph (users → merchants → devices) → GNN detects fraud rings — mention as extension
