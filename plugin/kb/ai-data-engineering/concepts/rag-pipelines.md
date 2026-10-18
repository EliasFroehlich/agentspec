# RAG Pipelines

> **Purpose**: RAG architecture -- chunking, indexing, hybrid search, reranking, agentic RAG, context engineering
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Retrieval-Augmented Generation (RAG) grounds LLM responses in external knowledge by retrieving relevant documents before generation. RAG has evolved beyond simple "embed + retrieve + generate" into multiple architecture variants: Advanced RAG (hybrid search + reranking + query transforms), Agentic RAG (agent decides when/what to retrieve), and GraphRAG (knowledge graph augmentation). The pipeline has distinct stages -- ingestion (load, chunk, embed, store) and inference (retrieve, rerank, generate) -- each with tunable parameters. Context engineering (structured input design) has replaced naive prompt stuffing.

## The Concept

```text
RAG Architecture (Ingestion + Inference)

INGESTION PIPELINE
==================

  Documents      Chunker         Embedder       Vector DB
  (PDF, HTML,    (recursive,     (OpenAI,       (pgvector,
   Markdown)      semantic)       Cohere)        Qdrant)

  [Raw Docs] --> [Chunks] --> [Vectors] --> [Index + Metadata]
       |              |             |                |
       v              v             v                v
  Parse/clean    512-1024 tok   768-3072 dim    HNSW / IVF-PQ
  + metadata     + 10-20%       batch embed     + payload filter
  extraction     overlap

INFERENCE PIPELINE
==================

  User Query --> Embed Query --> Hybrid Search --> Rerank --> LLM
       |              |              |                |         |
       v              v              v                v         v
  Query          Dense +        Top-K (20-50)    Top-N (3-5)  Generate
  expansion      sparse BM25   candidates        reranked     with context
```

## Quick Reference

| Chunking Strategy | Best For | Typical Size | Overlap |
|-------------------|----------|-------------|---------|
| Fixed-size | Uniform docs (logs, code) | 512 tokens | 10-20% |
| Recursive | Structured text (docs, articles) | 512-1024 tokens | 10-20% |
| Semantic | Mixed content (research papers) | Variable | N/A (boundary-based) |
| Document | Short docs (emails, tickets) | Full doc | N/A |

| Search Strategy | Precision | Recall | Latency | When to Use |
|-----------------|-----------|--------|---------|-------------|
| Dense only | High | Medium | Low | Semantic similarity tasks |
| Sparse (BM25) | Medium | High | Low | Keyword-heavy queries |
| Hybrid (dense + sparse) | High | High | Medium | Production RAG systems |
| Hybrid + rerank | Highest | High | Higher | Quality-critical applications |

| Managed option (Databricks) | What it replaces |
|-----------------------------|------------------|
| `ai_parse_document` + `ai_prep_search` | Loaders + splitter; adds document context to `chunk_to_embed` |
| AI Search Delta Sync index | Embedding job + vector DB sync |
| `query_type="HYBRID"` / `FULL_TEXT` | Hand-rolled BM25 + fusion |
| MLflow `RetrievalGroundedness` / `RetrievalRelevance` | Custom faithfulness judges |

## Common Mistakes

### Wrong

```python
# Chunking without overlap loses context at boundaries
chunks = text_splitter.split_text(
    document,
    chunk_size=512,
    chunk_overlap=0  # context lost between chunks!
)

# Storing vectors without metadata -- impossible to filter or cite
vector_store.add(embeddings=vectors)  # no metadata!
```

### Correct

```python
from datetime import datetime, timezone
from langchain_text_splitters import RecursiveCharacterTextSplitter   # LangChain 1.x package

# Overlap preserves cross-boundary context
chunks = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,  # ~12% overlap
    separators=["\n\n", "\n", ". ", " "]
).split_documents(documents)

# Always store metadata for filtering and citations
vector_store.add(
    embeddings=vectors,
    metadatas=[{
        "source": doc.source,
        "page": doc.page,
        "section": doc.section,
        "ingested_at": datetime.now(timezone.utc).isoformat()
    } for doc in chunks]
)
```

## Related

- [vector-databases](../concepts/vector-databases.md)
- [embedding-pipelines](../concepts/embedding-pipelines.md)
- [rag-pipeline-implementation](../patterns/rag-pipeline-implementation.md)
- [databricks-native-rag](../patterns/databricks-native-rag.md)

## Sources

- https://docs.langchain.com/oss/python/releases/langchain-v1
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_prep_search
- https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://mlflow.org/docs/latest/genai/eval-monitor/scorers/llm-judge/rag
