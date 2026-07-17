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

## Case Template

Each case follows this structure:
- **Problem** — what to design, scale requirements
- **Requirements** — functional + non-functional
- **Design** — components, data flow, key decisions
- **Tradeoffs** — what was sacrificed, alternatives considered
- **Patterns used** — links back to `/patterns/`
- **Interview tips** — what interviewers look for
