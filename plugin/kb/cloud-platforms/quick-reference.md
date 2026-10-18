# Cloud Platforms Quick Reference

> **Purpose**: Fast lookup for Snowflake, Databricks and BigQuery — architecture, AI functions, prices and 2025–2026 changes
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

> Fast lookup tables. For code examples, see linked files.

## Platform Comparison

| Feature | Snowflake | Databricks | BigQuery |
|---------|-----------|------------|----------|
| Architecture | Shared storage, isolated warehouses | Lakehouse on Delta/Iceberg + Spark/Photon | Serverless, slot-based |
| Open table formats | Snowflake-managed Iceberg; Horizon Catalog (Open Catalog: no new sign-ups) | Delta + UC managed Iceberg (GA 2026-05-21), Iceberg REST catalog | Apache Iceberg managed tables + Lakehouse runtime catalog (Iceberg REST) |
| SQL dialect | Snowflake SQL | Databricks SQL / Spark SQL (ANSI mode default, DBR 17+) | GoogleSQL |
| Python | Snowpark | PySpark, Spark Connect | BigQuery DataFrames |
| Declarative pipelines | Dynamic tables | Lakeflow Spark Declarative Pipelines | Dataform, scheduled queries |
| Streaming ingest | Snowpipe Streaming (high-performance) | Structured Streaming, Lakeflow Connect, Zerobus | Storage Write API (gRPC), Pub/Sub BigQuery subscriptions |
| Serverless | Warehouses are managed; Adaptive Warehouses | Serverless jobs, pipelines, notebooks, SQL warehouses | Always |
| Cost unit | Credits (per second) | DBUs (per second) | TiB scanned or slot-hours |

## AI in SQL

| Capability | Snowflake | Databricks | BigQuery |
|-----------|-----------|------------|----------|
| General LLM call | `AI_COMPLETE` | `ai_query` | `AI.GENERATE` (scalar), `AI.GENERATE_TEXT` (TVF) |
| Classify / filter | `AI_CLASSIFY`, `AI_FILTER` | `ai_classify` | `AI.CLASSIFY`, `AI.IF` |
| Score / aggregate | `AI_AGG`, `AI_SUMMARIZE_AGG` | `ai_summarize` | `AI.SCORE`, `AI.AGG` |
| Embeddings / similarity | `AI_EMBED`, `AI_SIMILARITY` | `ai_similarity`, AI Search | `AI.EMBED`, `AI.SIMILARITY` |
| Documents | `AI_PARSE_DOCUMENT`, `AI_EXTRACT` | `ai_parse_document`, `ai_extract` | ObjectRef + `AI.GENERATE` |
| Agents / NL BI | Cortex Agents, Snowflake Intelligence | Agent Bricks, Genie One / Genie Agents | Conversational analytics (GA 2026-06-23) |
| Model catalog | Regional availability page | Foundation Model APIs supported-models page | Gemini model versions page |

## Prices (as of 2026-10-17 — always re-check)

| Platform | Compute | Storage |
|----------|---------|---------|
| Snowflake | $2 / $3 / $4 per credit (Standard / Enterprise / Business Critical, AWS US East, on-demand) | $23/TB-month on-demand |
| Databricks | From $0.15/DBU (Data Engineering), $0.22 (SQL), $0.40 (Interactive), $0.07 (AI) | Cloud object storage |
| BigQuery | $6.25/TiB on-demand (1 TiB/month free); $0.04 / $0.06 / $0.10 per slot-hour (Standard / Enterprise / Enterprise Plus) | ≈ $0.023/GiB-month active logical |

Sources: [Snowflake](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf) ·
[Databricks](https://www.databricks.com/product/pricing) · [BigQuery](https://cloud.google.com/bigquery/pricing)

## What Changed (2025–2026)

| Platform | Change | Date |
|----------|--------|------|
| Snowflake | Gen2 standard warehouses GA (now the default) | 2025-05-05 |
| Snowflake | Cortex Agents GA | 2025-11-04 |
| Snowflake | `AI_COMPLETE` GA (legacy `SNOWFLAKE.CORTEX.*` removal in inactive accounts: BCR 2026_08) | 2025-11-21 |
| Snowflake | Interactive tables and warehouses GA | 2025-12-11 |
| Snowflake | Adaptive Compute GA | 2026-06-16 |
| Databricks | DLT → Lakeflow pipelines; `from pyspark import pipelines as dp` | 2025–2026 |
| Databricks | Lakebase GA (AWS; Azure 2026-03-02) | 2026-01-22 |
| Databricks | Vector Search renamed AI Search | 2026-06-01 |
| Databricks | DBR 18 LTS (Spark 4.1) / DBR 19 Beta (Spark 4.2) | 2026-06-10 / 06-15 |
| Databricks | Lakeflow Designer GA (Preview 2026-04-22) | 2026-06-16 |
| Databricks | Genie Spaces renamed Genie Agents | 2026-07-08 |
| BigQuery | `AI.GENERATE`, `AI.GENERATE_TABLE/TEXT/EMBEDDING` GA | 2025-11-18 |
| BigQuery | `AI.EMBED`, `AI.SIMILARITY` GA | 2026-03-23 |
| BigQuery | `AI.IF`, `AI.SCORE`, `AI.CLASSIFY` GA; `AI.AGG` GA 2026-10-13 | 2026-04-13 |
| BigQuery | BigLake → Google Cloud Lakehouse; Vertex AI → Gemini Enterprise Agent Platform | 2026-04-22 |
| BigQuery | Iceberg managed tables: partitioning + transactions GA | 2026-07-13 |
| Google | `gemini-2.0-flash` retired; current: `gemini-3.5-flash`, `gemini-3.8-flash`, `gemini-4-argon` | 2026-06-01 |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Pass a remote model to BigQuery `AI.GENERATE` | `AI.GENERATE(prompt, endpoint => 'gemini-3.5-flash')` |
| Use the legacy High-Concurrency profile as "serverless" | Serverless jobs/pipelines (no cluster spec); `data_security_mode` on classic |
| DBFS mount paths and the old `dlt` module | `/Volumes/...` and `from pyspark import pipelines as dp` |
| `LIMIT` to save BigQuery cost | Select columns, filter partitions, `--maximum_bytes_billed`, dry run |
| `SNOWFLAKE.CORTEX.COMPLETE` | `AI_COMPLETE` |
| Leave warehouses idle | `AUTO_SUSPEND = 60`; serverless SQL warehouse Auto Stop (default 10 min) |
| ZORDER / static partitioning on new Delta tables | Liquid clustering, `CLUSTER BY AUTO` |

## Related Documentation

| Topic | Path |
|-------|------|
| Dialects | `concepts/cross-platform-patterns.md` |
| Costs | `patterns/cost-optimization.md` |
| Full Index | `index.md` |

## Sources

- https://cloud.google.com/bigquery/pricing · https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf · https://www.databricks.com/product/pricing
- https://docs.cloud.google.com/bigquery/docs/release-notes · https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/model-versions
- https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql · https://docs.snowflake.com/en/user-guide/warehouses-adaptive
- https://docs.databricks.com/aws/en/release-notes/product/2026/june · https://docs.databricks.com/aws/en/release-notes/lakebase/ · https://docs.databricks.com/aws/en/ldp/concepts/where-is-dlt
