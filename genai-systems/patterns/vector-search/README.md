# Pattern: Vector Search

Convert data to dense embeddings and retrieve semantically similar items via approximate nearest neighbor (ANN) search — enabling meaning-based lookup beyond keyword matching.

---

## What It Is

Vector search maps text (or images, audio) into high-dimensional float vectors where semantic similarity = geometric proximity. A query embedding is compared against a pre-built index of document embeddings; the closest vectors are returned as results.

It powers: semantic search, RAG retrieval, recommendation, deduplication, anomaly detection.

---

## When to Use

| Use | Avoid |
|---|---|
| Semantic/meaning-based retrieval | Exact keyword match is sufficient (use BM25) |
| Query and documents express same concept differently | Structured data filtering (use SQL) |
| Cross-lingual retrieval | Small dataset (< 10K items) — brute force exact search fine |
| Image/multimodal similarity | Boolean / facet filtering only |
| Recommendation via embedding similarity | |

**Rule:** use hybrid (BM25 + vector) when both exact and semantic match matter — which is most enterprise use cases.

---

## Core Concepts

### Embedding
```
text → Embedding Model → float32[1536]

"What is the refund policy?"  → [0.12, -0.87, 0.34, ...]
"How do I return an item?"    → [0.11, -0.85, 0.36, ...]  ← similar vector
"Best pizza in Rome?"         → [0.91,  0.23, -0.54, ...] ← distant vector
```

Similarity measured via cosine similarity (angle between vectors) or dot product.

### Exact vs Approximate NN

| Method | Recall | Latency | Use |
|---|---|---|---|
| Exact (brute force) | 100% | O(N × D) — slow | < 100K vectors, latency flexible |
| ANN (HNSW, IVF) | ~95–99% | O(log N) — fast | Production; millions of vectors |

ANN trades ~1–5% recall for 100–1000× speedup. Recall gap closed by reranker.

### Embedding Model Choice
| Model | Dims | Best for |
|---|---|---|
| `text-embedding-3-large` (OpenAI) | 3072 | Quality-first, general purpose |
| `text-embedding-3-small` (OpenAI) | 1536 | Cost/speed balance |
| `text-embedding-ada-002` (OpenAI) | 1536 | Legacy, widely used |
| `BGE-large-en` | 1024 | Open-source, strong benchmark |
| `E5-mistral-7b` | 4096 | Best open-source quality |
| `CLIP` (OpenAI) | 512 | Multimodal (text + image same space) |

---

## Architecture

### Indexing Pipeline
```
Raw Documents
    ↓
Chunker (split into segments ≤ model token limit)
    ↓
Embedder (batch API calls to embedding model)
    ↓
Vector DB (index vectors + metadata)
    ↓
Ready for search
```

### Query Pipeline
```
User Query
    ↓
Query Embedder (same model as indexing — critical)
    ↓
ANN Search (top-K candidates, cosine similarity)
    ↓
[Optional] Reranker (cross-encoder, precision boost)
    ↓
Results + Metadata
```

### ANN Index Algorithms

#### HNSW (Hierarchical Navigable Small World) ← recommended
```
Multi-layer graph structure:
  Layer 2 (sparse): long-range connections
  Layer 1 (medium): medium-range
  Layer 0 (dense):  all nodes, short-range

Search: start at top layer → greedy descent → Layer 0 → find nearest

Params:
  M = 16–32          (edges per node — higher = better recall, more memory)
  ef_construction = 200  (build-time exploration — higher = better quality)
  ef_search = 50–200     (query-time exploration — tune for recall/latency tradeoff)

Memory: ~(M × 8 bytes × N) overhead per index
```

#### IVF (Inverted File Index) — for very large indexes
```
K-means cluster vectors into C centroids
Index: centroid → list of nearby vectors

Search:
  1. Find nearest nprobe centroids to query
  2. Search only those clusters
  3. Return top-K

Memory: lower than HNSW; trades some recall
Use when: > 100M vectors; memory-constrained
```

---

## Implementation Details

### Indexing (Python + Qdrant)
```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
import openai

client = QdrantClient(url="http://localhost:6333")

# Create collection
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

# Batch embed + index
def index_chunks(chunks: list[dict]):
    texts = [c["text"] for c in chunks]

    # Batch embedding (max 2048 per call)
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    vectors = [r.embedding for r in response.data]

    # Upsert to vector DB
    points = [
        PointStruct(
            id=chunk["chunk_id"],
            vector=vector,
            payload={
                "text": chunk["text"],
                "doc_id": chunk["doc_id"],
                "page": chunk["page"],
                "section": chunk["section"],
            }
        )
        for chunk, vector in zip(chunks, vectors)
    ]
    client.upsert(collection_name="documents", points=points)
```

### Query (with metadata filter)
```python
def search(query: str, tenant_id: str, top_k: int = 20) -> list:
    query_vector = openai.embeddings.create(
        model="text-embedding-3-small",
        input=query
    ).data[0].embedding

    results = client.search(
        collection_name="documents",
        query_vector=query_vector,
        query_filter={                          # metadata filter applied at search time
            "must": [{"key": "tenant_id", "match": {"value": tenant_id}}]
        },
        limit=top_k,
        with_payload=True,
        score_threshold=0.7,                   # minimum similarity score
    )

    return [
        {"text": r.payload["text"], "score": r.score, "doc_id": r.payload["doc_id"]}
        for r in results
    ]
```

### Hybrid Search (BM25 + Vector with RRF)
```python
def hybrid_search(query: str, top_k: int = 20) -> list:
    # Dense retrieval
    vector_results = vector_search(query, top_k=50)

    # Sparse retrieval (BM25 via Elasticsearch or Qdrant sparse)
    bm25_results = bm25_search(query, top_k=50)

    # Reciprocal Rank Fusion
    k = 60
    scores = {}
    for rank, result in enumerate(vector_results):
        scores[result.id] = scores.get(result.id, 0) + 1 / (k + rank + 1)
    for rank, result in enumerate(bm25_results):
        scores[result.id] = scores.get(result.id, 0) + 1 / (k + rank + 1)

    merged = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return [fetch(id) for id, _ in merged[:top_k]]
```

### Reranker (Cross-encoder)
```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, candidates: list[dict], top_k: int = 5) -> list:
    pairs = [(query, c["text"]) for c in candidates]
    scores = reranker.predict(pairs)           # cross-encoder scores all pairs
    ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return [c for c, _ in ranked[:top_k]]
```

Cross-encoder reads (query, doc) jointly → higher quality than bi-encoder dot product. Use on small candidate set (top-20 from ANN) for best latency.

### Embedding Cache
```python
# Cache query embeddings — same query asked multiple times
def get_embedding(text: str) -> list[float]:
    cache_key = f"embed:{hashlib.sha256(text.encode()).hexdigest()}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    vector = embed_api(text)
    redis.set(cache_key, json.dumps(vector), ex=3600)
    return vector
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| ANN over exact search | Speed (ms not seconds) | ~1–5% recall (closed by reranker) |
| HNSW over IVF | Better recall, faster query | Higher memory (~4 GB per 1M 1536-dim vectors) |
| Bi-encoder (separate query/doc encoding) | Fast: doc encoded offline | Less accurate than cross-encoder |
| Cross-encoder reranker | High precision on top-K | Slow: can't use on full corpus |
| Cosine over dot product | Magnitude-invariant | Dot product faster; fine if embeddings normalized |
| Hybrid (BM25 + vector) | Covers keyword + semantic | Two retrieval systems to maintain |
| Score threshold (0.7 cutoff) | No low-relevance results | May miss relevant results with different vocab |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Different embedding models for indexing vs querying | Always use same model; pin model version |
| No score threshold | Returns irrelevant results when no good match exists |
| Indexing full documents (not chunks) | Chunks (512 tokens) retrieve more precise context |
| Missing metadata filters (tenant isolation) | Always filter by tenant_id to prevent data leakage |
| No reranker on production | ANN recall gaps hurt quality; add cross-encoder reranker |
| Rebuilding index on every update | Incremental upsert for adds; soft-delete + nightly rebuild for deletes |
| Ignoring embedding model upgrades | New model = breaking change to index (incompatible vector space); must re-embed all docs |

---

## Used In
- [RAG Pipeline](../../cases/rag-pipeline/) — core retrieval: chunk embeddings → ANN search → reranker → LLM context
- [AI Agent](../../cases/ai-agent/) — long-term memory retrieval: goal embedding → similar past tasks/preferences
- [LLM Gateway](../../cases/llm-gateway/) — semantic response cache: query embedding → similar past queries → cached response
- [Recommendation Engine](../../../ml-systems/cases/recommendation-engine/) — item/user embeddings → ANN for candidate generation

---

## Vector DB Options
| DB | ANN Algorithm | Best for |
|---|---|---|
| **Qdrant** | HNSW | Production-grade; rich filtering; Rust-based |
| **Pinecone** | Proprietary (HNSW-based) | Managed; easy ops; expensive at scale |
| **Weaviate** | HNSW | Multi-modal; built-in hybrid search |
| **pgvector** | IVF / HNSW | Already use PostgreSQL; smaller scale |
| **Faiss** | HNSW / IVF / PQ | Library (not server); max perf; self-managed |
| **Chroma** | HNSW | Local dev / prototyping |
| **Milvus** | IVF + HNSW | Large-scale (billions of vectors) |
