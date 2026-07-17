# Pattern: Training Pipeline

An orchestrated, reproducible pipeline that takes raw data → produces a validated, versioned model artifact ready for serving.

---

## What It Is

Training pipeline automates the full ML lifecycle from data ingestion to model registration. It ensures:
- **Reproducibility:** same data + same code = same model
- **Validation:** data quality and model quality gates before promotion
- **Versioning:** every artifact (data, features, model, metrics) tracked
- **Orchestration:** steps run in correct order, failures surface cleanly
- **Scheduling:** retrain on schedule or triggered by data/drift events

---

## When to Use

| Use | Avoid |
|---|---|
| Model needs periodic retraining (weekly, daily) | One-off research models |
| Multiple engineers/teams contribute to training | Solo experimentation |
| Regulatory requirement for audit trail | Prototype before production |
| Model performance degrades over time (drift) | Static datasets, no new data |
| CI/CD parity with model deployments | |

---

## Core Concepts

### Pipeline Stages (in order)

```
1. Data Ingestion        → pull training data from source
2. Data Validation       → schema check, distribution check, null check
3. Feature Engineering   → compute features from raw data
4. Dataset Split         → train / validation / test (time-aware for temporal data)
5. Model Training        → fit model on train set
6. Model Evaluation      → metrics on validation + test set
7. Model Validation Gate → pass/fail thresholds; compare vs baseline
8. Model Registration    → push to model registry if passed
9. (Optional) Trigger serving pipeline
```

Any stage fails → pipeline stops; alert fired; no bad model reaches registry.

### Experiment Tracking vs Pipeline

| Concept | Purpose |
|---|---|
| Experiment tracking (MLflow runs) | Log hyperparams, metrics, artifacts per training run |
| Training pipeline (Airflow / Kubeflow) | Orchestrate stages; enforce gates; schedule runs |
| Model registry | Store validated artifacts; manage versions + status |

All three work together. Pipeline orchestrates; tracking records; registry promotes.

---

## Architecture

```
Trigger
  ├── Schedule (cron: daily 2AM)
  ├── Data trigger (new data volume threshold)
  └── Drift trigger (monitoring detects feature/label drift)
        ↓
Pipeline Orchestrator (Airflow / Kubeflow / Prefect)
  │
  ├── [Stage 1] Data Ingestion
  │     └── Pull from: S3 / BigQuery / Feature Store
  │     └── Output: raw_dataset.parquet (versioned, S3)
  │
  ├── [Stage 2] Data Validation
  │     └── Great Expectations / custom checks
  │     └── Gate: fail if > 1% nulls, schema mismatch, distribution anomaly
  │
  ├── [Stage 3] Feature Engineering
  │     └── Spark / pandas transforms
  │     └── Output: feature_matrix.parquet (versioned)
  │
  ├── [Stage 4] Dataset Split
  │     └── Time-based split (no random shuffle — prevents leakage on temporal data)
  │     └── Output: train / val / test parquet files
  │
  ├── [Stage 5] Model Training
  │     └── Train on train set; hyperparams from config / HPO result
  │     └── Log: all params, metrics, artifacts to MLflow
  │     └── Output: model.bin + feature_schema.json
  │
  ├── [Stage 6] Model Evaluation
  │     └── Evaluate on held-out test set
  │     └── Compute: primary metric + fairness metrics + slice metrics
  │
  ├── [Stage 7] Validation Gate
  │     └── Pass if: metric > threshold AND metric > baseline_model − delta
  │     └── Fail: stop pipeline, alert, do NOT promote
  │
  └── [Stage 8] Model Registration
        └── Push to Model Registry with status = "staging"
        └── Trigger: serving pipeline canary deployment
```

---

## Implementation Details

### Data Validation Checks
```python
# Great Expectations style
suite = {
    "schema": [
        expect_column_to_exist("user_id"),
        expect_column_values_to_be_of_type("amount", "float"),
    ],
    "completeness": [
        expect_column_values_to_not_be_null("user_id", mostly=0.99),
        expect_column_values_to_not_be_null("label", mostly=1.0),
    ],
    "distribution": [
        expect_column_mean_to_be_between("amount", 50, 200),
        expect_column_values_to_be_between("fraud_rate", 0.0005, 0.005),
    ],
}
# Failure → pipeline stops; alert data engineering team
```

### Time-Based Dataset Split
```python
# For temporal data: NEVER random shuffle
data = data.sort_values("event_time")

train_cutoff = "2026-04-01"
val_cutoff   = "2026-06-01"

train = data[data.event_time < train_cutoff]
val   = data[(data.event_time >= train_cutoff) & (data.event_time < val_cutoff)]
test  = data[data.event_time >= val_cutoff]

# Prevents data leakage: future events don't inform past predictions
```

### Model Validation Gate
```python
baseline_metric = registry.get_production_model().metrics["auc_pr"]
new_metric      = evaluate(new_model, test_set)["auc_pr"]

ABSOLUTE_THRESHOLD = 0.75     # must beat regardless
REGRESSION_DELTA   = 0.02     # can't be more than 2% worse than baseline

if new_metric < ABSOLUTE_THRESHOLD:
    raise PipelineGateFailure(f"AUC-PR {new_metric} below threshold {ABSOLUTE_THRESHOLD}")

if new_metric < baseline_metric - REGRESSION_DELTA:
    raise PipelineGateFailure(f"Regression: {new_metric} vs baseline {baseline_metric}")

# Also check: slice metrics (fairness), latency on validation set
```

### Artifact Versioning
```
S3 artifact layout:
  s3://ml-artifacts/
    fraud-model/
      run_20260717_020000/
        raw_data.parquet       ← input data snapshot
        features.parquet       ← engineered feature matrix
        train_config.json      ← all hyperparameters
        model.bin              ← trained model
        feature_schema.json    ← feature names + types + order
        metrics.json           ← all evaluation metrics
        data_validation.json   ← validation report
```

Every run fully reproducible: same data snapshot + same config = same output.

### Experiment Tracking (MLflow)
```python
with mlflow.start_run(run_name="fraud_v43"):
    mlflow.log_params({
        "n_estimators": 200,
        "max_depth": 6,
        "learning_rate": 0.05,
        "train_cutoff": "2026-04-01",
        "data_version": "s3://ml-artifacts/fraud/run_.../raw_data.parquet",
    })

    model.fit(X_train, y_train)

    mlflow.log_metrics({
        "auc_pr": 0.87,
        "recall_at_p95": 0.81,
        "train_time_sec": 420,
    })

    mlflow.xgboost.log_model(model, "model")
```

### Retraining Triggers

| Trigger | When | Why |
|---|---|---|
| Schedule | Weekly (cron) | Regular refresh; catch slow drift |
| Data volume | New data > threshold | Enough signal for meaningful update |
| Feature drift | KL divergence > threshold | Feature distribution shifted (new fraud pattern, seasonal) |
| Label drift | Fraud rate spike | Ground truth distribution changed |
| Manual | On-demand by ML engineer | Emergency retrain after incident |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Time-based split | No temporal leakage | Less training data than random split |
| Validation gate (hard threshold) | No bad models reach prod | Occasional good model blocked if threshold too tight |
| Full artifact versioning (data + model) | Full reproducibility | Storage cost (mitigated by lifecycle policies) |
| Scheduled retraining | Predictable, simple | Slow to react to sudden drift (mitigated by drift trigger) |
| MLflow for experiment tracking | Standard, open-source | UI not as polished as Weights & Biases |
| Orchestration (Airflow) | Mature, widely supported | Steep learning curve; heavy infra |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Random shuffle on temporal data | Always time-based split; never shuffle before split |
| No data validation step | Silent bad data → silent bad model |
| Validation gate too loose | Model regresses; no alert |
| Not logging training data version | Can't reproduce a run from 6 months ago |
| Feature engineering code diverges between training and serving | Shared library; Feature Store definition as source of truth |
| HPO results not tracked | Can't explain why one run beat another |
| Retraining always on full history | Old data can hurt on recent patterns; use sliding window |

---

## Used In
- [Recommendation Engine](../../cases/recommendation-engine/) — weekly two-tower retraining on interaction logs; daily feature refresh
- [Fraud Detection](../../cases/fraud-detection/) — weekly GBM retraining; drift-triggered fast retrain; label delay handled with proxy labels
- [Search Ranking](../../cases/search-ranking/) — weekly LTR retraining on click logs; IPS position bias correction in training

---

## Tools & Implementations
| Layer | Options |
|---|---|
| Orchestration | Apache Airflow, Kubeflow Pipelines, Prefect, Metaflow, Vertex AI Pipelines |
| Data validation | Great Expectations, Deequ (Spark), TFX Data Validation |
| Experiment tracking | MLflow, Weights & Biases, Neptune, Comet |
| Model registry | MLflow Model Registry, SageMaker Model Registry, Vertex AI Model Registry |
| Feature engineering | Spark, dbt, Feast transform |
| HPO | Optuna, Ray Tune, SageMaker Automatic Model Tuning |
| Compute | Spark on EMR/Dataproc, Ray on Kubernetes, SageMaker Training Jobs |
