# Pattern: Model Serving

Infrastructure for deploying ML models to production and serving predictions — reliably, at low latency, with safe rollout.

---

## What It Is

Model serving bridges the gap between a trained model artifact and a production prediction API. It handles:
- Loading and versioning model artifacts
- Pre/post processing of inputs and outputs
- Safe deployment (shadow mode, canary, rollback)
- Batching requests for throughput optimization
- Monitoring prediction quality in production

---

## When to Use

| Use | Avoid |
|---|---|
| Model needs real-time or near-real-time predictions | Batch-only scoring (use batch inference instead) |
| Multiple models serving different surfaces | Single model, infrequent predictions |
| Need canary/shadow rollout for model updates | Prototype with direct library calls |
| Prediction latency SLA < 500ms | |
| Multiple teams consuming same model predictions | |

---

## Core Concepts

### Serving Modes

| Mode | Latency | Throughput | Use case |
|---|---|---|---|
| **Online (synchronous)** | < 100ms | Medium | User-facing: fraud score, recommendation |
| **Near-real-time (async)** | < 1s | High | Feed ranking, content moderation |
| **Batch (offline)** | Hours | Very high | Pre-compute scores for all users nightly |
| **Streaming** | < 1s | High | Event-driven scoring (Kafka consumer) |

### Shadow Mode vs Canary

```
Shadow Mode:
  100% traffic → Current Model (production decisions)
  100% traffic → New Model    (predictions logged, NOT used)
  Compare: new model predictions vs ground truth, no user impact

Canary:
  95% traffic → Current Model
   5% traffic → New Model    (real decisions)
  Gradual ramp: 5% → 25% → 50% → 100% on metric pass
```

Shadow mode = zero risk comparison. Canary = real-world validation with controlled blast radius.

---

## Architecture

### Online Serving

```
Client Request
    ↓
Model Serving API (FastAPI / TorchServe / Triton)
    ├── Input Validation & Pre-processing
    ├── Feature Fetch (Feature Store)
    ├── Model Inference (in-process or gRPC to inference server)
    ├── Post-processing (threshold, calibration, formatting)
    └── Response
         ↓ async
    Prediction Logger → Kafka → Monitoring
```

### Batch Inference

```
Schedule (daily/hourly)
    ↓
Batch Job (Spark / Ray)
    ├── Load entity list (all users / all items)
    ├── Feature batch read (Offline Feature Store)
    ├── Model.predict(feature_matrix)   ← vectorized, no HTTP overhead
    └── Write predictions → DB / Cache
         ↓
Online serving reads pre-computed scores (no model at request time)
```

### Model Registry Flow

```
Training → Evaluate → Register (Model Registry)
                            ↓
                     Staging Environment (integration test)
                            ↓
                     Shadow Mode (compare vs production)
                            ↓
                     Canary (5% → 25% → 50% → 100%)
                            ↓
                     Production
                            ↓ on regression
                     Rollback (previous version, < 60s)
```

---

## Implementation Details

### Inference Server Options

| Tool | Best for |
|---|---|
| **Triton Inference Server** | GPU models, multiple framework support (TensorFlow, PyTorch, ONNX), batching |
| **TorchServe** | PyTorch models, custom handlers |
| **BentoML** | Python-first, packaging + serving together |
| **Ray Serve** | Python models, async, scales horizontally |
| **TensorFlow Serving** | TensorFlow models, gRPC, production-grade |
| **FastAPI + joblib** | Sklearn/GBM models, simple, low overhead |

### Dynamic Batching
```
Incoming requests (100 RPS):
  Request 1 → wait 5ms
  Request 2 → wait 4ms
  Request 3 → wait 3ms   → batch together → single model.predict([r1, r2, r3, ...])
  ...

Benefit: GPU utilization ↑ (GPU is idle on single-request inference)
Cost:    Added latency (batch wait time, ~5ms)
Config:  max_batch_size=64, max_delay=5ms
```

### Model Versioning Schema
```
Model Registry entry:
  model_id:       "fraud_gbt_v42"
  framework:      "xgboost"
  artifact_uri:   "s3://ml-models/fraud/v42/model.bin"
  feature_schema: "s3://ml-models/fraud/v42/features.json"  ← critical: exact feature list + order
  metrics:        { auc_pr: 0.87, recall_at_p95: 0.81 }
  training_data:  "s3://training-data/fraud/2026-07-01/"
  status:         ENUM (staging, shadow, canary, production, retired)
  created_at:     TIMESTAMP
  deployed_at:    TIMESTAMP
```

`feature_schema` must be pinned per model version — prevents silent feature mismatch between training and serving.

### Prediction Logging
```
Every prediction logged:
  {
    request_id:     UUID,
    model_version:  "v42",
    entity_id:      user_id / transaction_id,
    input_hash:     SHA256(feature_vector),   ← not raw features (PII)
    prediction:     0.87,
    latency_ms:     12,
    timestamp:      ...
  }
```

Logged to Kafka → consumed by:
1. Monitoring service (distribution shift detection)
2. Label joiner (when ground truth arrives, join with prediction for model eval)

### Rollback Trigger
```
Monitor metrics every 5 minutes during canary:
  if (error_rate > 1%) OR (p99_latency > SLA) OR (prediction_distribution_shift > threshold):
    auto_rollback(previous_version)
    alert(on_call)
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Shadow before canary | Zero-risk comparison | Takes longer before new model goes live |
| Dynamic batching | GPU throughput | Added batch wait latency (~5ms) |
| Pre-compute (batch serving) | Zero serving latency | Stale predictions; storage cost |
| Online serving | Fresh predictions | Infrastructure cost; latency SLA burden |
| Separate inference server (Triton) | GPU optimization, multi-framework | Operational complexity vs in-process inference |
| Feature schema pinned per model | No silent skew | Schema migration effort on feature changes |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Training/serving skew in pre-processing | Same pre-processing code for training and serving (shared library) |
| No shadow mode before canary | Shadow first; compare prediction distributions before real traffic |
| Model loaded fresh per request | Load model once at startup; keep in memory |
| No prediction logging | Can't detect drift or debug production failures |
| Feature order mismatch | Pin feature schema (name + order) in model artifact |
| Rollback takes > 5min | Pre-load previous model version; rollback = pointer swap |

---

## Used In
- [Recommendation Engine](../../cases/recommendation-engine/) — online ranker serving (LightGBM), batch pre-compute of candidate sets
- [Fraud Detection](../../cases/fraud-detection/) — online GBM scoring, < 100ms SLA, canary rollout
- [Search Ranking](../../cases/search-ranking/) — L2 LTR model serving per query, shadow testing new ranking models

---

## Tools & Implementations
| Layer | Options |
|---|---|
| Inference server | Triton, TorchServe, BentoML, Ray Serve, FastAPI |
| Model registry | MLflow, Weights & Biases, SageMaker Model Registry, Vertex AI |
| Orchestration | Kubernetes + Helm, SageMaker Endpoints, Vertex AI Endpoints |
| Monitoring | Evidently AI, Arize, Fiddler, WhyLabs, custom on Grafana |
| Feature serving | Feast, Tecton, Redis (custom) |
