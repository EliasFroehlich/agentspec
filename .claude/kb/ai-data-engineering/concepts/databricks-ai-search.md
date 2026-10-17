# Databricks AI Search

> **Purpose**: Databricks AI Search (formerly Vector Search) — endpoint types, index types, ANN/hybrid/full-text queries, embedding models, and SDK/SQL access
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

Databricks AI Search (formerly Databricks Vector Search) is the managed retrieval service of the platform, governed by
Unity Catalog. You create an index from a source table — Delta Lake, streaming table, or managed Iceberg v3+ table — and
Databricks keeps it in sync, optionally computing the embeddings itself. Queries support approximate nearest neighbor
(ANN), hybrid keyword-similarity search (BM25 + vectors), full-text keyword search (Beta), filtering and reranking. API and
SDK names still use `vector_search` / `vector-search` (`w.vector_search_indexes`, `vector_search()` SQL function).

## The Concept

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# 1. Endpoint: STANDARD (low latency, high QPS) or STORAGE_OPTIMIZED (> 1B vectors, cheaper)
w.vector_search_endpoints.create_endpoint(name="docs-endpoint", endpoint_type="STANDARD")

# 2. Delta Sync index with Databricks-computed embeddings (source must have a primary key + text column;
#    standard endpoints need change data feed on the source table)
w.vector_search_indexes.create_index(
    name="main.docs.chunks_index",
    endpoint_name="docs-endpoint",
    primary_key="chunk_id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "main.docs.chunks",
        "embedding_source_columns": [
            {"name": "chunk_to_embed", "embedding_model_endpoint_name": "databricks-gte-large-en"}],
        "pipeline_type": "TRIGGERED",          # or CONTINUOUS
    },
)

# 3. Query: ANN by default, HYBRID adds BM25 keyword scoring
res = w.vector_search_indexes.query_index(
    index_name="main.docs.chunks_index",
    columns=["chunk_id", "chunk_to_retrieve", "source_path"],
    query_text="executor lost FetchFailed error",
    query_type="HYBRID",
    num_results=10,
    filters_json='{"source_path": ["/Volumes/main/docs/runbooks/a.pdf"]}',   # standard endpoints
)
rows = res.result.data_array                    # score is the last column
```

```sql
-- SQL access (Public Preview); arguments by name. DBR 15.3+ uses query_text / query_vector.
SELECT * FROM vector_search(index => 'main.docs.chunks_index',
                            query_text => 'executor lost FetchFailed error',
                            query_type => 'HYBRID', num_results => 5);
```

## Quick Reference

| Endpoint | Capacity | Notes |
|----------|----------|-------|
| Standard | ~320M vectors at 768 dims | High QPS option; source needs change data feed |
| Storage-optimized | > 1B vectors at 768 dims | 10–20x faster indexing, ~250 ms query latency, lower price per vector; SQL-like string filters (`databricks-vectorsearch` `filters=`) |

| Index type | Embeddings | Sync | Use |
|------------|------------|------|-----|
| Delta Sync, managed embeddings | Databricks computes from a text column | Automatic from source | Easiest; one embedding model per column |
| Delta Sync, self-managed | You provide a vector column | Automatic from source | Custom or multimodal embeddings |
| Direct Vector Access | You provide vectors | You upsert/delete via API | Real-time writes outside Delta |
| Full-text index (Beta) | None | Triggered sync | Storage-optimized only; BM25 over exact terms and IDs |

| Query type | Needs | When |
|------------|-------|------|
| `ANN` (default) | `query_text` or `query_vector` | Semantic similarity |
| `HYBRID` | `query_text` (optionally `query_vector`) | Queries with codes, IDs, proper nouns |
| `FULL_TEXT` (Beta) | `query_text` | Keyword-only search on any endpoint |

Embedding models referenced for AI Search indexes include `databricks-gte-large-en`, `databricks-bge-large-en` and
`databricks-qwen3-embedding-0-6b` (Public Preview; up to 1024 dims, ~32K context, 100+ languages). Use a Foundation
Model APIs endpoint or your own serving endpoint for managed embeddings.

## Common Mistakes

### Wrong

```python
# Embedding the raw chunk and returning the context-enriched text to the LLM (or vice versa)
"embedding_source_columns": [{"name": "chunk_to_retrieve", ...}]
```

### Correct

```python
# ai_prep_search output: embed chunk_to_embed (title/headers/page context), return chunk_to_retrieve
"embedding_source_columns": [{"name": "chunk_to_embed",
                              "embedding_model_endpoint_name": "databricks-gte-large-en"}]
columns = ["chunk_id", "chunk_to_retrieve", "source_path"]
```

Other pitfalls: only synced columns come back in results (`columns_to_sync`); filter syntax differs between standard
(dict `filters_json`) and storage-optimized (SQL-like string) endpoints; triggered indexes need `sync_index()` after
source updates.

## Related

- [Vector Databases](../concepts/vector-databases.md)
- [Databricks-native RAG](../patterns/databricks-native-rag.md)
- [Vector DB Operations](../patterns/vector-db-operations.md)

## Sources

- https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_search
- https://docs.databricks.com/aws/en/machine-learning/foundation-model-apis/supported-models
- https://docs.databricks.com/aws/en/generative-ai/agent-bricks/knowledge-assistant (supported embedding models)
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_prep_search
