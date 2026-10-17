# Vector DB Operations

> **Purpose**: Production operations on pgvector 0.8, Qdrant 1.19 and Databricks AI Search — indexes, filtered search, native hybrid search, and quantization
> **MCP Validated**: 2026-10-17

## When to Use

- Storing and querying embeddings with metadata filters at production scale
- ACID guarantees alongside vectors (pgvector), rich payload filtering and quantization (Qdrant), or managed sync from
  lakehouse tables (Databricks AI Search)
- Hybrid search: combine keyword matching (exact terms, IDs) with semantic similarity

## Implementation

```sql
-- pgvector 0.8.7 (Postgres 13+). vector up to 2,000 dims, halfvec up to 4,000, bit up to 64,000.
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id         BIGSERIAL PRIMARY KEY,
    content    TEXT NOT NULL,
    metadata   JSONB DEFAULT '{}',
    embedding  vector(1536),
    textsearch tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);

-- HNSW index (cosine). m = connections per layer, ef_construction = build-time candidate list
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200);
CREATE INDEX ON documents USING gin (textsearch);

-- Filtered ANN: iterative index scans (0.8.0+) keep scanning until enough rows pass the filter
SET hnsw.ef_search = 100;
SET hnsw.iterative_scan = relaxed_order;          -- or strict_order
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
WHERE metadata->>'department' = 'engineering'
ORDER BY embedding <=> $1::vector
LIMIT 10;

-- Hybrid: Postgres full-text search for the keyword side; fuse with RRF (or a cross-encoder) in the app
SELECT id, ts_rank_cd(textsearch, query) AS rank
FROM documents, plainto_tsquery('english', 'executor FetchFailed') query
WHERE textsearch @@ query ORDER BY rank DESC LIMIT 20;

-- Smaller working set: half-precision index over the same column, or binary quantization + re-rank
CREATE INDEX ON documents USING hnsw ((embedding::halfvec(1536)) halfvec_cosine_ops);
```

```python
"""Qdrant 1.19 (qdrant-client 1.19.1): Query API, payload indexes, native hybrid, TurboQuant."""
from qdrant_client import QdrantClient, models

client = QdrantClient(url="http://localhost:6333")   # Cloud: url=..., api_key=os.environ["QDRANT_API_KEY"]

client.create_collection(
    collection_name="documents",
    vectors_config={"dense": models.VectorParams(size=1536, distance=models.Distance.COSINE)},
    sparse_vectors_config={"sparse": models.SparseVectorParams()},
    hnsw_config=models.HnswConfigDiff(m=16, ef_construct=200),
    quantization_config=models.TurboQuantization(          # 4-bit default; available since v1.18.0
        turbo=models.TurboQuantQuantizationConfig(memory=models.Memory.PINNED)),
)
client.create_payload_index("documents", field_name="department",
                            field_schema=models.PayloadSchemaType.KEYWORD)

client.upsert(collection_name="documents", points=[
    models.PointStruct(id=1, vector={"dense": dense_vec,
                                     "sparse": models.SparseVector(indices=sp_idx, values=sp_val)},
                       payload={"department": "engineering", "year": 2026, "source": "wiki"}),
])

# Filtered dense search with the Query API (v1.10+; the interface the current docs use)
hits = client.query_points(
    collection_name="documents", query=query_dense, using="dense", limit=10,
    query_filter=models.Filter(must=[
        models.FieldCondition(key="department", match=models.MatchValue(value="engineering")),
        models.FieldCondition(key="year", range=models.Range(gte=2025)),
    ]),
    with_payload=True,
).points

# Native hybrid: dense + sparse prefetch fused with Reciprocal Rank Fusion
hybrid = client.query_points(
    collection_name="documents",
    prefetch=[
        models.Prefetch(query=models.SparseVector(indices=q_idx, values=q_val), using="sparse", limit=20),
        models.Prefetch(query=query_dense, using="dense", limit=20),
    ],
    query=models.RrfQuery(rrf=models.Rrf()),          # or DBSF fusion
    limit=10,
).points
```

```python
"""Databricks AI Search: managed Delta Sync index, hybrid query."""
from databricks.sdk import WorkspaceClient

res = WorkspaceClient().vector_search_indexes.query_index(
    index_name="main.docs.chunks_index", columns=["chunk_id", "chunk_to_retrieve"],
    query_text="executor FetchFailed", query_type="HYBRID", num_results=10)
```

## Configuration

| Engine | Setting | Tradeoff |
|--------|---------|----------|
| pgvector HNSW | `m`, `ef_construction` | Higher = better recall, more memory and build time |
| pgvector | `hnsw.ef_search` (runtime) | Higher = better recall, slower |
| pgvector | `hnsw.iterative_scan` | `relaxed_order` returns more filtered rows; `strict_order` keeps exact distance order |
| pgvector IVFFlat | `lists`, `ivfflat.probes` | More probes = better recall, slower |
| Qdrant | `HnswConfigDiff(m, ef_construct)` | Same as HNSW above |
| Qdrant | Quantization | 4-bit TurboQuant suggested over scalar int8 (except Manhattan distance); binary for high-dim centered vectors |
| AI Search | Endpoint type, `query_type` | See [Databricks AI Search](../concepts/databricks-ai-search.md) |

| Distance | pgvector operator | Qdrant |
|----------|-------------------|--------|
| Cosine | `<=>` | `Distance.COSINE` |
| L2 | `<->` | `Distance.EUCLID` |
| Inner product | `<#>` (negative inner product) | `Distance.DOT` |

## Example Usage

```python
# pgvector with SQLAlchemy (the pgvector Python package, 0.5.1 on PyPI, adds ORM column types)
from sqlalchemy import create_engine, text

engine = create_engine("postgresql+psycopg://app:***@localhost/vectordb")
with engine.connect() as conn:
    rows = conn.execute(
        text("SELECT id, content, 1 - (embedding <=> CAST(:q AS vector)) AS sim "
             "FROM documents ORDER BY embedding <=> CAST(:q AS vector) LIMIT 5"),
        {"q": str(query_embedding)},
    )
    for row in rows:
        print(f"[{row.sim:.3f}] {row.content[:80]}")
```

## See Also

- [Vector Databases Concept](../concepts/vector-databases.md)
- [Databricks AI Search](../concepts/databricks-ai-search.md)
- [RAG Pipeline Implementation](rag-pipeline-implementation.md)

## Sources

- https://github.com/pgvector/pgvector
- https://qdrant.tech/documentation/search/search/
- https://qdrant.tech/documentation/concepts/hybrid-queries/
- https://qdrant.tech/documentation/guides/quantization/
- https://github.com/qdrant/qdrant-client (create_collection, VectorParams, create_payload_index)
- https://docs.databricks.com/aws/en/ai-search/ai-search/
