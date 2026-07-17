# How to Solve ML System Design Problems

ML pipelines, feature engineering, model training, serving, monitoring. See [universal guide](../HOW_TO_SOLVE.md) first.

---

## What "ML System Design" Means

Not "design an ML algorithm." Design the **infrastructure** around an ML model:
- How data flows from raw events → features → training → model → predictions
- How predictions reach users at low latency
- How the system stays accurate over time

You are designing a system that has a model inside it — not just the model.

---

## The Core Mental Model: Two Pipelines

Every ML system has two pipelines. Always draw both.

```
OFFLINE PIPELINE (batch, runs on schedule):
  Raw Data → Feature Engineering → Training → Evaluation → Model Registry

ONLINE PIPELINE (real-time, runs per request):
  Request → Feature Fetch → Model Inference → Business Rules → Response
                ↑
          Feature Store (pre-computed features from offline pipeline)
```

Offline = quality (freshness of model). Online = latency (speed of serving).
The Feature Store is the bridge between them.

---

## Step 1 — Clarifying Questions (ML-Specific)

Beyond universal questions, ask these:

**Problem framing:**
- What is the prediction task? (classification, ranking, regression, clustering)
- What is the label / ground truth? (how do you know if the model was right?)
- How fast does the label arrive? (real-time click, or chargeback after 30 days?)
- Is this online (per-request) or batch (pre-computed) inference?

**Data:**
- What data is available? (logs, user history, item metadata, signals)
- How much historical data? (affects model complexity and training time)
- How fast does data change? (affects retraining frequency)
- Any class imbalance? (fraud: 0.1% positive rate)

**Scale:**
- How many predictions/sec at peak?
- Latency SLA? (< 100ms → can't call heavy model; < 2s → ML model fine)
- How many entities? (users, items — drives feature store size)

**Accuracy vs latency:**
- What is the cost of a false positive vs false negative? (fraud: FN = missed fraud; FP = blocked legit user)
- Is there a hard latency SLA that constrains model complexity?

---

## Step 2 — Classify the ML Problem

```
What kind of prediction?

Recommend N items from large catalog?
  → Two-stage: candidate generation (ANN) + ranking (LTR/GBM)

Score a single entity (fraud, spam, churn)?
  → Single model: GBM / logistic regression; optimize for latency

Rank a set of retrieved items?
  → Learning to Rank (LambdaMART); pairwise training on click data

Generate content / embeddings?
  → Neural network (transformer); higher latency; GPU serving

Time-series forecasting?
  → LSTM / Prophet / statistical models; batch prediction common

Anomaly detection?
  → Unsupervised (isolation forest, autoencoder); or supervised if labeled
```

---

## Step 3 — Model Selection Decision Tree

```
Tabular data + latency SLA < 100ms?
  └── GBM (XGBoost / LightGBM) ← default choice for tabular
      Fast inference, interpretable, great on structured features

Tabular data + latency flexible + need max accuracy?
  └── Neural network or ensemble

Semantic / text / image input?
  └── Transformer-based (BERT, DistilBERT, CLIP)
      Trade: slower inference → need GPU or distilled model

Recommendation (user-item matching)?
  └── Two-tower neural network for embeddings + GBM for ranking

Need interpretability (regulatory)?
  └── Logistic regression or GBM + SHAP values

Need online learning (fast adaptation)?
  └── Vowpal Wabbit / bandit algorithms / incremental GBM
```

> Rule: GBM for tabular inference under latency pressure. Neural nets for semantic/unstructured input. Two-tower for embedding-based retrieval.

---

## Step 4 — Feature Design (Most Important Part)

Features determine model quality more than model choice. Spend time here.

**Feature categories:**

| Category | Examples | Source |
|---|---|---|
| Entity features | user age, account tenure, country | User DB / Feature Store |
| Behavioral features | txn_count_7d, avg_amount_30d | Aggregated from event logs |
| Contextual features | time_of_day, device, day_of_week | Request context |
| Relational features | merchant_fraud_rate, item_ctr | Aggregated across entities |
| Interaction features | user × item affinity | Computed from co-occurrence |
| Embedding features | user_embedding, item_embedding | Learned by neural model |

**Key principle — avoid training/serving skew:**
```
Training time:  feature computed from historical data (batch)
Serving time:   feature fetched from Feature Store (real-time)

Same feature, different computation path → SKEW → silent model degradation

Fix: define feature once in Feature Store; same code for both paths
```

**Velocity features (common in fraud, recommendation):**
```
txn_count_1h  = count of transactions in last 1 hour
txn_amount_24h = sum of amounts in last 24 hours

Store in Redis sorted sets (sliding window)
Update async after each event
```

---

## Step 5 — HLD Skeleton for ML Systems

```
DATA LAYER:
  Event Stream (Kafka) → Data Lake (S3) → Feature Pipeline (Spark/Flink)
                                                ↓
                                         Feature Store
                                    (offline: S3 / BigQuery)
                                    (online:  Redis)

TRAINING LAYER:
  Feature Store (offline) → Training Dataset → Model Training → Evaluation → Model Registry

SERVING LAYER:
  Request → Feature Fetch (online Feature Store) → Model Inference → Business Rules → Response
                                                         ↑
                                               Model loaded from Registry
                                                         ↓ async
                                               Prediction Logger → Monitoring
```

Draw both pipelines. Name all data stores. Interviewers grade on completeness.

---

## Step 6 — Training Pipeline Checklist

Ask yourself these in order:

```
1. Data split strategy?
   Temporal data → time-based split (NEVER random shuffle → leakage)
   i.i.d. data   → random stratified split

2. Label quality?
   Clean labels? (clicks, purchases)
   Noisy labels? (chargebacks arrive 30 days later)
   Proxy labels? (use until ground truth arrives)

3. Class imbalance?
   Fraud (0.1%) → SMOTE, scale_pos_weight, optimize AUC-PR not AUC-ROC

4. Feature engineering reproducible?
   Same Spark job used for training and feature store population?

5. Validation gate?
   Block model from registry if metric < threshold or regression vs baseline

6. Retraining trigger?
   Schedule (weekly)? Data volume? Drift detection?
```

---

## Step 7 — Serving Architecture Decision

```
Need predictions in < 100ms?
  → Online serving: model loaded in memory, features from Redis
  → GBM inference ~1–10ms; neural net ~50–200ms (GPU) / 500ms+ (CPU)

Latency flexible but high throughput?
  → Batch inference: score all entities nightly, store predictions in DB
  → Zero serving latency (just DB read); acceptable staleness

Mixed (some users real-time, rest pre-computed)?
  → Hybrid: real-time for active users; pre-computed for inactive

GPU needed?
  → Triton Inference Server: dynamic batching, multi-model, GPU utilization
  → Without batching: GPU 90% idle (single requests don't fill GPU)
```

---

## Step 8 — The Hard Problems (Go Deep Here)

### Cold Start
```
New user, no history?
  Option 1: Popularity-based fallback (no personalization)
  Option 2: Content-based (use stated preferences or demographics)
  Option 3: Onboarding flow (collect 3–5 signals; hybrid model after)

New item, no interactions?
  Option 1: Content embedding (metadata → embedding → ANN search)
  Option 2: Exploration slot (force new items into small % of results)
  Option 3: Borrow signals from similar items
```

### Training/Serving Skew
```
Symptom: offline metrics good, online metrics bad

Causes:
  - Feature computed differently at training vs serving time
  - Label leakage (future data used in training features)
  - Distribution shift (training data ≠ production data)

Fix:
  - Feature Store: define features once, serve from same definition
  - Time-based split: no shuffling on temporal data
  - Log features at serving time; use logged features for retraining
```

### Feedback Loop / Filter Bubble
```
Symptom: model trained on its own recommendations → amplifies popular items

Fix:
  - Epsilon-greedy exploration: 5% random items injected into results
  - Coverage metric: alert if < 10% of catalog ever recommended
  - Separate exploration policy from exploitation model
```

### Concept Drift
```
Symptom: model accuracy degrades over time as world changes

Detection:
  - Monitor feature distribution (KL divergence on key features)
  - Monitor prediction distribution (score distribution shifts)
  - Monitor business metric (CTR, fraud catch rate drops)

Response:
  - Scheduled retraining (baseline; catches slow drift)
  - Drift-triggered retraining (faster response to sudden shifts)
  - Online learning (Vowpal Wabbit; fast but complex)
```

### Position Bias (ranking/search)
```
Problem: users click top results → model learns "top = good" not "relevant = good"

Fix: Inverse Propensity Scoring (IPS)
  P(click | position) estimated from randomized ranking experiments
  Training weight = 1 / P(click | position)
  Downweights clicks at position 1 (high exposure) vs position 10 (low exposure)
```

---

## Patterns by Problem Type

| Problem | Pattern | Key component |
|---|---|---|
| Recommendation | Two-stage: ANN retrieval + GBM ranking | Two-tower model, Feature Store |
| Fraud detection | Rule engine + GBM; async velocity features | Redis sorted sets, XGBoost |
| Search ranking | Multi-stage pipeline; LTR | Elasticsearch, LambdaMART |
| Content moderation | Classifier + human-in-loop queue | GBM/BERT, review workflow |
| Churn prediction | Batch scoring; cohort analysis | Feature Store, batch inference |
| Anomaly detection | Unsupervised + threshold tuning | Isolation Forest, monitoring |
| Forecasting | Time-series + feature engineering | Prophet, LSTM, Spark |

---

## Interview Tips (ML Systems)

**Always mention:**
- Both pipelines (offline training + online serving) — never just one
- Training/serving skew — shows production ML experience
- Feature Store — why online + offline stores needed
- Cold start handling — interviewers always ask
- Retraining strategy — not just "retrain weekly" but when and triggered by what

**Interviewers probe:**
- "How do you handle a user with no history?"
- "What if the model starts performing worse after 2 weeks?"
- "How do features get to the model at serving time in under 100ms?"
- "How do you know your offline metrics translate to online performance?"
- "What if your training data has class imbalance?"

**Quick answers:**
- No history → cold start: popularity fallback + content-based hybrid
- Worse over time → concept drift: monitor KL divergence on features; drift-triggered retrain
- Features in 100ms → Redis Feature Store; pre-computed batch features
- Offline vs online gap → shadow mode A/B; log features at serving time for retraining
- Class imbalance → SMOTE + scale_pos_weight; optimize AUC-PR not AUC-ROC
