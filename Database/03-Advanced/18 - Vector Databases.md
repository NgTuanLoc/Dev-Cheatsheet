---
tags: [database, advanced, postgresql, nosql]
aliases: [pgvector, Pinecone, Qdrant, Embeddings, Similarity Search, RAG]
level: Advanced
---

# Vector Databases

> **One-liner**: Vector DBs index high-dimensional embeddings and answer "what's most similar to this?" in milliseconds — the storage layer behind semantic search and RAG.

---

## Quick Reference

| Engine | Type |
|--------|------|
| **pgvector** | Postgres extension — start here if you already use Postgres |
| **Pinecone** | managed; popular default |
| **Qdrant / Weaviate / Milvus** | open-source, dedicated vector DBs |
| **Chroma** | embedded-friendly; great for prototyping |
| **OpenSearch / Elasticsearch** | k-NN plugins; useful when you also need text search |
| **Redis** | vector module; fast in-memory |
| **Cosmos DB / Azure AI Search / AlloyDB** | cloud vector support |

| Concept | Meaning |
|---------|---------|
| **Embedding** | numeric vector representing text/image/audio meaning |
| **Dimensionality** | length of the vector (OpenAI: 1536, Cohere: 1024, etc.) |
| **Distance metric** | cosine / L2 / inner-product — how similarity is measured |
| **ANN** (Approximate Nearest Neighbor) | trade exact match for speed |
| **HNSW** | Hierarchical Navigable Small World — fast graph-based ANN |
| **IVFFlat** | Inverted File Flat — clustering-based ANN |
| **RAG** (Retrieval-Augmented Generation) | retrieve relevant docs, feed to LLM as context |
| **Hybrid search** | combine vector similarity + keyword filter |

---

## Core Concept

An **embedding** is a numerical vector that captures the semantic content of text/image/audio. Two pieces of text with similar meaning have vectors that are close together by some **distance metric** (cosine similarity is most common).

A **vector DB** stores embeddings and answers k-Nearest-Neighbor queries: "give me the top 10 most similar vectors to this query vector." Done naively, this is O(N×D) per query — fine for thousands of items, broken for millions. **ANN indexes** (HNSW, IVFFlat) trade a tiny accuracy loss for orders-of-magnitude speedup.

The canonical use case is **RAG** for LLMs:
1. Chunk your documents
2. Embed each chunk (OpenAI/Cohere/local model)
3. Store vectors + metadata in the DB
4. At query time: embed the user's question, fetch top-k similar chunks, pass to the LLM as context
5. LLM answers with grounded knowledge

For most apps already on Postgres, **pgvector** is the right answer: keeps relational + vector in one DB, uses your existing backups/HA/security. Reach for a dedicated vector DB only if you're at hundreds of millions of vectors or need specialized features (filtered search, replication patterns).

---

## Syntax & API

### pgvector setup
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title       TEXT NOT NULL,
    chunk       TEXT NOT NULL,
    embedding   VECTOR(1536) NOT NULL,         -- OpenAI ada-002 size
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Insert embeddings
```sql
INSERT INTO documents (title, chunk, embedding, metadata) VALUES
('Postgres docs', 'CREATE INDEX builds an index on a table.',
 '[0.012, -0.043, 0.087, ...]'::vector,        -- 1536 numbers
 '{"section":"DDL","page":42}');
```

```csharp
// .NET — get embedding from OpenAI then insert
var emb = await openAi.GetEmbeddingAsync(chunk);   // float[1536]
var vec = "[" + string.Join(",", emb) + "]";

await conn.ExecuteAsync(@"
    INSERT INTO documents (title, chunk, embedding, metadata)
    VALUES (@t, @c, @e::vector, @m::jsonb)",
    new { t = title, c = chunk, e = vec, m = JsonSerializer.Serialize(metadata) });
```

### Distance metrics in pgvector
```sql
-- Cosine distance (the default for text embeddings)
SELECT id, title, embedding <=> $1 AS distance
FROM documents
ORDER BY embedding <=> $1
LIMIT 5;

-- Euclidean (L2) distance
SELECT id, embedding <-> $1 AS distance FROM documents ORDER BY 2 LIMIT 5;

-- Inner product (negative — pgvector uses negated for ORDER BY)
SELECT id, (embedding <#> $1) * -1 AS similarity FROM documents ORDER BY 2 DESC LIMIT 5;
```

### ANN index (HNSW, pgvector 0.5+)
```sql
-- HNSW: fast, accurate, more memory
CREATE INDEX idx_docs_emb_hnsw
    ON documents USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- IVFFlat: less memory, slightly slower; needs ANALYZE-like training
CREATE INDEX idx_docs_emb_ivf
    ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
-- "lists" should be roughly sqrt(n_rows). Run ANALYZE first.
```

```sql
-- Tune query-time recall vs speed
SET hnsw.ef_search = 80;        -- higher = more accurate, slower (default 40)
SET ivfflat.probes = 10;        -- higher = more accurate, slower (default 1)
```

### Hybrid search (vector + filter)
```sql
-- Top-5 similar chunks within a category
SELECT id, title, embedding <=> $1 AS distance
FROM documents
WHERE metadata @> '{"category":"sql"}'
  AND created_at > now() - INTERVAL '1 year'
ORDER BY embedding <=> $1
LIMIT 5;
-- Index helps the embedding sort; filters reduce the candidate set.
```

### Combine with full-text search
```sql
-- Re-rank by linear combination of vector similarity + BM25-ish score
WITH vec AS (
    SELECT id, embedding <=> $1 AS d
    FROM documents
    ORDER BY embedding <=> $1
    LIMIT 50
), kw AS (
    SELECT id, ts_rank(search_tsv, websearch_to_tsquery('english', $2)) AS r
    FROM documents
    WHERE search_tsv @@ websearch_to_tsquery('english', $2)
)
SELECT d.id, d.title,
       0.7 * (1 - v.d) + 0.3 * COALESCE(k.r, 0) AS score
FROM documents d
LEFT JOIN vec v USING (id)
LEFT JOIN kw  k USING (id)
WHERE v.id IS NOT NULL OR k.id IS NOT NULL
ORDER BY score DESC
LIMIT 10;
```

### Pinecone (managed)
```python
import pinecone
pinecone.init(api_key="...")
index = pinecone.Index("docs")

index.upsert([
    ("doc-1", [0.012, -0.043, ...], {"category": "sql"})
])

result = index.query(vector=query_vec, top_k=5, filter={"category": "sql"})
```

### Qdrant (open-source, REST/gRPC)
```bash
# Create a collection
curl -X PUT 'http://localhost:6333/collections/docs' -d '{
  "vectors": { "size": 1536, "distance": "Cosine" }
}'

# Search
curl -X POST 'http://localhost:6333/collections/docs/points/search' -d '{
  "vector": [0.012, -0.043, ...],
  "limit": 5,
  "filter": { "must": [{ "key": "category", "match": { "value": "sql" } }] }
}'
```

---

## Common Patterns

```text
Pattern: chunking strategy for RAG
- Split documents into overlapping chunks (e.g., 500 tokens, 50-token overlap)
- Smaller chunks = more precise retrieval but more storage
- Embed each chunk, store with metadata (source, page, section)
- At query time: embed question → top-k → assemble context → call LLM
```

```text
Pattern: re-ranking
- Coarse retrieval: ANN gets top-50 candidates
- Fine ranking: a smaller, slower cross-encoder rerank to top-5
- Better quality than ANN alone; costs more compute
```

```sql
-- Pattern: store original + embedding in one row (simplifies joins)
ALTER TABLE products ADD COLUMN embedding VECTOR(1024);
UPDATE products SET embedding = $1 WHERE id = $2;
-- One DB; relational + vector + JSONB metadata together.
```

```text
Pattern: chunk dedup with cosine threshold
- Before insert, query top-1; if distance < 0.05, treat as duplicate
- Avoids cluttering the index with near-identical content
```

---

## Gotchas & Tips

- **Embedding model = vector dimension** — they must match between insert and query. Re-embed everything when switching models.
- **Cosine vs L2** — for text embeddings, cosine (or normalized inner product) is standard. L2 is fine for image embeddings if vectors are unit-normalized.
- **pgvector index choice**:
  - **HNSW**: fast queries, accurate, slower to build, more memory. Default for new code.
  - **IVFFlat**: cheaper memory, slightly slower queries, needs training data before index build.
- **Filter narrows AFTER ANN** — pgvector's ANN doesn't push down WHERE filters efficiently. If you filter heavily, the ANN may return < k matching results. Workaround: `LIMIT 200` before applying filters.
- **Approximate ≠ exact** — ANN can miss the absolute closest point. Tune `ef_search` / `probes` for desired recall.
- **Vector storage is large** — 1536-dim float32 = ~6KB per row. Compress with `halfvec` or quantization extensions if scale demands.
- **HNSW build time** — building on millions of rows takes hours. Use `CREATE INDEX CONCURRENTLY`.
- **Use `<=>` for cosine, `<->` for L2** — different operators; using the wrong one returns garbage rankings.
- **Keep embeddings out of git** — they're large binary blobs. Recompute or fetch from object storage.
- **RAG quality is mostly about chunking and retrieval, not the LLM** — invest there first.
- **Hybrid search wins** — pure vector misses keyword-precise queries (names, codes); pure keyword misses semantic. Combine.
- **Cost model differs** — managed vector DBs charge per stored vector + per query. pgvector charges only for compute + storage you'd pay anyway.
- **Don't store PII in embeddings** — embeddings are *invertible enough* that they can leak content via similarity searches. Treat them like ciphertext-with-leaks.

---

## See Also

- [[18 - JSON and JSONB]]
- [[16 - Search Engines]]
- [[19 - Full-Text Search]]
- [[17 - Cloud Databases]]
