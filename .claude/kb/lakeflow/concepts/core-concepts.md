# Lakeflow Pipelines - Core Concepts

> **Purpose**: Datasets, flows, sinks, and pipelines in Lakeflow Spark Declarative Pipelines (formerly Delta Live Tables)
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

Lakeflow Spark Declarative Pipelines (SDP), called "Lakeflow pipelines" in the Databricks docs, is a
declarative framework for batch and streaming pipelines in SQL and Python. The product formerly known
as Delta Live Tables (DLT) was renamed with no migration required; old code still runs. Lakeflow
pipelines extend Apache Spark Declarative Pipelines, which ships with Apache Spark 4.1+.

## The Concept

```python
from pyspark import pipelines as dp   # Python module (formerly the `dlt` module)

@dp.table                              # streaming table: streaming read
def orders_raw():
    return spark.readStream.table("main.sales.orders_landing")

@dp.materialized_view                  # materialized view: batch read
def orders_by_day():
    return spark.read.table("orders_raw").groupBy("order_date").count()

@dp.temporary_view                     # view: not persisted, pipeline-scoped
def open_orders():
    return spark.read.table("orders_raw").where("status = 'OPEN'")
```

### Datasets

| Dataset | Processing semantics | Use for |
|---------|----------------------|---------|
| Streaming table | Each record processed once (append-only source) | Ingestion, incremental transforms |
| Materialized view | Recomputed as needed, incrementally when possible | Joins, aggregations, sources with updates/deletes |
| View (temporary) | Evaluated on demand, not published | Intermediate steps, checks |

### Flows

| Flow type | Notes |
|-----------|-------|
| Append (default) | Any number of append flows can write to one target; `ONCE` = one-time backfill |
| Auto CDC (previously apply changes) | Targets streaming tables only; multiple Auto CDC flows per target |
| Update (Public Preview) | Python only; global non-watermarked aggregates to a sink |

The docs state that only Append and Update streaming output modes are exposed (Complete is not).

### Sinks

Streaming targets outside the pipeline: Delta tables (UC managed or external), Apache Kafka, Azure
Event Hubs (Kafka interface), custom Python data sources, and the ForEachBatch sink. Python only.

### Pipeline

The unit of development and execution. It analyzes dependencies between datasets and runs flows in
order with maximum parallelism. Mode: triggered (default) or continuous.

## Runtime: channel, not cluster version

Pipelines are versionless; the `channel` setting picks the runtime. As of 2026-10-17:
`CURRENT` (default) = Databricks Runtime 18, `PREVIEW` = Databricks Runtime 19. This is independent of
the DBR version your Lakeflow Jobs clusters use (for example DBR 17.3 LTS).

## Quick Reference

| Need | Use |
|------|-----|
| Ingest files incrementally | Streaming table + Auto Loader |
| Aggregate or join | Materialized view |
| Reusable intermediate logic | Temporary view |
| SCD Type 1/2 from a change feed | `AUTO CDC` flow |
| Write to Kafka or an external table | Sink + append flow |
| Develop and debug | Lakeflow Pipelines Editor (files in `transformations/`) |

## Common Mistakes

### Wrong

```python
import dlt  # legacy module name; still works, not recommended

@dlt.table  # legacy: one decorator for both STs and MVs
def daily_totals():
    return spark.read.table("orders").groupBy("order_date").count()
```

### Correct

```python
from pyspark import pipelines as dp

@dp.materialized_view  # batch query -> materialized view
def daily_totals():
    return spark.read.table("orders").groupBy("order_date").count()
```

## Recent Platform Changes

| When | Change |
|------|--------|
| 2025.04 | Default publishing mode: multi-catalog/schema, `LIVE` schema no longer required |
| 2025.29 | Move tables between pipelines GA |
| 2025.30 | Auto CDC: `name` (multiple flows per target), `once`, `CREATE FLOW` in SQL |
| Feb 2026 | Type widening; SCD Type 1 materialization with AUTO CDC |
| May 2026 | Bitemporal AUTO CDC (Beta) |
| Jun 2026 | Liquid clustering on AUTO CDC targets |
| Aug 2026 | `CREATE TABLE ... FLOW` / `dp.create_table()` (Beta); event log system table (Beta) |
| Sep 2026 | Run-as account group; Rewind (Beta); Python unit testing (Beta) |

## Related

- [Python module migration](python-module-migration.md)
- [Getting started](getting-started.md)
- [Flows and sinks](../patterns/flows-and-sinks.md)
- [Limitations](../reference/limitations.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/concepts
- https://docs.databricks.com/aws/en/ldp/concepts/where-is-dlt
- https://docs.databricks.com/aws/en/ldp/concepts/flows
- https://docs.databricks.com/aws/en/ldp/ldp-sinks
- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/release-notes/dlt/
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/04/
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/30/
