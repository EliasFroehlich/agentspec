# Vector Databases

> **Purpose**: ANN algorithms, distance metrics, metadata filtering, and a platform comparison including Databricks AI Search
> **Confidence**: 0.90
> **MCP Validated**: 2026-03-26

## Overview

Vector databases are purpose-built for storing and searching high-dimensional embeddings using approximate nearest neighbor (ANN) algorithms. Common options are pgvector (Postgres extension, 0.8.7), Qdrant (Rust, rich filtering, 1.19), Pinecone (managed), Weaviate, Milvus (billion-scale), LanceDB (embedded), Chroma (prototyping) and, on Databricks, AI Search (formerly Vector Search) synced from lakehouse tables. The choice of index type, distance metric, and metadata filtering strategy determines retrieval latency, recall, and cost.

## The Concept

```text
ANN Index Algorithms

HNSW (Hierarchical Navigable Small World)
==========================================
Layer 3:  A ─────────────── D              (few nodes, long-range links)
Layer 2:  A ──── C ──── D ──── F          (more nodes, medium links)
Layer 1:  A ─ B ─ C ─ D ─ E ─ F ─ G      (all nodes, short links)

- Search: start at top layer, greedily descend
- Build time: O(N log N)  |  Query: O(log N)
- Memory: HIGH (in-memory graph)
- Best for: low-latency serving (<10ms), datasets < 50M vectors

IVF-PQ (Inverted File + Product Quantization)
=============================================
Step 1 - IVF: Partition vectors into K clusters (Voronoi cells)
Step 2 - PQ:  Compress each vector into sub-quantized codes

  [Full Vector 768d] --> [8 sub-vectors x 96d] --> [8 centroid IDs]
  Memory: 768 x 4B = 3KB  -->  8 x 1B = 8 bytes (375x compression)

- Best for: billion-scale, cost-sensitive, batch retrieval

DiskANN
=======
- Stores graph on SSD, not RAM
- Near-HNSW recall at fraction of memory cost
- Best for: large datasets on budget hardware
```

## Quick Reference

| Distance Metric | Formula | Range | Use Case |
|-----------------|---------|-------|----------|
| Cosine similarity | 1 - (A . B) / (|A| |B|) | [0, 2] | Text embeddings (normalized) |
| L2 (Euclidean) | sqrt(sum((a-b)^2)) | [0, inf) | Image embeddings, spatial data |
| Dot product | A . B | (-inf, inf) | Pre-normalized, max inner product |

| Platform | Index Types | Hybrid Search | Multi-tenancy | Best For |
|----------|-------------|---------------|--------------|----------|
| **pgvector** | HNSW, IVFFlat; halfvec, bit, binary quantization | Postgres full-text search + RRF | Row-level security | Postgres teams; iterative scans (0.8+) for filtered ANN |
| **Qdrant** | HNSW + TurboQuant/scalar/binary quantization | Sparse + dense via Query API prefetch, RRF/DBSF | Native | Complex filtering, multi-tenancy |
| **Pinecone** | Proprietary | Sparse-dense | Namespaces | Zero-ops, fast prototyping |
| **Weaviate** | HNSW + PQ | Native BM25 + vector | Built-in | GraphQL API, hybrid search |
| **Milvus** | HNSW, IVF, DiskANN | Built-in | Partitions | Billion-scale, GPU acceleration |
| **LanceDB** | IVF-PQ | Built-in | Tables | Embedded/local, serverless |
| **Chroma** | HNSW | Check current docs | Collections | Prototyping, local development |
| **Databricks AI Search** | Managed (standard / storage-optimized endpoints) | Hybrid (BM25 + vectors), full-text (Beta) | UC permissions + filters | Lakehouse data, Delta Sync, governance |

## Common Mistakes

### Wrong
```python
# Using L2 distance with non-normalized embeddings for text search
index = faiss.IndexFlatL2(768)
index.add(embeddings)  # embeddings not normalized -- wrong for text!

# No metadata filtering -- scanning the whole collection for a tenant-specific query
results = client.query_points("documents", query=query_vector, limit=10)
# Returns results from ALL tenants -- data leak (OWASP LLM08)
```

### Correct
```python
# Normalize embeddings, use cosine (or IP after normalization)
import numpy as np
embeddings = embeddings / np.linalg.norm(embeddings, axis=1, keepdims=True)
index = faiss.IndexFlatIP(768)  # inner product = cosine after normalization
index.add(embeddings)

# Always scope search with metadata filters (Qdrant Query API, qdrant-client 1.19)
from qdrant_client import models
results = client.query_points(
    "documents", query=query_vector, limit=10,
    query_filter=models.Filter(
        must=[models.FieldCondition(key="tenant_id", match=models.MatchValue(value="acme"))]),
).points
```

## Related

- [rag-pipelines](../concepts/rag-pipelines.md)
- [embedding-pipelines](../concepts/embedding-pipelines.md)
- [vector-db-operations](../patterns/vector-db-operations.md)
- [databricks-ai-search](../concepts/databricks-ai-search.md)

## Sources

- https://github.com/pgvector/pgvector
- https://qdrant.tech/documentation/search/search/
- https://qdrant.tech/documentation/concepts/hybrid-queries/
- https://qdrant.tech/documentation/guides/quantization/
- https://docs.databricks.com/aws/en/ai-search/ai-search/
