# Pattern: Prompt Caching

Reuse previously computed LLM KV-cache states to avoid reprocessing repeated prompt prefixes — reducing latency and cost on repeated or structured inputs.

---

## What It Is

LLM inference computes a KV (key-value) cache for every token in the input. When the same prefix appears repeatedly (system prompt, few-shot examples, retrieved context), recomputing it is pure waste. Prompt caching stores and reuses those KV states so the model only processes the **new** portion of each request.

Two distinct layers:
1. **Provider-level prompt caching** (Anthropic, OpenAI) — caches KV states on the model server; automatic; reduces input token cost 50–90%
2. **Application-level semantic cache** (LLM Gateway, RAG) — cache full request → response pairs; avoids model call entirely on cache hit

---

## When to Use

### Provider-level (KV cache)
| Use | Avoid |
|---|---|
| Large fixed system prompt (> 1024 tokens) | Short prompts (< 500 tokens — savings minimal) |
| Repeated few-shot examples | Highly variable prompts (no shared prefix) |
| Document analysis (same doc, many questions) | Temperature > 0 with high variance expected output |
| Agent tasks with long static tool definitions | |

### Application-level (response cache)
| Use | Avoid |
|---|---|
| High query repetition (FAQ bots, support) | Creative generation (different answer expected) |
| Expensive generation (long output, large model) | Personalized responses (answer depends on user state) |
| LLM gateway serving many tenants | Temperature > 0 |

---

## Core Concepts

### Provider-Level: How It Works (Anthropic)
```
Request 1 (cache miss):
  [System prompt: 2000 tokens] [User message: 100 tokens]
  → Model processes all 2100 tokens
  → Cost: 2100 input tokens (full price)
  → Server caches KV state for system prompt prefix

Request 2 (cache hit — same system prompt):
  [System prompt: 2000 tokens ← CACHED] [User message: 150 tokens]
  → Model loads cached KV, processes only 150 new tokens
  → Cost: 2000 cache read tokens (10% of full price) + 150 input tokens
  → Latency: significantly lower (no recomputation of prefix)
```

Cache key = exact byte match of the prefix up to the cache checkpoint.

### Provider-Level: Cache Checkpoints
```
Anthropic: mark cache boundary explicitly in API
  messages = [
    {
      "role": "system",
      "content": [
        {
          "type": "text",
          "text": "<large system prompt>",
          "cache_control": { "type": "ephemeral" }  ← cache up to here
        }
      ]
    },
    { "role": "user", "content": "What is the refund policy?" }
  ]
```

OpenAI: automatic — caches longest matching prefix automatically (no explicit marking).

### Application-Level: Semantic Cache
```
Exact match cache: only hits if query is byte-identical → useless for LLMs
Semantic cache: embed query → find similar past queries → return cached response if similarity > threshold

Flow:
  Query → Embed → Vector search in cache index
    ├── Similarity > 0.95 → return cached response (skip LLM)
    └── Similarity < 0.95 → call LLM → store (embedding, response) in cache
```

---

## Architecture

### Provider-Level Caching
```
Application Code
    ↓
LLM API Request (with cache_control markers)
    ↓
Anthropic / OpenAI API
    ├── Cache miss: full KV computation → store prefix KV → return response
    └── Cache hit: load stored KV + process suffix only → return response
         ↓ billing
    Usage: { input_tokens, cache_read_tokens, cache_write_tokens, output_tokens }
```

### Application-Level Semantic Cache (in LLM Gateway)
```
Incoming Request
    ↓
Embed query (text-embedding-3-small — cheap)
    ↓
Vector search in cache index (Redis + vector extension, or Qdrant)
    ├── Similarity > threshold → return cached response (no LLM call)
    └── Miss → call LLM provider
                    ↓
              Store in cache: { embedding, response, model, system_prompt_hash }
              TTL: configurable (1hr default)
```

---

## Implementation Details

### Provider Cache — Anthropic API
```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = """
You are a helpful customer support assistant for Acme Corp.
<knowledge_base>
... 5000 tokens of product documentation ...
</knowledge_base>
"""  # Large, static — perfect for caching

def answer_question(user_question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},  # cache this prefix
            }
        ],
        messages=[{"role": "user", "content": user_question}],
    )

    # Check cache hit in usage
    usage = response.usage
    cache_hit = usage.cache_read_input_tokens > 0
    cost_saving = usage.cache_read_input_tokens * 0.9  # 90% cheaper than input tokens

    return response.content[0].text
```

### Provider Cache — Multi-turn Conversation
```python
# Cache the system prompt + conversation history up to a point
messages = [
    {
        "role": "user",
        "content": [
            {"type": "text", "text": long_document_to_analyze},
            {"type": "text", "text": "cache_control", ...}  # cache document
        ]
    },
    {"role": "assistant", "content": "I have read the document."},
    # ↑ cache checkpoint here — everything above reused across questions
    {"role": "user", "content": "What are the key risks mentioned?"},
]
```

Useful for: document Q&A sessions, code review (cache the codebase, ask many questions).

### Application-Level Semantic Cache
```python
import hashlib
import numpy as np
from qdrant_client import QdrantClient

class SemanticCache:
    def __init__(self, threshold: float = 0.95):
        self.threshold = threshold
        self.qdrant = QdrantClient(url="http://localhost:6333")
        self._ensure_collection()

    def _ensure_collection(self):
        self.qdrant.recreate_collection(
            collection_name="prompt_cache",
            vectors_config={"size": 1536, "distance": "Cosine"},
        )

    def _cache_key(self, model: str, system_prompt: str) -> str:
        return hashlib.sha256(f"{model}:{system_prompt}".encode()).hexdigest()[:16]

    def get(self, query: str, model: str, system_prompt: str) -> str | None:
        query_vector = embed(query)
        sp_hash = self._cache_key(model, system_prompt)

        results = self.qdrant.search(
            collection_name="prompt_cache",
            query_vector=query_vector,
            query_filter={"must": [{"key": "sp_hash", "match": {"value": sp_hash}}]},
            limit=1,
            score_threshold=self.threshold,
        )

        if results:
            return results[0].payload["response"]
        return None

    def set(self, query: str, response: str, model: str, system_prompt: str, ttl_sec: int = 3600):
        query_vector = embed(query)
        sp_hash = self._cache_key(model, system_prompt)
        cache_id = hashlib.sha256(query.encode()).hexdigest()

        self.qdrant.upsert(
            collection_name="prompt_cache",
            points=[{
                "id": cache_id,
                "vector": query_vector,
                "payload": {
                    "response": response,
                    "sp_hash": sp_hash,
                    "model": model,
                    "created_at": time.time(),
                    "expires_at": time.time() + ttl_sec,
                }
            }]
        )
```

### What NOT to Cache
```python
def should_cache(request) -> bool:
    # Never cache:
    if request.temperature > 0.2:      # non-deterministic output
        return False
    if request.stream:                  # streaming — only full responses cacheable
        return False
    if has_tool_calls_with_side_effects(request):  # tools with external effects
        return False
    if contains_pii(request.messages):  # don't cache PII-containing prompts
        return False
    return True
```

### Cost Calculation with Caching
```
Anthropic pricing (claude-sonnet-4-6, approx):
  Input tokens:        $3.00 / 1M
  Cache write tokens:  $3.75 / 1M  (25% premium to write to cache)
  Cache read tokens:   $0.30 / 1M  (90% cheaper than input)
  Output tokens:       $15.00 / 1M

Example: 1000 requests with 2000-token system prompt + 100-token user message

Without caching:
  2100 × 1000 × ($3.00/1M) = $6.30

With caching (1 cache write, 999 cache reads):
  Write: 2100 × $3.75/1M = $0.0079
  Reads: 2000 × 999 × $0.30/1M = $0.60
  User tokens: 100 × 1000 × $3.00/1M = $0.30
  Total: ~$0.91  ← 85% cost reduction
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Provider cache (KV) | Transparent, no correctness risk | Only prefix match; vendor-specific API |
| Semantic cache (response) | Skip LLM entirely on hit | Cache poisoning risk; stale responses |
| High similarity threshold (0.95) | Fewer wrong cache hits | Lower hit rate |
| Low similarity threshold (0.85) | Higher hit rate | Occasional semantically-different queries served same response |
| Short TTL (1hr) | Fresh responses | Lower hit rate on repeated questions across sessions |
| Long TTL (24hr) | Higher hit rate | Stale answers if knowledge base updated |
| Cache all tenants together | Shared hit rate benefit | Must strictly namespace by (tenant, system_prompt_hash) to prevent cross-tenant responses |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Exact match cache for LLM queries | Hit rate ~0%; use semantic cache |
| Caching non-deterministic responses (temp > 0) | Gate: only cache when temperature ≤ 0.2 |
| No system_prompt_hash in cache key | Different system prompts → different correct answers; namespace properly |
| Caching PII-containing prompts | Scrub PII before cache key; don't store raw queries |
| No TTL on cache entries | Stale answers served indefinitely; set TTL based on knowledge base update frequency |
| Cache write before validating response quality | Check response for errors/refusals before caching |
| Forgetting to mark cache checkpoints (Anthropic) | Without `cache_control`, no caching; must explicitly mark |

---

## Used In
- [LLM Gateway](../../cases/llm-gateway/) — semantic response cache (vector similarity, cosine > 0.95)
- [RAG Pipeline](../../cases/rag-pipeline/) — provider cache for system prompt + retrieved context prefix
- [AI Agent](../../cases/ai-agent/) — provider cache for system prompt + tool definitions (large static prefix)

---

## Provider Cache Availability (2026)
| Provider | Cache type | Minimum prefix | Discount |
|---|---|---|---|
| Anthropic | Explicit checkpoint | 1024 tokens | 90% off cache reads |
| OpenAI | Automatic prefix | 1024 tokens | 50% off cached tokens |
| Google Gemini | Explicit (Context Caching) | 32K tokens | ~75% off |
| Self-hosted (vLLM) | Automatic prefix | Configurable | Latency only (no billing) |
