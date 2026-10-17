# AI Data Engineering Quick Reference

> Fast lookup tables. For code examples, see linked files.
> **MCP Validated**: 2026-10-17 — versions and models as of 2026-10-17.

## Vector Store Comparison

| Feature | pgvector 0.8.7 | Qdrant 1.19 | Databricks AI Search | Pinecone | Weaviate | Milvus |
|---------|----------------|-------------|----------------------|----------|----------|--------|
| Type | Postgres extension | Standalone (Rust) | Managed, UC-governed | Managed | Standalone | Standalone |
| Hybrid search | Postgres full-text + RRF in app | Sparse + dense prefetch, RRF/DBSF | `HYBRID` (BM25 + ANN), `FULL_TEXT` (Beta) | Sparse-dense | Native BM25 | Built-in |
| Filtering | SQL `WHERE` + iterative scans | Payload filters + indexes | Filters (dict or SQL-like by endpoint) | Filter API | Filters | Expressions |
| Compression | `halfvec`, binary quantization | TurboQuant, scalar, binary | Managed | Managed | PQ | Several |
| Best for | Postgres-centric apps | Complex filtering, self-hosting | Lakehouse data, Delta Sync | Zero-ops | Hybrid search | Billion-scale |

Chroma and LanceDB are embedded options for local development and prototypes.

## Feature Store Comparison

| Feature | Feast 0.67 | Databricks Feature Store (UC) | Tecton | Hopsworks | SageMaker FS |
|---------|------------|-------------------------------|--------|-----------|--------------|
| License | OSS (Apache 2.0) | Databricks | Commercial | OSS + Enterprise | AWS |
| Authoring | Python definitions | Feature tables (GA) or Feature Views (declarative) | Declarative | Platform | API |
| Online serving | Redis, DynamoDB, others | Online Feature Store (Lakebase) | Managed | RonDB | DynamoDB |
| Vector features | Alpha (Milvus, pgvector, Qdrant, …) | Use AI Search | — | Native | — |

## Embedding Models (as of 2026-10-17)

| Model | Provider | Dims | Context | Notes |
|-------|----------|------|---------|-------|
| `voyage-4-large` / `voyage-4` / `voyage-4-lite` | Voyage AI | 1024 (256–2048) | 32K | `voyage-4-nano` open-weight |
| `embed-v5.0-pro` / `-fast` | Cohere | 256–2048 | 128K | Text + images + PDFs |
| `gemini-embedding-2` | Google | 3072 (MRL; 768/1536 recommended) | 8,192 | Multimodal; `google-genai` SDK |
| `text-embedding-3-large` / `-small` | OpenAI | `dimensions` param | 8,192 | Still current |
| `databricks-qwen3-embedding-0-6b` | Databricks FMAPI | ≤ 1024 | ~32K | Public Preview |
| BGE-M3, Nomic embed | Open source | 1024 / 768 | 8K | Self-hosted |

## RAG Pipeline Checklist

| Step | Component | Key decision |
|------|-----------|--------------|
| 1. Load / parse | Loaders or `ai_parse_document` | Layout-aware parsing for PDFs and tables |
| 2. Chunk | `langchain-text-splitters` or `ai_prep_search` | Recursive default; context-enriched chunks |
| 3. Embed | Embedding model | Pin model + dimensions; managed embeddings on AI Search |
| 4. Store | Vector store | pgvector / Qdrant / AI Search |
| 5. Retrieve | Hybrid (dense + keyword) | Fusion (RRF) or platform hybrid |
| 6. Rerank | `rerank-v4.0-pro` / `-fast` (Cohere), `rerank-2.5` (Voyage) | Retrieve 20–50, keep 3–5 |
| 7. Generate | `claude-sonnet-5-5`, `gpt-6.1-sol`, `gemini-3.8-flash` | Cite sources; structured output when needed |
| 8. Evaluate | MLflow 3 scorers, RAGAS 0.4 | Groundedness, relevance, context recall |

## LLMOps and Observability

| Tool | Type | Strength |
|------|------|----------|
| MLflow 3 | OSS / Databricks managed | Tracing (OTel), `mlflow.genai.evaluate`, Prompt Registry, monitoring |
| Langfuse 5.x | OSS / cloud | OTel-based tracing, prompts, scores |
| LangSmith | Commercial | LangChain/LangGraph native |
| Arize Phoenix | OSS | Traces, embeddings, evals |
| Unity Gateway | Databricks | Usage, budgets, rate limits, PII masking for models/MCP |

## Databricks Text-to-SQL

| Need | Use |
|------|-----|
| NL questions over UC tables / metric views | Genie Agent + Conversation API (`w.genie.start_conversation`) |
| From an external agent | Genie One MCP server |
| Outside Databricks | Guarded pipeline in `patterns/text-to-sql.md` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Mix embedding models or dimensions in one index | One index per model+dimension; record both per vector |
| Count Claude tokens with tiktoken | `client.messages.count_tokens` |
| Import `langchain.text_splitter` / `langchain.chains` (1.x) | `langchain_text_splitters`; `create_agent` or `langchain_classic` |
| Use the deprecated `google.generativeai` SDK | `from google import genai` (`google-genai`) |
| Write new Qdrant code against `search` / `/points/search` | Query API: `query_points` / `/points/query` (v1.10+) |
| Rebuild text-to-SQL over Unity Catalog | Curate a Genie Agent |
| Skip evaluation | Fixed dataset + MLflow/RAGAS from day one |

## Sources

- https://github.com/pgvector/pgvector · https://qdrant.tech/documentation/search/search/ · https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://docs.feast.dev/reference/alpha-vector-database · https://docs.databricks.com/aws/en/machine-learning/feature-store/
- https://docs.voyageai.com/docs/embeddings · https://docs.cohere.com/docs/cohere-embed · https://ai.google.dev/gemini-api/docs/embeddings · https://docs.cohere.com/docs/rerank
