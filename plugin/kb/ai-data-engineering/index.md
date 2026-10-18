# AI Data Engineering Knowledge Base

> **Purpose**: AI data engineering -- RAG pipelines, vector stores (including Databricks AI Search), embeddings, feature stores (Feast and Databricks UC), Genie text-to-SQL, LLMOps and observability
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/rag-pipelines.md](concepts/rag-pipelines.md) | Chunking, indexing, hybrid search, reranking, managed Databricks options |
| [concepts/vector-databases.md](concepts/vector-databases.md) | HNSW, IVF, distance metrics, platform comparison |
| [concepts/databricks-ai-search.md](concepts/databricks-ai-search.md) | AI Search (formerly Vector Search): endpoints, index types, ANN/hybrid/full-text, SDK and SQL |
| [concepts/feature-stores.md](concepts/feature-stores.md) | Online/offline stores, Feast, Databricks Feature Store in UC |
| [concepts/embedding-pipelines.md](concepts/embedding-pipelines.md) | Current embedding models, `google-genai`, Matryoshka, versioning and drift |
| [concepts/llmops-patterns.md](concepts/llmops-patterns.md) | Versioning, token counting, cost, MLflow 3 tracing, Langfuse v5 |
| [concepts/genie-agents.md](concepts/genie-agents.md) | Genie Agents (formerly Genie Spaces): Conversation API, curation, limits |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/rag-pipeline-implementation.md](patterns/rag-pipeline-implementation.md) | RAG with LangChain 1.x packages + RAGAS evaluation |
| [patterns/databricks-native-rag.md](patterns/databricks-native-rag.md) | `ai_parse_document` → `ai_prep_search` → AI Search → MLflow evaluation |
| [patterns/vector-db-operations.md](patterns/vector-db-operations.md) | pgvector 0.8, Qdrant 1.19 Query API and hybrid, AI Search queries |
| [patterns/feature-engineering.md](patterns/feature-engineering.md) | Feast definitions, on-demand views, materialization, serving |
| [patterns/uc-feature-engineering.md](patterns/uc-feature-engineering.md) | Databricks Feature Engineering in Unity Catalog |
| [patterns/text-to-sql.md](patterns/text-to-sql.md) | Genie Agents first; guarded single-dialect pipeline elsewhere |
| [patterns/training-data-pipelines.md](patterns/training-data-pipelines.md) | DVC, labeling, bias detection, reproducibility |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| ai-data-engineer | All files | RAG, embeddings, feature stores, text-to-SQL |
| qdrant-specialist | concepts/vector-databases.md, patterns/vector-db-operations.md | Qdrant collections and search |
| data-quality-analyst | concepts/rag-pipelines.md | Training data quality |
