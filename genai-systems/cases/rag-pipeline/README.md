# RAG Pipeline

Design a Retrieval-Augmented Generation system that answers natural language questions grounded in a private knowledge base.

---

## Requirements

### Functional Requirements
- Ingest documents (PDF, HTML, Markdown, DOCX, etc.)
- Answer natural language queries using ingested content
- Return source citations alongside answers
- Support incremental document updates (add, update, delete)
- Filter results by metadata (date, department, document type)

### Non-Functional Requirements
- Latency: query response < 2s p95 end-to-end
- Availability: 99.9% on query path; ingestion can tolerate delays
- Consistency: new documents searchable within 5 minutes of upload
- Security & Privacy: access control per document (tenant/role-based); PII not logged raw
- Scalability: 10M+ document chunks; 1,000 QPS

### Out of Scope
- User authentication (handled upstream)
- Document access permissions UI
- Fine-tuning LLM on ingested content
- Real-time streaming ingestion (batch ingestion sufficient)

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| DAU | 50,000 |
| Queries/day | 500,000 |
| Queries/sec (peak) | ~1,000 |
| Ingestion events/day | 10,000 docs |

### Storage Estimation
| Metric | Value |
|---|---|
| Avg doc size | 50 KB |
| Chunks per doc (512 tokens, 50% overlap) | ~50 chunks |
| Total chunks (1M docs) | ~50M chunks |
| Embedding size (1536-dim float32) | 6 KB/chunk |
| Vector index total | ~300 GB |
| Raw doc storage (5yr) | ~5 TB |

### Memory Estimation
| Metric | Value |
|---|---|
| Hot vector index (in-memory HNSW) | ~50 GB |
| Query embedding cache (Redis) | ~5 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Ingestion throughput | ~500 KB/s |
| Query response (avg 1KB) | ~1 MB/s |

### Cost Estimation
| Metric | Value |
|---|---|
| Embeddings at ingest (50M chunks) | ~$10,000 one-time |
| Query embeddings/day (1,000 QPS) | ~500M tokens/day |
| LLM generation/day | ~500M tokens/day |
| Total cost/day (est.) | ~$2,000–$5,000 |

---

## High-Level Design

### Architecture Style
**Event-driven (ingestion) + Synchronous Microservices (query path)** — ingestion is decoupled async pipeline triggered by document upload events; query path is low-latency sync microservices.

### Architecture Diagram

**Ingestion Pipeline:**
```
Document Upload (S3 / API)
    ↓
Message Queue (Kafka)
    ↓
Ingestion Worker
    ├── Parser (PDF/HTML/DOCX → text)
    ├── Chunker
    ├── Embedder (batch)
    └── Writer
         ├── Vector DB (embeddings + metadata)
         └── Metadata Store (doc info, chunk map)
```

**Query Pipeline:**
```
User Query
    ↓
Query Service
    ├── Query Embedder
    ├── Vector DB (ANN search, top-k)
    ├── Reranker (cross-encoder)
    ├── Context Assembler
    └── LLM (generation)
         ↓
    Response + Citations
```

### Core Components
| Component | Responsibility |
|---|---|
| Parser | Convert raw docs to clean text; preserve structure (headers, tables) |
| Chunker | Split text into overlapping chunks; attach metadata |
| Embedder | Convert chunks/queries to vector representations |
| Vector DB | Store embeddings; ANN search at query time |
| Metadata Store | Doc registry, chunk map, access control |
| Reranker | Cross-encoder re-scores top-k retrieved chunks for precision |
| Context Assembler | Select and order chunks within LLM context window |
| LLM | Generate grounded answer from assembled context |

### Data Flow

**Ingestion:**
1. Doc uploaded → event published to Kafka topic `doc.uploaded`
2. Ingestion worker consumes event → downloads doc from S3
3. Parser extracts clean text + structure
4. Chunker splits into 512-token chunks with 50-token overlap
5. Embedder batches chunks → calls embedding API → returns vectors
6. Vectors + metadata written to Vector DB and Metadata Store

**Query:**
1. User query arrives → embed query (cache hit check first)
2. Vector DB ANN search → top-20 candidate chunks
3. Optional BM25 keyword search → merge with RRF fusion
4. Reranker scores all candidates → top-5 kept
5. Context assembler builds prompt with chunks + citations
6. LLM generates answer → returned with source links

### Key Design Decisions
- **Hybrid search (BM25 + vector):** Pure vector search misses exact keyword matches (product codes, names). BM25 + vector fused with RRF handles both semantic and lexical retrieval.
- **Reranker stage:** ANN search optimizes for recall; cross-encoder reranker optimizes for precision. Two-stage avoids precision/recall tradeoff in one step.
- **Chunk overlap:** 50-token overlap prevents context loss at boundaries. Tradeoff: 1.5× chunk count.
- **Event-driven ingestion:** Decouples upload from processing. Ingestion spikes don't affect query latency.
- **Model Selection:** `text-embedding-3-large` (OpenAI) for quality; `text-embedding-3-small` for cost-sensitive deployments. Reranker: Cohere Rerank or local cross-encoder (BGE).

---

## Low-Level Design

### Data Models
```
documents (PostgreSQL):
  doc_id        UUID      PK
  tenant_id     UUID
  source_url    TEXT
  filename      TEXT
  doc_type      ENUM (pdf, html, md, docx)
  status        ENUM (pending, processing, indexed, failed)
  created_at    TIMESTAMP
  updated_at    TIMESTAMP

chunks (PostgreSQL):
  chunk_id      UUID      PK
  doc_id        UUID      FK → documents
  chunk_index   INTEGER
  text          TEXT
  token_count   INTEGER
  page_number   INTEGER
  section       TEXT

vector_index (Vector DB — e.g. Qdrant):
  id            = chunk_id
  vector        = float32[1536]
  payload       = { doc_id, tenant_id, text, page_number, section, created_at }
```

### API Design
```
POST /ingest
Request:  { "source_url": "s3://...", "doc_type": "pdf", "metadata": {} }
Response: { "job_id": "...", "status": "queued" }

GET /ingest/{job_id}
Response: { "status": "indexed", "chunk_count": 48, "doc_id": "..." }

POST /query
Request:  {
  "query": "What is our refund policy?",
  "top_k": 5,
  "filters": { "doc_type": "pdf", "date_after": "2024-01-01" }
}
Response: {
  "answer": "...",
  "sources": [{ "doc_id": "...", "chunk_id": "...", "page": 3, "score": 0.94 }]
}

DELETE /documents/{doc_id}
Response: { "status": "deleted", "chunks_removed": 48 }
```

### Component Deep-Dives

#### Chunker
```
Document text
    ↓
Recursive character splitter
    ├── Split on: \n\n → \n → sentence → word
    ├── Target chunk size: 512 tokens
    └── Overlap: 50 tokens
    ↓
Each chunk → attach { doc_id, chunk_index, page_number, section_header }
```

Semantic chunking (split on meaning boundaries) preferred for structured docs (legal, financial). Fixed-size for unstructured text.

#### Hybrid Search + RRF Fusion
```
Query
  ├── Dense: embed → Vector DB ANN → ranked list A (top-20)
  └── Sparse: BM25 index → ranked list B (top-20)
                    ↓
         RRF score = Σ 1 / (k + rank_i)   where k=60
                    ↓
         Merged ranked list → Reranker → top-5
```

#### Guardrails / Safety Layer
- **Input:** Max query length (4096 chars); reject if prompt injection pattern detected (`ignore previous instructions`, `jailbreak`, etc.)
- **Output:** If retrieved context scores below threshold (< 0.7 similarity), return "I don't have enough information" instead of hallucinating
- **Access control:** `tenant_id` filter on every vector search — users cannot retrieve other tenants' chunks
- **PII:** Strip PII from query before logging; flag docs with PII for restricted access

### Algorithms & Strategies
- **Chunking strategy:** Recursive split → 512 tokens → 50-token overlap. Semantic chunking for legal/financial docs.
- **Embedding model:** `text-embedding-3-large` (1536-dim) for quality; batch at ingest (up to 2048 chunks/call). Cache query embeddings in Redis (TTL: 1hr) — same queries reuse embedding.
- **ANN indexing:** HNSW in Qdrant. `ef_construction=200, m=16` for quality-speed balance. Full index loaded in memory for latency.
- **Reranking:** Cohere Rerank API or local `bge-reranker-large`. Input: (query, chunk_text) pairs. Output: relevance score → sort → keep top-5.
- **Context assembly:** Sort top-5 chunks by page/section order (not relevance score) for coherent reading. Include chunk source metadata for citations. Fit within context window (8K tokens).
- **Prompt design:** System prompt includes grounding instruction: "Answer ONLY using the provided context. If context is insufficient, say so." Context inserted between `<context>` tags.

### Security Design
- AuthN/AuthZ: API key → tenant_id resolution; all vector queries scoped to tenant_id
- Document access: role-based filter in metadata (doc.allowed_roles); checked at context assembly
- Encryption at rest: S3 (SSE-S3), PostgreSQL (pg_crypto), Vector DB (at-rest encryption)
- Encryption in transit: TLS 1.3
- Secrets management: embedding API keys and LLM keys in AWS Secrets Manager
- PII handling: scan ingested docs for PII (presidio); flag for restricted access or redact before chunking

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Query latency (p95) | Histogram | > 2s |
| Retrieval recall@5 | Gauge | < 0.7 (eval set) |
| Reranker latency | Histogram | > 300ms |
| Ingestion lag | Gauge | > 10 min |
| Vector DB ANN latency | Histogram | > 50ms |
| Cache hit rate (query embed) | Gauge | < 20% |

### Logging
- Per query: request_id, tenant_id, query_hash (not raw), top_k_doc_ids, reranker_scores, llm_latency, answer_length
- Per ingestion: doc_id, chunk_count, embed_latency, status
- Do NOT log: raw query text or doc content (PII risk); log hashes and IDs

### Tracing
- Trace spans: query_embed → vector_search → rerank → context_assemble → llm_call
- Useful for: identifying which stage contributes most to latency
- Sample: 10% normal; 100% on errors or latency > 3s

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Query latency spike | p95 > 2s for 5min | Page on-call |
| Ingestion backlog | Queue depth > 1000 for 10min | Scale ingestion workers |
| Retrieval quality drop | Recall@5 < 0.65 on eval set | Investigate embedding drift |
| Vector DB memory | > 90% | Add capacity |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| ANN over exact NN | ms-latency retrieval | ~2% recall loss (mitigated by reranker) |
| Reranker stage | Precision | Extra 200ms latency |
| Hybrid search (BM25 + vector) | Handles keyword + semantic | Two retrieval systems to maintain |
| Chunk overlap | Boundary coverage | 1.5× storage and embed cost |
| Event-driven ingestion | Decoupled, scalable | Eventual consistency (5min lag) |
| Separate vector + metadata store | Query flexibility | Sync complexity on updates/deletes |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Hallucination | Context insufficient or misleading | Low-similarity fallback: "I don't know"; grounding prompt |
| Retrieval miss | Poor chunking or embedding model mismatch | Hybrid search (BM25 catches keyword misses) |
| Context overflow | Too many chunks exceed context window | Rerank → top-5; summarize long chunks |
| Stale results | Index lag after doc update/delete | Soft-delete chunks on update; async re-embed; invalidate query cache |
| Embedding model outage | External API down | Fallback to local embedding model (BGE) |
| Vector DB OOM | Index grew beyond memory | Disk-based HNSW (Qdrant supports); shard index |

---

## Feedback Loop & Improvements
- User feedback: thumbs up/down per answer → logged with (query_hash, chunk_ids)
- Low-rated answers trigger: manual review → identify retrieval or generation failure
- Retrieval improvement: add failing queries to eval set; tune similarity threshold or chunking strategy
- Eval cadence: run RAGAs (retrieval recall, context precision, answer faithfulness) on golden set per deploy
- Long-term: use feedback to fine-tune reranker on domain-specific relevance

---

## Interview Tips

### What Interviewers Expect
- Chunking strategy is first question — answer: recursive split + overlap; semantic chunking for structured docs
- Two-stage retrieval: ANN for recall → reranker for precision
- Hybrid search (BM25 + vector) beats pure vector for enterprise knowledge bases
- Grounding the LLM: system prompt + context injection + "I don't know" fallback

### Common Follow-ups
- Multi-hop RAG: query requires chaining multiple retrievals — agent-style loop
- How to handle tables/images in PDFs? → Multimodal embeddings (CLIP) or table-aware parsers
- Evaluation: RAGAs framework — retrieval recall, context precision, answer faithfulness
- Scaling Vector DB beyond single node: shard by tenant_id or doc category
