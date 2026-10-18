# Databricks Lakeflow & AI Platform

> **Purpose**: Databricks platform map in 2026 — Lakeflow (Connect, Spark Declarative Pipelines, Jobs, Designer), Unity Catalog, serverless, runtimes, Lakebase, Agent Bricks, AI Search and Genie
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Data engineering on Databricks is branded **Lakeflow** (lowercase "f"): **Lakeflow Connect** (managed
ingestion), **Lakeflow Spark Declarative Pipelines** ("Lakeflow pipelines", formerly Delta Live Tables),
**Lakeflow Jobs** (orchestration) and **Lakeflow Designer** (visual, no-code data prep). Everything is
governed by Unity Catalog and runs best on serverless compute. The AI layer is **Agent Bricks**
(agent platform), **AI Search** (formerly Vector Search), Foundation Model APIs behind **Unity Gateway**,
and the **Genie** family for business users.

## The Concept

```python
# Lakeflow pipeline source file (Python). The module was previously `dlt`.
from pyspark import pipelines as dp
from pyspark.sql import functions as F

@dp.table(comment="Raw orders, incremental from a UC volume")      # streaming table
def orders_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load("/Volumes/main/raw/landing/orders/"))

@dp.table(comment="Validated orders", cluster_by_auto=True)
@dp.expect_or_drop("valid_amount", "order_amount > 0")
@dp.expect_or_fail("valid_id", "order_id IS NOT NULL")
@dp.expect("has_customer", "customer_id IS NOT NULL")                # warn only
def orders_silver():
    return (spark.readStream.table("orders_bronze")
            .withColumn("ingested_at", F.current_timestamp())
            .where(F.col("status") != "TEST"))

@dp.materialized_view(comment="Daily revenue")                      # batch, incrementally refreshed
def daily_revenue_gold():
    return (spark.read.table("orders_silver")
            .groupBy("order_date")
            .agg(F.sum("order_amount").alias("total_revenue"),
                 F.count("order_id").alias("order_count")))
```

`@dp.table` returning a streaming DataFrame = streaming table; `@dp.materialized_view` = MV;
`@dp.temporary_view` = pipeline-scoped view; CDC via `dp.create_auto_cdc_flow` (replaces `apply_changes`).
Read other datasets with `spark.read.table()` / `spark.readStream.table()`. Apache Spark 4.1+ ships the
same `pyspark.pipelines` module; `dp.expect*` and AUTO CDC flows are Databricks-only.

## Quick Reference

| Area | Current state (2026-10-17) |
|------|----------------------------|
| Lakeflow Connect | Managed connectors in "various release states": database CDC (MySQL, PostgreSQL, SQL Server), SaaS (Salesforce, Workday, HubSpot, Jira...), files (Google Drive, SharePoint), query-based, streaming; check status per connector |
| Lakeflow pipelines | Serverless by default; files in a `transformations` folder edited in the Lakeflow Pipelines Editor (GA 2026-05-04) |
| Lakeflow Jobs | Serverless for notebook, Python script, wheel, dbt and JAR tasks; triggers: schedule, file arrival, table update, continuous |
| Deployment | **Declarative Automation Bundles** (DABs, formerly Databricks Asset Bundles): jobs, pipelines and apps as YAML; mechanics in the Lakeflow KB |
| Lakeflow Designer | Visual "visual data preps" backed by code, UC-governed; Public Preview 2026-04-22, GA 2026-06-16 |
| Runtimes | 17.3 LTS (Spark 4.0.0, EoS 2028-10-22), **18 LTS** (Spark 4.1.0, 2026-06-10), 19 (Spark 4.2.0, Beta 2026-06-15); 16.4/15.4/14.3 LTS still supported |
| Storage paths | DBFS root and mounts (`/mnt`) deprecated; use UC volumes (`/Volumes/...`) and external locations |
| Table layout | Liquid clustering (GA DBR 15.4 LTS+) replaces partitioning and ZORDER; `CLUSTER BY AUTO` on UC managed tables with predictive optimization |
| Iceberg | UC managed Iceberg, foreign Iceberg and Iceberg v3 features GA 2026-05-21; Iceberg REST catalog in UC |
| Lakebase | Managed Postgres (autoscaling, scale-to-zero, branching, instant restore); GA 2026-01-22 on AWS, 2026-03-02 on Azure |
| Agent Bricks | Agent developer platform: Agent Runtime, AgentKit, Sandbox, memory/sessions on Lakebase, MCP servers, Agent Bricks CLI |
| Models | Foundation Model APIs (`databricks-<model>` endpoints, e.g. Claude, GPT, Gemini, Llama, Qwen) through Unity Gateway; list changes often, see supported-models page |
| AI Search | Formerly Vector Search (renamed 2026-06-01): vector, hybrid and full-text indexes with reranking |
| Genie | Genie One (business-user interface, account-level, mobile), Genie Agents (formerly Genie Spaces, renamed 2026-07-08), Genie Code (coding assistant) |
| SQL | `QUALIFY` (DBR 10.4 LTS+), `MERGE ... UPDATE SET *`, VARIANT (15.4+), pipe syntax (16.2+) |

## Common Mistakes

### Wrong

```python
# Legacy: still runs, but Databricks recommends pyspark.pipelines; DBFS mounts are deprecated
import dlt
@dlt.table
def orders_bronze():
    return spark.readStream.format("cloudFiles").load("/mnt/landing/orders/")

spark.sql("CREATE TABLE hive_metastore.default.orders (order_id STRING)")  # legacy metastore
```

### Correct

```python
from pyspark import pipelines as dp

@dp.table
def orders_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load("/Volumes/main/raw/landing/orders/"))

spark.sql("CREATE TABLE main.sales.orders (order_id STRING, amount DECIMAL(10,2)) CLUSTER BY AUTO")
```

## Related

- [Databricks Patterns](../patterns/databricks-patterns.md) — expectations, serverless jobs, UC setup, MLflow 3
- [Cost Optimization](../patterns/cost-optimization.md) — serverless modes, SQL warehouse auto stop, billing tables
- [Cross-Platform Patterns](cross-platform-patterns.md) — SQL dialect differences
- [Lakeflow KB](../../lakeflow/index.md) — deep dive on pipelines (this file is only the platform map)
- [Lakeflow Jobs reference](../../lakeflow/reference/lakeflow-jobs.md) — tasks, triggers, compute
- [Bundles CLI reference](../../lakeflow/reference/bundles-cli.md) — Declarative Automation Bundles

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/concepts/where-is-dlt
- https://docs.databricks.com/aws/en/ingestion/lakeflow-connect/
- https://docs.databricks.com/aws/en/jobs/run-serverless-jobs
- https://docs.databricks.com/aws/en/dev-tools/bundles/
- https://docs.databricks.com/aws/en/designer/
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://docs.databricks.com/aws/en/release-notes/product/2026/april
- https://docs.databricks.com/aws/en/release-notes/product/2026/may
- https://docs.databricks.com/aws/en/release-notes/product/2026/june
- https://docs.databricks.com/aws/en/release-notes/product/2026/july
- https://docs.databricks.com/aws/en/release-notes/lakebase/
- https://learn.microsoft.com/en-us/azure/databricks/release-notes/lakebase/
- https://docs.databricks.com/aws/en/dbfs/unity-catalog
- https://docs.databricks.com/aws/en/delta/clustering
- https://docs.databricks.com/aws/en/generative-ai/agent-bricks/
- https://docs.databricks.com/aws/en/vector-search/vector-search
- https://docs.databricks.com/aws/en/genie/
- https://docs.databricks.com/aws/en/machine-learning/foundation-model-apis/supported-models
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify
