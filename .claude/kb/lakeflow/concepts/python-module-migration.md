# Python Module Migration (`dlt` to `pyspark.pipelines`)

> **Purpose**: Map the legacy Delta Live Tables Python API to the current `pyspark.pipelines` API and explain what is Databricks-only
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

All Lakeflow pipelines Python APIs live in `pyspark.pipelines`, imported as `dp`. The product was
formerly known as Delta Live Tables, and its Python module was `dlt`. The `dlt` module still works,
but Databricks recommends `pyspark.pipelines`. The same module exists in Apache Spark 4.1+ (Spark
Declarative Pipelines); Databricks adds APIs on top of it.

## The Concept

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

@dp.table(comment="Raw events")                      # streaming table
def events_raw():
    return spark.readStream.table("main.raw.events")

@dp.materialized_view(cluster_by_auto=True)          # materialized view
def events_daily():
    return (spark.read.table("events_raw")
            .groupBy(F.to_date("event_ts").alias("event_date"))
            .count())

@dp.temporary_view                                   # not published to the catalog
def events_valid():
    return spark.read.table("events_raw").where("event_id IS NOT NULL")
```

## Quick Reference

Legacy names below are shown only for migration. Use the right-hand column in new code.

| Formerly (`dlt` module) | Current (`dp`) | In Apache Spark SDP |
|-------------------------|----------------|---------------------|
| `import dlt` (legacy) | `from pyspark import pipelines as dp` | Yes (4.1+) |
| `@dlt.table` + streaming DataFrame (legacy) | `@dp.table` | Yes |
| `@dlt.table` + batch DataFrame (legacy) | `@dp.materialized_view` | Yes |
| `@dlt.view` (legacy) | `@dp.temporary_view` | Yes |
| `@dlt.append_flow` (legacy) | `@dp.append_flow` | Yes |
| `dlt.create_streaming_table` (legacy) | `dp.create_streaming_table` | Yes |
| `dlt.create_sink` (legacy) | `dp.create_sink(name, format, options)` | Yes |
| `dlt.apply_changes` (legacy) | `dp.create_auto_cdc_flow` | Databricks only |
| `dlt.apply_changes_from_snapshot` (legacy) | `dp.create_auto_cdc_from_snapshot_flow` | Databricks only |
| `@dlt.expect*` (legacy) | `@dp.expect*` | Databricks only |
| (none) | `@dp.update_flow` (Public Preview) | Databricks only |
| (none) | `@dp.foreach_batch_sink` | Databricks only |
| `dlt.read("x")` / `dlt.read_stream("x")` (legacy) | `spark.read.table("x")` / `spark.readStream.table("x")` | Yes |
| `temporary=True` | `private=True` | - |

Other current APIs: `dp.create_table` (Beta, Aug 2026), `dp.replace_flow`. Event log: query with
`event_log(<pipeline_id>)` or a published event log table.

## Apache Spark Declarative Pipelines (open source)

| Item | Value |
|------|-------|
| First release | Apache Spark 4.1.0 (SPARK-51727) |
| Install | `pip install pyspark[pipelines]` |
| CLI | `spark-pipelines init`, `spark-pipelines run`, `spark-pipelines dry-run` |
| Project spec | `spark-pipeline.yml` |
| Spark 4.2.0 | Adds Auto CDC for SCD Type 1 (SPARK-56249) |

The Databricks Python reference still lists `create_auto_cdc_flow`, `create_auto_cdc_from_snapshot_flow`
and `@dp.expect` as not part of Apache Spark. Check the Spark 4.2 docs before relying on Auto CDC in
open source; SCD Type 2 and bitemporal remain Databricks-only.

Version note: the open-source module needs Spark 4.1+. Databricks Runtime 17.3 LTS ships Spark 4.0,
so `spark-pipelines` does not run on a DBR 17.3 classic cluster. Lakeflow pipelines on Databricks use
their own channel runtime (CURRENT = DBR 18, PREVIEW = DBR 19 as of 2026-10-17).

## Common Mistakes

### Wrong

```python
import dlt                      # Wrong: legacy module (formerly DLT)

@dlt.view                       # legacy decorator
def stage():
    return dlt.read("raw")      # legacy reader
```

### Correct

```python
from pyspark import pipelines as dp

@dp.temporary_view
def stage():
    return spark.read.table("raw")
```

## Related

- [Core concepts](core-concepts.md)
- [Python decorators](../patterns/python-decorators.md)
- [CDC fundamentals](cdc-fundamentals.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/concepts/where-is-dlt
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-table
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-view
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://spark.apache.org/docs/latest/declarative-pipelines-programming-guide.html
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/releases/spark-release-4-2-0.html
