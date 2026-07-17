# LLM Gateway

Design a centralized gateway that routes LLM requests across multiple providers, handles rate limits, tracks cost, and caches responses.

---

## Requirements

### Functional Requirements
- Unified API — clients use one interface regardless of backend LLM provider
- Route requests to multiple providers (OpenAI, Anthropic, Gemini, Mistral, local models)
- Semantic response caching for repeated/similar queries
- Cost tracking per user / team / project
- Rate limiting per tenant at both request and token level
- Automatic failover when a provider fails or is slow

### Non-Functional Requirements
- Latency: gateway overhead < 20ms p99 on cache miss path
- Availability: 99.99% — higher than any single LLM provider
- Consistency: cost ledger eventually consistent (< 1min lag acceptable)
- Security & Privacy: PII scrubbing before logging; audit trail of all requests
- Scalability: 10M requests/day, 5+ providers

### Out of Scope
- Fine-tuning or hosting LLM models
- Prompt template management (separate system)
- Per-request content moderation (handled upstream)

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| Requests/day | 10M |
| Requests/sec (peak) | ~500 |
| Avg tokens/request | 2,000 input + 500 output |

### Storage Estimation
| Metric | Value |
|---|---|
| Audit log/request | ~2 KB |
| Audit log/day | ~20 GB |
| Cost ledger/day | ~1 GB |

### Memory Estimation
| Metric | Value |
|---|---|
| Semantic cache (vector index) | ~50 GB |
| Rate limit counters (Redis) | ~1 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Token throughput (input) | ~1B tokens/day |
| Token throughput (output) | ~250M tokens/day |

### Cost Estimation
| Metric | Value |
|---|---|
| Tokens/day | ~1.25B |
| Expected cost/day (blended) | ~$5,000–$20,000 |
| Cache hit target (30%) saves | ~$1,500–$6,000/day |

---

## High-Level Design

### Architecture Style
**Stateless Microservice + External State Stores** — gateway nodes are stateless for easy horizontal scaling; all state (cache, rate limits, cost) lives in Redis and Vector DB.

### Architecture Diagram
```
Client SDK
    ↓
Load Balancer
    ↓
LLM Gateway (stateless, horizontally scaled)
    ├── 1. Auth & Tenant Resolution
    ├── 2. Rate Limiter (Redis token bucket)
    ├── 3. Semantic Cache (check)  ──── hit → return cached response
    │         ↓ miss
    ├── 4. Router (provider + model selection)
    ├── 5. Provider Adapter (normalize request)
    │         ↓
    ├── LLM Providers (OpenAI / Anthropic / Gemini / ...)
    │         ↓
    ├── 6. Semantic Cache (write, async)
    ├── 7. Cost Tracker (async → Kafka → ClickHouse)
    └── 8. Audit Logger (async → S3)
```

### Core Components
| Component | Responsibility |
|---|---|
| Auth & Tenant Resolution | Validate API key; resolve tenant_id, plan, quotas |
| Rate Limiter | Token bucket per tenant per model; request + token level |
| Semantic Cache | Embed query, find similar past responses, return on hit |
| Router | Select provider + model based on strategy (cost, latency, capability) |
| Provider Adapter | Normalize request/response across provider APIs |
| Cost Tracker | Async cost calculation and ledger update |
| Audit Logger | Immutable log of all requests for compliance |

### Data Flow

**Cache miss (full path):**
1. Client sends request → Auth resolves tenant + quota
2. Rate limiter checks token bucket (Redis) — reject if exceeded
3. Query embedded → semantic cache search — miss
4. Router selects provider (e.g., cheapest that meets capability)
5. Provider adapter normalizes request → calls LLM provider
6. Response returned to client
7. Async: write to semantic cache, publish cost event to Kafka, write audit log to S3

**Cache hit:**
1-3 same as above — cache hit at step 3
4. Return cached response immediately (no provider call)
5. Async: log cache hit event for analytics

### Key Design Decisions
- **Semantic cache over exact match:** LLM queries are natural language — exact match hit rate near 0%. Embedding similarity (cosine > 0.95 threshold) achieves 30%+ hit rate.
- **Async cost tracking:** Writing cost synchronously adds DB latency on critical path. Kafka + ClickHouse pipeline gives < 1min eventual consistency — acceptable for budget enforcement.
- **Token-level rate limiting:** Request-level alone is insufficient — one request with 100K tokens is very different from one with 500. Enforce both dimensions.
- **Stateless gateway nodes:** All state in Redis/Vector DB — nodes can scale horizontally and fail independently.
- **Model Selection:** Provider adapters abstract provider differences. Router selects based on configured strategy (cost, latency, capability requirements).

---

## Low-Level Design

### Data Models
```
api_keys table (PostgreSQL):
  key_hash      VARCHAR   PK
  tenant_id     UUID
  plan          ENUM (free, pro, enterprise)
  created_at    TIMESTAMP
  revoked_at    TIMESTAMP (nullable)

tenant_quotas table (PostgreSQL):
  tenant_id           UUID    PK
  rpm_limit           INTEGER  (requests per minute)
  tpm_limit           INTEGER  (tokens per minute)
  daily_token_budget  BIGINT

cost_ledger (ClickHouse):
  request_id    UUID
  tenant_id     UUID
  model         VARCHAR
  provider      VARCHAR
  input_tokens  INTEGER
  output_tokens INTEGER
  cost_usd      DECIMAL
  timestamp     DATETIME

semantic_cache (Vector DB):
  id            = cache_id
  vector        = float32[1536]
  metadata      = { response, model, system_prompt_hash, tenant_id, created_at }

rate_limit_counters (Redis):
  key: "rl:{tenant_id}:{model}:rpm"  → token bucket state
  key: "rl:{tenant_id}:tpm"          → tokens-per-minute counter
  key: "budget:{tenant_id}:daily"    → daily token spend counter
```

### API Design
```
POST /v1/chat/completions          ← OpenAI-compatible interface
Request:  {
  "model": "claude-sonnet-4-6",   ← gateway model alias
  "messages": [...],
  "temperature": 0.7,
  "x-gateway-strategy": "cost"    ← optional routing hint
}
Response: { "id": "...", "choices": [...], "usage": {...} }

GET /v1/usage
Response: { "tokens_today": 1234567, "cost_today_usd": 12.34, "budget_remaining_usd": 87.66 }

GET /v1/providers/health
Response: { "openai": "healthy", "anthropic": "degraded", "gemini": "healthy" }
```

### Component Deep-Dives

#### Semantic Cache
```
Incoming query
    ↓
Embed (text-embedding-3-small — cheap, fast)
    ↓
Vector search (cosine similarity)
    ↓
sim > 0.95 AND same (system_prompt_hash, model, tenant_id)?
    ├── YES → return cached response
    └── NO  → call provider → store (vector, response) async
```

Do NOT cache:
- `temperature > 0` (non-deterministic)
- Streaming requests (cache only on complete response)
- Tool calls with side effects

#### Provider Adapter
```python
class ProviderAdapter:
    def complete(self, messages, model, tools, max_tokens) -> Response
    def stream(self, ...) -> AsyncIterator[Chunk]
    def embed(self, texts) -> List[Vector]
    def count_tokens(self, messages) -> int
```

Each provider (OpenAI, Anthropic, Gemini) implements this interface. Gateway never calls provider SDK directly — always through adapter.

#### Router — Routing Strategies
| Strategy | Logic |
|---|---|
| Cost-optimized | Cheapest model meeting capability requirement |
| Latency-optimized | Provider with lowest live p50 (tracked in Redis, refreshed every 30s) |
| Failover | Primary → secondary on 5xx or timeout (circuit breaker per provider) |
| Capability-based | Route vision/tool-use/long-context to capable provider |
| A/B | Split traffic % for model comparison experiments |

#### Guardrails / Safety Layer
- PII scrubbing: scan request messages for emails, phone, SSN → redact before logging and caching
- Prompt injection: detect `ignore previous instructions` patterns → flag, optionally block
- Output filtering: optional profanity / harmful content filter (configurable per tenant)

### Algorithms & Strategies
- **Rate limiting:** Token bucket per (tenant, model) in Redis. Bucket refills at `tpm_limit / 60` tokens/sec. Reject with 429 when empty.
- **Token pre-estimation:** Use tiktoken (OpenAI) or model-specific tokenizer to estimate input tokens before call — needed for token-level rate check. Actual count reconciled after response.
- **Circuit breaker per provider:** Open after 5 consecutive 5xx in 10s window → route to backup → half-open after 30s.
- **Caching strategy:** Semantic cache with cosine threshold 0.95. Cache key includes `system_prompt_hash` to prevent cross-context hits. TTL: 1 hour default, configurable.
- **Cost calculation:** `cost = (input_tokens × input_price) + (output_tokens × output_price)`. Price table stored in config (Redis), not DB — update without deploy.

### Security Design
- AuthN: HMAC-SHA256 API key validation (compare key_hash, never store raw key)
- AuthZ: tenant_id from resolved key — client cannot spoof
- Encryption at rest: ClickHouse and audit S3 bucket encrypted
- Encryption in transit: TLS 1.3 to clients and providers
- Secrets management: provider API keys in AWS Secrets Manager, rotated monthly
- PII handling: redact before audit log; raw prompts never stored long-term
- Audit log: immutable (S3 Object Lock) — compliance requirement

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Gateway latency (p99) | Histogram | > 20ms overhead |
| Cache hit rate | Gauge | < 25% |
| Provider error rate (per provider) | Counter | > 1% |
| Rate limit rejection rate | Counter | > 5% |
| Daily cost vs budget | Gauge | > 90% budget |

### Logging
- Per request: request_id, tenant_id, model, provider, latency, tokens_in, tokens_out, cost, cache_hit, status
- Do NOT log: raw message content (PII risk); log content_hash instead for dedup

### Tracing
- Trace: Client → Gateway → Cache → Provider
- Annotate: cache lookup latency, provider call latency, token estimation time
- Sample: 5% normal, 100% on errors or latency > 500ms

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Provider degraded | Error rate > 1% for 2min | Failover, notify team |
| Budget near limit | Tenant at 90% daily budget | Notify tenant, throttle |
| Cache hit rate drop | < 20% for 10min | Investigate embedding quality or threshold |
| Gateway latency spike | p99 > 50ms overhead | Scale gateway nodes |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Semantic cache (vector sim) | 30%+ hit rate | Occasional wrong cache hit (tune threshold) |
| Async cost tracking | Zero latency on critical path | Eventual consistency — budget enforcement lags < 1min |
| Token-level rate limiting | Prevents prompt stuffing abuse | Need token pre-estimation (slight overhead) |
| Stateless gateway | Easy horizontal scale | Requires Redis + Vector DB for all state |
| OpenAI-compatible API | Easy client migration | Lowest-common-denominator feature set |
| Circuit breaker per provider | Fast failover | Flapping risk on unstable providers (tune thresholds) |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| All providers down | Simultaneous outage | Local model fallback (Ollama/vLLM); degrade gracefully |
| Cache poisoning | Bad response cached | TTL expiry; user feedback triggers cache invalidation |
| Budget overrun | Async lag in cost tracker | Soft limit (warn at 90%) + hard limit via Redis counter (sync) |
| Rate limit bypass | Token pre-estimation wrong | Reconcile post-response; deduct difference from next bucket |
| Provider API change | Breaking schema change | Provider adapter absorbs — update adapter, not gateway core |
| Redis down | Rate limit state lost | Fail open (allow requests) with conservative global limit |

---

## Feedback Loop & Improvements
- Track cache hit/miss + user satisfaction (if gateway has access to downstream rating)
- Tune similarity threshold per tenant based on observed hit quality
- A/B routing experiments: compare model outputs on same query → quality metrics → update default routing
- Cost anomaly detection: alert on 3x spike vs 7-day rolling average per tenant

---

## Interview Tips

### What Interviewers Expect
- Semantic cache is the key differentiator — explain why exact match is useless for LLMs
- Cost tracking pipeline: why async (Kafka), why ClickHouse (OLAP for cost queries)
- Rate limiting: both request-level AND token-level — most candidates miss token-level
- Failover: circuit breaker pattern per provider

### Common Follow-ups
- How to handle streaming responses? SSE passthrough — cache only on full response completion
- PII in prompts → scrub before logging; never cache raw user content
- How to support multimodal (image) requests? Provider adapter handles; image tokens counted separately
- How to compare model quality? A/B routing with same query → LLM-as-judge scoring
