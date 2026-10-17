# BigQuery AI & ML Platform

> **Purpose**: BigQuery generative AI functions (AI.GENERATE, AI.EMBED, managed AI.IF/AI.CLASSIFY/AI.SCORE/AI.AGG), Gemini model IDs, BQML and Iceberg managed tables
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

BigQuery calls Gemini directly from SQL. Scalar functions (`AI.GENERATE`, `AI.EMBED`, `AI.SIMILARITY`)
need **no remote model** and, without `connection_id`, run with the caller's end-user credentials.
Table-valued functions (`AI.GENERATE_TEXT`, `AI.GENERATE_TABLE`, `AI.GENERATE_EMBEDDING`) still take a
remote model (`MODEL dataset.model`). BQML keeps in-SQL training (`CREATE MODEL`, `ML.PREDICT`).
Renames: Vertex AI is now **Gemini Enterprise Agent Platform** ("Agent Platform"); BigLake is now
**Google Cloud Lakehouse** and BigLake metastore the **Lakehouse runtime catalog** (both 2026-04-22).

## The Concept

```sql
-- Scalar: no CREATE MODEL. Default endpoint is gemini-3.5-flash-lite.
SELECT
  ticket_id,
  AI.GENERATE(
    CONCAT('Classify this support ticket: ', ticket_text),
    endpoint      => 'gemini-3.5-flash',
    model_params  => JSON '{"generation_config": {"thinking_config": {"thinking_level": "LOW"}}}',
    output_schema => 'category STRING, urgency STRING, needs_human BOOL'
  ).* EXCEPT (full_response, status)
FROM mydataset.support_tickets;

-- Managed functions return a scalar (no status column)
SELECT title,
       AI.CLASSIFY(body, categories => ['tech', 'sport', 'business', 'other']) AS topic
FROM mydataset.articles
WHERE AI.IF(('This article is about a natural disaster: ', body));

-- Embeddings and similarity (scalar)
SELECT AI.EMBED('A piece of text', endpoint => 'gemini-embedding-001').result AS vec,
       AI.SIMILARITY('happy', 'glad', endpoint => 'text-embedding-005') AS cosine;
```

`AI.GENERATE([prompt =>] prompt, [endpoint =>], [model_params =>], [output_schema =>],
[connection_id =>], [request_type =>])` returns `STRUCT<result STRING, full_response JSON, status STRING>`;
with `output_schema`, your fields replace `result`. `prompt` is a STRING or a STRUCT mixing strings and
`ObjectRef` values (images, PDFs, audio, video). `request_type` = `SHARED` | `DEDICATED` | `UNSPECIFIED`.

## Function Catalog

| Function | Kind | Returns | Status |
|----------|------|---------|--------|
| `AI.GENERATE` | general-purpose, scalar | STRUCT (result/full_response/status) | GA 2025-11-18 |
| `AI.GENERATE_BOOL` / `_INT` / `_DOUBLE` | general-purpose, scalar | STRUCT with one typed field | Listed in docs |
| `AI.GENERATE_TEXT` | table-valued, remote model | table | GA 2025-11-18 (preferred over `ML.GENERATE_TEXT`) |
| `AI.GENERATE_TABLE` | table-valued, remote model | table with your schema | GA 2025-11-18 |
| `AI.GENERATE_EMBEDDING` | table-valued, remote model | table + embedding column | GA 2025-11-18 (preferred over `ML.GENERATE_EMBEDDING`) |
| `AI.EMBED` | scalar | STRUCT<result ARRAY<FLOAT64>, status> | GA 2026-03-23 |
| `AI.SIMILARITY` | scalar | FLOAT64 cosine similarity | GA 2026-03-23 |
| `AI.IF` | managed | BOOL | GA 2026-04-13 (optimized mode: Preview) |
| `AI.SCORE` | managed | FLOAT64 | GA 2026-04-13 |
| `AI.CLASSIFY` | managed | STRING (array with `output_mode`) | GA 2026-04-13 (optimized mode: Preview) |
| `AI.AGG` | managed, aggregate | STRING (or schema via `output_schema`) | GA 2026-10-13 |
| `AI.COUNT_TOKENS` | utility | token estimate | Listed in docs |

`ML.GENERATE_TEXT` / `ML.GENERATE_EMBEDDING` still work; Google recommends the `AI.*` versions for new queries.

## Models (as of 2026-10-17)

| Use | Model IDs | Notes |
|-----|-----------|-------|
| Text / multimodal | `gemini-4-argon`, `gemini-3.8-flash`, `gemini-3.5-flash`, `gemini-3.5-flash-lite`, `gemini-3.1-flash-lite` | `gemini-4-argon` in BigQuery GA 2026-10-13 |
| Embeddings | `gemini-embedding-2` (GA in BigQuery 2026-10-05; multi-region/global endpoints only), `gemini-embedding-001`, `text-embedding-005` | |
| In-slot embeddings | `model => 'embeddinggemma-300m'` (Preview) | Runs on BigQuery slots, no Agent Platform charge |
| Retired | `gemini-2.0-flash`, `gemini-2.0-flash-lite` (2026-06-01), all 1.x and PaLM models | Calls fail |
| Retiring | `gemini-2.5-pro`, `gemini-2.5-flash`, `gemini-2.5-flash-lite` on 2026-10-20 | Move to 3.x |

Always check the model versions page before pinning an ID.

## Lakehouse and Platform

| Capability | Current state |
|------------|---------------|
| **Apache Iceberg managed tables** (formerly BigLake tables for Apache Iceberg in BigQuery; GA 2025-06-03) | Data in your Cloud Storage bucket; DML, Storage Write API streaming, schema evolution, time travel; partitioning, multi-statement transactions and advanced runtime GA 2026-07-13 |
| Iceberg external tables | Read Iceberg written by other engines; Iceberg v3 (binary deletion vectors) in Preview since 2026-04-15 |
| Lakehouse runtime catalog | Apache Iceberg REST catalog for open-source engines (credential vending supported) |
| Multimodal | `ObjectRef` columns (`OBJ.MAKE_REF`, `OBJ.GET_READ_URL`) passed straight into AI functions |
| Conversational analytics | GA 2026-06-23 (Gemini in BigQuery agents over your data) |
| BQML | `CREATE MODEL` (boosted trees, ARIMA_PLUS, imported ONNX/TF), `ML.EVALUATE`, `ML.PREDICT`, `ML.FORECAST` |
| Dataform | SQLX + `ref()` transformations native to BigQuery |

## Common Mistakes

### Wrong

```sql
-- Legacy pattern: AI.GENERATE does NOT take a MODEL argument, and gemini-pro/2.0 are retired
CREATE OR REPLACE MODEL mydataset.gemini_pro
  REMOTE WITH CONNECTION `us.my_conn` OPTIONS (ENDPOINT = 'gemini-pro');
SELECT AI.GENERATE(MODEL `mydataset.gemini_pro`, prompt).result FROM mydataset.t;
```

### Correct

```sql
-- Scalar call with an explicit current endpoint; connection optional (EUC by default)
SELECT AI.GENERATE(prompt, endpoint => 'gemini-3.8-flash').result FROM mydataset.t;
-- Need a partner/open model (Claude, Llama, Mistral)? Use AI.GENERATE_TEXT with a remote model.
```

## Related

- [BigQuery Patterns](../patterns/bigquery-patterns.md) — remote models, AI.GENERATE_TABLE, Iceberg DDL, BQML
- [Cross-Platform Patterns](cross-platform-patterns.md) — dialect differences
- [Cost Optimization](../patterns/cost-optimization.md) — on-demand vs editions

## Sources

- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate
- https://docs.cloud.google.com/bigquery/docs/generative-ai-overview
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-embed
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-similarity
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-if
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-classify
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-agg
- https://docs.cloud.google.com/bigquery/docs/choose-text-generation-function
- https://docs.cloud.google.com/bigquery/docs/release-notes
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-versions
- https://docs.cloud.google.com/gemini-enterprise-agent-platform/vertex-ai-name-changes
- https://docs.cloud.google.com/bigquery/docs/iceberg-tables
