# ML Systems

Machine learning infrastructure and pipeline design.

**Core concerns:** Data freshness, Training/serving skew, Latency vs accuracy, Model drift, Reproducibility

## Cases

| System | Key Challenge |
|---|---|
| [Recommendation Engine](./cases/recommendation-engine/) | Real-time vs batch features, cold start, feedback loop |
| [Fraud Detection](./cases/fraud-detection/) | Low latency inference, imbalanced data, concept drift |
| [Search Ranking](./cases/search-ranking/) | Feature freshness, A/B testing, multi-stage ranking |

## Patterns

| Pattern | Used In |
|---|---|
| [Feature Store](./patterns/feature-store/) | Shared features across models, point-in-time correctness |
| [Model Serving](./patterns/model-serving/) | Online inference, shadow mode, canary deployments |
| [Training Pipeline](./patterns/training-pipeline/) | Orchestration, data validation, experiment tracking |
