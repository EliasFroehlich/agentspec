# Databricks-native RAG

> **Purpose**: End-to-end RAG on Databricks with managed pieces — `ai_parse_document` → `ai_prep_search` → AI Search Delta Sync index → hybrid retrieval → generation → MLflow evaluation
> **MCP Validated**: 2026-10-17

## When to Use

- Documents (PDF, Office, images) land in Unity Catalog volumes and answers must respect UC governance
- You want parsing, chunking, embedding and sync managed instead of hand-written LangChain loaders
- Low-code alternative: a Genie Agent over the same documents and tables; this pattern is for custom apps and agents

## Implementation

```sql
-- Step 1: parse + chunk (ai_parse_document DBR 17.3+; ai_prep_search Beta, DBR 18.2+ / serverless env v3+)
-- Persist the output: each call is billed inference, so never recompute it per query.
CREATE OR REPLACE TABLE main.docs.chunks
TBLPROPERTIES (delta.enableChangeDataFeed = true) AS        -- standard endpoints need CDF
WITH prepped AS (
  SELECT path AS source_path, ai_prep_search(ai_parse_document(content)) AS prep
  FROM read_files('/Volumes/main/docs/landing/', format => 'binaryFile')
)
SELECT
  variant_get(chunk, '$.chunk_id',          'STRING') AS chunk_id,
  variant_get(chunk, '$.chunk_to_retrieve', 'STRING') AS chunk_to_retrieve,   -- raw text for the LLM
  variant_get(chunk, '$.chunk_to_embed',    'STRING') AS chunk_to_embed,      -- text + doc context for embedding
  source_path
FROM prepped
LATERAL VIEW explode(variant_get(prep, '$.document.contents', 'ARRAY<VARIANT>')) c AS chunk;
```

```python
# Step 2: AI Search index with managed embeddings (databricks-sdk)
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
w.vector_search_indexes.create_index(
    name="main.docs.chunks_index",
    endpoint_name="docs-endpoint",                  # STANDARD or STORAGE_OPTIMIZED endpoint
    primary_key="chunk_id",
    index_type="DELTA_SYNC",
    delta_sync_index_spec={
        "source_table": "main.docs.chunks",
        "embedding_source_columns": [{"name": "chunk_to_embed",
                                      "embedding_model_endpoint_name": "databricks-gte-large-en"}],
        "pipeline_type": "TRIGGERED",
    },
)
# After each load of main.docs.chunks: w.vector_search_indexes.sync_index(index_name="main.docs.chunks_index")
```

```python
# Step 3: retrieve (hybrid) + generate, traced for evaluation
import anthropic
import mlflow
from mlflow.entities import Document

mlflow.anthropic.autolog()
claude = anthropic.Anthropic()          # or call system.ai.claude-sonnet-5-5 through Unity Gateway

@mlflow.trace(span_type="RETRIEVER")
def retrieve(question: str, k: int = 5) -> list[Document]:
    res = w.vector_search_indexes.query_index(
        index_name="main.docs.chunks_index",
        columns=["chunk_id", "chunk_to_retrieve", "source_path"],
        query_text=question, query_type="HYBRID", num_results=k,
    )
    return [Document(id=r[0], page_content=r[1], metadata={"doc_uri": r[2]})
            for r in res.result.data_array]

@mlflow.trace
def answer(question: str) -> str:
    docs = retrieve(question)
    if not docs:
        return "I don't have information on that in the indexed documents."
    sources = "\n".join(f"<source id='{d.id}' uri='{d.metadata['doc_uri']}'>\n{d.page_content}\n</source>"
                        for d in docs)
    resp = claude.messages.create(
        model="claude-sonnet-5-5", max_tokens=16000,
        system="Answer only from the sources and cite source ids. Say so when the sources lack the answer.",
        messages=[{"role": "user", "content": f"<sources>\n{sources}\n</sources>\n\n{question}"}],
    )
    return "".join(b.text for b in resp.content if b.type == "text")
```

```python
# Step 4: evaluate with MLflow 3 (RAG judges need the RETRIEVER span above)
from mlflow.genai.scorers import RelevanceToQuery, RetrievalGroundedness, RetrievalRelevance

results = mlflow.genai.evaluate(
    data=[{"inputs": {"question": "What is the data retention period?"},
           "expectations": {"expected_facts": ["7 years"]}}],
    predict_fn=answer,
    scorers=[RetrievalRelevance(), RetrievalGroundedness(), RelevanceToQuery()],
)
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Endpoint type | `STANDARD` | `STORAGE_OPTIMIZED` above hundreds of millions of vectors or for cheaper scale |
| Embedding model | `databricks-gte-large-en` | `databricks-qwen3-embedding-0-6b` (Public Preview) for multilingual text |
| `pipeline_type` | `TRIGGERED` | `CONTINUOUS` for near-real-time sync (higher cost) |
| `query_type` | `HYBRID` | `ANN` for purely semantic queries; `FULL_TEXT` (Beta) for keyword-only |
| `num_results` | `5` | Retrieve more and rerank if answers miss context |
| Generation model | `claude-sonnet-5-5` | Via Anthropic API or Unity Gateway (`system.ai.claude-sonnet-5-5`) |

## Example Usage

```python
print(answer("Who approves exceptions to the retention policy?"))
```

```sql
-- Batch alternative entirely in SQL (vector_search is Public Preview)
SELECT q.question, s.chunk_to_retrieve
FROM main.docs.questions q,
     LATERAL (SELECT * FROM vector_search(index => 'main.docs.chunks_index',
                                          query_text => q.question, query_type => 'HYBRID',
                                          num_results => 3)) s;
```

`ai_search` (Beta) returns ranked documents plus a grounded answer in one SQL call; evaluate it against this pipeline on
your own questions before choosing.

## See Also

- [Databricks AI Search](../concepts/databricks-ai-search.md)
- [RAG Pipeline Implementation](rag-pipeline-implementation.md)
- [RAG Pipelines Concept](../concepts/rag-pipelines.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_parse_document
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_prep_search
- https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_search
- https://docs.databricks.com/aws/en/large-language-models/ai-functions
- https://mlflow.org/docs/latest/genai/eval-monitor/scorers/llm-judge/rag
- https://mlflow.org/docs/latest/genai/concepts/span
