# Grokking System Designs

Real-world system design cases and patterns across three domains.

> Theory notes live in Obsidian. This repo = examples, cases, tradeoffs.

---

## Domains

### [Classical Systems](./classical/)
Traditional distributed systems — scalability, availability, consistency.

| Cases | Patterns |
|---|---|
| [URL Shortener](./classical/cases/url-shortener/) | [Rate Limiting](./classical/patterns/rate-limiting/) |
| [Twitter Feed](./classical/cases/twitter-feed/) | [Caching](./classical/patterns/caching/) |
| [Uber](./classical/cases/uber/) | [Consistent Hashing](./classical/patterns/consistent-hashing/) |

---

### [ML Systems](./ml-systems/)
ML infra and pipelines — data freshness, training/serving skew, latency vs accuracy.

| Cases | Patterns |
|---|---|
| [Recommendation Engine](./ml-systems/cases/recommendation-engine/) | [Feature Store](./ml-systems/patterns/feature-store/) |
| [Fraud Detection](./ml-systems/cases/fraud-detection/) | [Model Serving](./ml-systems/patterns/model-serving/) |
| [Search Ranking](./ml-systems/cases/search-ranking/) | [Training Pipeline](./ml-systems/patterns/training-pipeline/) |

---

### [GenAI Systems](./genai-systems/)
LLM-native architectures — context windows, cost, hallucination, eval.

| Cases | Patterns |
|---|---|
| [RAG Pipeline](./genai-systems/cases/rag-pipeline/) | [Prompt Caching](./genai-systems/patterns/prompt-caching/) |
| [AI Agent](./genai-systems/cases/ai-agent/) | [Vector Search](./genai-systems/patterns/vector-search/) |
| [LLM Gateway](./genai-systems/cases/llm-gateway/) | [Evaluation](./genai-systems/patterns/evaluation/) |

---

## How to Use This Repo

- New to system design? Read [HOW_TO_SOLVE.md](./HOW_TO_SOLVE.md), then the guide for your domain:
  [classical](./classical/HOW_TO_SOLVE.md) · [ML](./ml-systems/HOW_TO_SOLVE.md) · [GenAI](./genai-systems/HOW_TO_SOLVE.md)
- Writing a case: copy [TEMPLATE.md](./TEMPLATE.md).
- Writing a pattern: copy [PATTERN_TEMPLATE.md](./PATTERN_TEMPLATE.md).
