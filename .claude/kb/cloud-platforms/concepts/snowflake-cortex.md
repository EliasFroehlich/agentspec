# Snowflake Cortex & AI Platform

> **Purpose**: Snowflake AI and data platform in 2026 — Cortex AI Functions (AI_*), legacy SNOWFLAKE.CORTEX.* retirement, Cortex Agents, warehouses (Gen2, Adaptive, interactive), dynamic tables, Snowpipe Streaming, Openflow and Iceberg
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Snowflake exposes LLMs as SQL functions (**Cortex AI Functions**, the `AI_*` family) running inside the
Snowflake service perimeter, plus agentic products (**Cortex Agents**, **Snowflake Intelligence**).
Compute moved to **Gen2** standard warehouses by default and to **Adaptive Warehouses** (no manual sizing).
Dynamic tables are the declarative pipeline layer; Snowpipe Streaming's high-performance architecture
and **Openflow** cover ingestion; Iceberg tables cover open formats.

## The Concept

```sql
-- Model-agnostic task functions (no model argument)
SELECT ticket_id,
       AI_CLASSIFY(description, ['billing', 'technical', 'account', 'other']):labels[0]::STRING AS category,
       AI_SENTIMENT(description)                                            AS sentiment
FROM support.tickets
WHERE AI_FILTER(PROMPT('Is this ticket about a refund? {0}', description));

-- General completion with a chosen model, hyperparameters and structured output
SELECT AI_COMPLETE(
         model            => 'claude-sonnet-5',
         prompt           => CONCAT('Summarize in one sentence: ', description),
         model_parameters => {'temperature': 0, 'max_tokens': 200},
         return_error_details => TRUE) AS summary
FROM support.tickets;

-- Embeddings: dimension is set by the model; match VECTOR(FLOAT, n)
SELECT doc_id, AI_EMBED('snowflake-arctic-embed-l-v2.0', content) AS embedding
FROM knowledge.documents;
```

## Cortex AI Functions

| Function | Purpose |
|----------|---------|
| `AI_COMPLETE` | General completion over text, images or a prompt object (GA 2025-11-21; replaces `COMPLETE`) |
| `AI_CLASSIFY` | Text/image classification into your labels (multi-label supported; read `:labels[0]`) |
| `AI_FILTER` | Boolean predicate for `SELECT`, `WHERE`, `JOIN ... ON` |
| `AI_AGG` / `AI_SUMMARIZE_AGG` | Aggregate insights / summary across rows, not limited by the context window |
| `AI_EMBED` / `AI_MULTI_EMBED` | Single-vector embeddings / multimodal embeddings with segment metadata |
| `AI_SIMILARITY` | Embedding similarity of two inputs |
| `AI_EXTRACT` | Structured extraction from text, images and documents (replaces `EXTRACT_ANSWER`) |
| `AI_SENTIMENT` | Overall and per-entity sentiment (replaces `SENTIMENT`, `ENTITY_SENTIMENT`) |
| `AI_SUMMARIZE` | Summaries of text, images, documents |
| `AI_TRANSCRIBE` | Audio/video transcription with timestamps and speakers |
| `AI_PARSE_DOCUMENT` | OCR or LAYOUT extraction from staged documents |
| `AI_REDACT` | PII redaction |
| `AI_TRANSLATE` | Translation |
| Helpers | `AI_COUNT_TOKENS`, `PROMPT`, `TO_FILE` |

Per-function GA/preview status is on the AI Functions page (functions not marked preview are GA).
Since 2025-09-11 the functions work in the `SELECT` clause of **incremental** dynamic tables.
Access needs the `USE AI FUNCTIONS` account privilege plus `CORTEX_USER` or `AI_FUNCTIONS_USER`.

**Models (as of 2026-10-17):** `AI_COMPLETE` offers Anthropic (`claude-opus-5`, `claude-sonnet-5`,
`claude-haiku-4-5`...), OpenAI (`openai-gpt-5.4`...), Google (`gemini-3.8-flash`...), Meta and Mistral
models; many need cross-region inference and some are preview. `mistral-large2`, `openai-gpt-4.1` and
`claude-4-sonnet` are **[legacy]**: only accounts that already used them can call them until end-of-life.
Run `SHOW CORTEX BASE MODELS` and check the regional availability page before pinning a model.

**Legacy functions:** `SNOWFLAKE.CORTEX.COMPLETE`, `TRY_COMPLETE`, `CLASSIFY_TEXT`, `SENTIMENT`,
`EMBED_TEXT_768/1024`, etc. map to `AI_*` replacements. Behavior change bundle **2026_08** (BCR-2432,
pending) makes them fail in accounts with no use between 2026-07-09 and 2026-10-06; outputs differ
(objects instead of strings/floats), so migrate per the official guide.

## Platform Map

| Area | Current state |
|------|---------------|
| Cortex Agents | GA 2025-11-04; orchestrate Cortex Analyst (structured) and Cortex Search (unstructured) |
| Snowflake Intelligence | Natural-language interface for business users (GA 2025-11-04) |
| Gen2 warehouses | GA 2025-05-05; default `GENERATION = '2'` for new standard warehouses where available |
| Adaptive Warehouses | GA 2026-06-16 on AWS, now in select AWS/Azure/GCP regions; query-based billing; `MAX_QUERY_PERFORMANCE_LEVEL`, `QUERY_THROUGHPUT_MULTIPLIER` |
| Interactive tables/warehouses | GA 2025-12-11 (select AWS regions); low-latency, high-concurrency serving |
| Dynamic tables | `TARGET_LAG` or `DOWNSTREAM`; `REFRESH_MODE = AUTO / FULL / INCREMENTAL / ADAPTIVE / CUSTOM_INCREMENTAL`; `SCHEDULER = ENABLE / DISABLE`; no cron clause |
| Snowpipe Streaming (high-performance) | GA on AWS 2025-09-23; up to 10 GB/s per table, ingest-to-query typically < 10 s, server-side `PIPE`, Java/Python SDKs (Rust core) + REST |
| Openflow | GA 2025-05-30; managed ingestion connectors (some still preview) |
| Iceberg | Snowflake-managed Iceberg tables in your storage; new customers use **Horizon Catalog** for multi-engine Iceberg (Snowflake Open Catalog, the managed Apache Polaris, accepts no new sign-ups) |

## Trade-offs

- AI functions bill per token; prompt size dominates cost. `AI_COMPLETE` results are cached for identical calls.
- Dynamic tables: for a fixed clock schedule set `SCHEDULER = DISABLE` and run `ALTER DYNAMIC TABLE ... REFRESH` from a task with `USING CRON`.
- Cortex AI Functions in incremental dynamic tables are supported only in the `SELECT` clause.
- Adaptive Warehouses need Enterprise Edition or higher; mostly-HTAP workloads stay on Gen2.

## Common Mistakes

### Wrong

```sql
-- Legacy namespace + legacy model; fails in inactive accounts once BCR 2026_08 is enabled
SELECT SNOWFLAKE.CORTEX.COMPLETE('mistral-large2', prompt) FROM t;
SELECT SNOWFLAKE.CORTEX.CLASSIFY_TEXT(review, ['refund','shipping']):label FROM t;
```

### Correct

```sql
SELECT AI_COMPLETE('claude-sonnet-5', prompt) FROM t;
SELECT AI_CLASSIFY(review, ['refund','shipping']):labels[0] FROM t;
```

## See Also

- [Snowflake Patterns](../patterns/snowflake-patterns.md) — dynamic tables, tasks, warehouses, AI in SQL
- [Cross-Platform Patterns](cross-platform-patterns.md) — dialect differences
- [Cost Optimization](../patterns/cost-optimization.md) — credits, Gen2/Adaptive sizing

## Sources

- https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql
- https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability
- https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-migrate-legacy-functions
- https://docs.snowflake.com/en/release-notes/bcr-bundles/2026_08/bcr-2432
- https://docs.snowflake.com/en/sql-reference/functions/ai_complete
- https://docs.snowflake.com/ja/release-notes/2025/other/2025-11-21-ai-complete-ga
- https://docs.snowflake.com/en/release-notes/2025/other/2025-09-11-dynamic-tables-cortex-aisql-support
- https://docs.snowflake.com/en/release-notes/2025/other/2025-11-04-cortex-agents
- https://www.snowflake.com/en/news/press-releases/snowflake-intelligence-brings-agentic-AI-to-the-enterprise/
- https://docs.snowflake.com/en/user-guide/warehouses-gen2
- https://docs.snowflake.com/en/user-guide/warehouses-adaptive
- https://docs.snowflake.com/en/release-notes/2025/other/2025-12-11-interactive-tables-ga
- https://docs.snowflake.com/en/release-notes/2025/other/2025-09-23-snowpipe-streaming-high-performance-architecture
- https://docs.snowflake.com/en/release-notes/2025/other/2025-05-30-openflow
- https://docs.snowflake.com/en/sql-reference/sql/create-dynamic-table
- https://docs.snowflake.com/en/user-guide/opencatalog/overview
