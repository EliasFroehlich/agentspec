# Cloud Platforms Knowledge Base

> **Purpose**: Cloud data platforms — Snowflake (Cortex AI Functions, Gen2/Adaptive warehouses), Databricks (Lakeflow, Unity Catalog, Agent Bricks), BigQuery (AI.GENERATE family, Iceberg managed tables), cross-platform SQL and cost
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17
> **Latest**: BigQuery managed AI functions GA (AI.AGG 2026-10-13); Databricks Lakeflow Designer GA and Vector Search → AI Search (Jun 2026); Snowflake Adaptive Compute GA (2026-06-16) and legacy SNOWFLAKE.CORTEX.* retirement (BCR 2026_08)

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/snowflake-cortex.md](concepts/snowflake-cortex.md) | Cortex AI Functions (AI_*), models, Cortex Agents, Gen2/Adaptive, dynamic tables, Snowpipe Streaming, Iceberg |
| [concepts/databricks-lakeflow.md](concepts/databricks-lakeflow.md) | Lakeflow (Connect, pipelines, Jobs, Designer), runtimes, Lakebase, Agent Bricks, AI Search, Genie |
| [concepts/bigquery-ai.md](concepts/bigquery-ai.md) | AI.GENERATE / AI.EMBED / managed AI functions, Gemini model IDs, Iceberg managed tables, BQML |
| [concepts/cross-platform-patterns.md](concepts/cross-platform-patterns.md) | Dialect differences (DATE_TRUNC, JSON, MERGE, QUALIFY), migration notes |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/snowflake-patterns.md](patterns/snowflake-patterns.md) | Snowpipe, dynamic tables, streams + tasks, warehouse sizing, AI functions in SQL |
| [patterns/databricks-patterns.md](patterns/databricks-patterns.md) | Lakeflow pipelines with expectations and AUTO CDC, serverless Lakeflow Jobs, UC setup, MLflow 3 |
| [patterns/bigquery-patterns.md](patterns/bigquery-patterns.md) | Idempotent loads, BQML forecasting, Cloud Run remote functions, Gemini in SQL, Iceberg DDL |
| [patterns/cost-optimization.md](patterns/cost-optimization.md) | Dated list prices, idle shutdown, spend caps, sizing, storage tiers, cost queries |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Platform comparison, AI-in-SQL map, prices, 2025–2026 changes

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Scalar AI functions** | LLM calls per row in SQL: `AI_COMPLETE` (Snowflake), `ai_query` (Databricks), `AI.GENERATE` (BigQuery) |
| **Managed AI functions** | Task-specific, model-agnostic: `AI_CLASSIFY`/`AI_FILTER`, `ai_classify`, `AI.CLASSIFY`/`AI.IF` |
| **Declarative pipelines** | Dynamic tables (Snowflake), Lakeflow Spark Declarative Pipelines (Databricks), Dataform (BigQuery) |
| **Open table formats** | Iceberg on all three; Delta + UniForm on Databricks |
| **Serverless-first compute** | Adaptive Warehouses, Databricks serverless jobs/pipelines, BigQuery editions |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | quick-reference.md, concepts/cross-platform-patterns.md |
| **Intermediate** | concepts/snowflake-cortex.md, concepts/databricks-lakeflow.md, concepts/bigquery-ai.md |
| **Advanced** | patterns/*.md, patterns/cost-optimization.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| data-platform-engineer | All files | Platform selection, configuration, cost |
| lakehouse-architect | patterns/cost-optimization.md, concepts/cross-platform-patterns.md | Infrastructure planning, format portability |
| gcp-data-architect | concepts/bigquery-ai.md, patterns/bigquery-patterns.md | BigQuery AI and cost |

---

## Sources

See the `## Sources` section of each file; pricing pages: https://cloud.google.com/bigquery/pricing ·
https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf · https://www.databricks.com/product/pricing
