# Photon

> **Purpose**: Databricks' native vectorized engine — where it is on by default, what falls back to Spark, which features require it, and when to turn it off
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Photon is a Databricks-only C++ vectorized execution engine. Catalyst still plans the query;
Photon executes the supported operators in columnar batches and **transparently falls back** to the
Spark JVM engine for unsupported operations. It is not part of Apache Spark, so nothing here applies
to OSS clusters.

| Compute | Photon |
|---------|--------|
| Serverless compute, SQL warehouses, serverless Lakeflow pipelines | Always on |
| Classic all-purpose / jobs compute, classic Lakeflow pipelines (UI) | On by default ("Use Photon Acceleration") |
| Clusters API / Jobs API | Must be requested explicitly: `runtime_engine: PHOTON` |
| Pipelines API | `photon: true` |

## The Concept

```python
# Photon helps: built-in expressions, scans, hash joins/aggregates, window, Delta/Parquet writes
daily = (spark.read.table("main.sales.orders")
    .filter(F.col("order_date") >= "2026-01-01")
    .groupBy("region", "order_date")
    .agg(F.sum("amount").alias("revenue")))

# Photon falls back to the JVM engine for this stage: Python UDF (UDFs, RDD and Dataset APIs
# are not supported by Photon)
@F.udf("string")
def normalize(s): return s.strip().lower() if s else None

slow = daily.withColumn("region_norm", normalize("region"))
fast = daily.withColumn("region_norm", F.lower(F.trim("region")))   # stays in Photon
```

## Quick Reference

| Topic | Fact |
|-------|------|
| Operators covered | Scan (Parquet/Delta/CSV/JSON), filter, project, hash aggregate/join/shuffle, nested-loop join, null-aware anti join, spatial join, union, expand, sort/top-K/limit, window, Delta/Parquet write sink |
| Not supported | UDFs, RDD API, Dataset API, **stateful** streaming (stateless streaming to Delta/Parquet sinks is supported) |
| Requires Photon | Predictive I/O (read/write); dynamic file pruning in `MERGE`/`UPDATE`/`DELETE` |
| Incompatible with | Databricks **real-time mode** streaming (setup requires Photon off) |
| Small queries | Little or no gain for queries that run in under ~2 seconds |
| Cost | Photon instance types bill DBUs at a different rate than non-Photon — measure cost per job, not only runtime |
| Where to see it | Spark UI SQL/DataFrame tab (Photon operators in orange); query profile on serverless/SQL warehouses (% task time in Photon) |

## Decision Guide

| Workload | Recommendation |
|----------|----------------|
| SQL/DataFrame ETL with scans, joins, aggregations, MERGE | Keep Photon on |
| Pipeline dominated by Python UDFs / pandas UDFs / RDD code | Benchmark Photon on vs off (fallback means you pay the Photon rate for JVM execution) |
| Stateful streaming (aggregations with state, `transformWithState`) | Photon does not accelerate it |
| Real-time mode (`.trigger(realTime=...)`) | Photon must be off |
| Very short interactive queries | Gains are negligible |

## Common Mistakes

### Wrong

```json
{"spark_version": "17.3.x-scala2.13", "node_type_id": "...", "num_workers": 4}
```

Assuming the API-created job cluster gets Photon "by default" like the UI does.

### Correct

```json
{"spark_version": "17.3.x-scala2.13", "runtime_engine": "PHOTON", "node_type_id": "...", "num_workers": 4}
```

Then confirm in the Spark UI that the heavy stages run as Photon operators, and replace UDFs with
built-in functions where a stage falls back.

## Related

- [performance-tuning](../patterns/performance-tuning.md)
- [python-udfs](../patterns/python-udfs.md)
- [real-time-mode](../concepts/real-time-mode.md)

## Sources

- https://docs.databricks.com/aws/en/compute/photon
- https://docs.databricks.com/aws/en/structured-streaming/real-time/setup
