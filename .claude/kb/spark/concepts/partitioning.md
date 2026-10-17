# Partitioning and Data Layout

> **Purpose**: In-memory partitioning (repartition vs coalesce, AQE, `shuffle.partitions=auto`) versus on-disk layout (liquid clustering for Delta, `partitionBy` for plain files)
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Two different things share the word "partition":

1. **In-memory partitions** — how rows are spread across tasks. Adaptive Query Execution (AQE,
   default on since Spark 3.2 and on by default in Databricks) coalesces small post-shuffle partitions and
   splits skewed ones at runtime. On Databricks, `spark.sql.shuffle.partitions = auto`
   (auto-optimized shuffle) picks the initial count from the plan and input size; it is the default
   on serverless and opt-in on classic compute (default there is `200`).
2. **On-disk layout** — how files are organized. For Delta tables on Databricks, **liquid
   clustering** replaces Hive-style partitioning and `ZORDER` and is recommended for all new tables
   (GA in DBR 15.4 LTS; OSS Delta `CLUSTER BY` since 3.1). `partitionBy` remains the tool for
   plain Parquet/CSV/JSON outputs.

## The Concept

```python
from pyspark.sql import functions as F

# In-memory: let AQE / auto-optimized shuffle size partitions (Databricks)
spark.conf.set("spark.sql.shuffle.partitions", "auto")      # Databricks only; OSS needs an integer

# repartition: full shuffle, hash-distributes by key (e.g. before a heavy per-key operation)
by_customer = df.repartition("customer_id")

# coalesce: no shuffle, only reduces the partition count
small = df.filter(F.col("status") == "active").coalesce(10)

# On disk, Delta / Unity Catalog: cluster, don't partition
(df.write
    .clusterBy("order_date", "customer_id")       # DataFrameWriter.clusterBy: Spark 4.0+
    .mode("overwrite")
    .saveAsTable("main.sales.orders"))

# On disk, plain Parquet files: Hive-style partition folders
df.write.partitionBy("order_date").mode("overwrite").parquet("s3://bucket/exports/orders/")
```

## Quick Reference

| Strategy | Shuffle? | Use when | Result |
|----------|----------|----------|--------|
| `repartition(n)` / `repartition(n, "col")` | Yes | Need even distribution or hash by key | Exactly n |
| `repartition("col")` | Yes | Downstream per-key work | `spark.sql.shuffle.partitions` (or AQE-coalesced) |
| `coalesce(n)` | No | Only reducing partitions | At most n |
| `coalesce(n)` / `repartition(n)` right before a Delta write | — | Avoid on Databricks when optimized writes apply (MERGE/UPDATE/DELETE, UC CTAS/INSERT on DBR 13.3 LTS+) | Let optimized writes size files |
| `/*+ REBALANCE(col) */` hint | Yes | Even-sized output files before a write (needs AQE) | AQE-sized |
| AQE coalesce | Runtime | Default | Merges small partitions (Databricks advisory size 64 MB) |
| `shuffle.partitions=auto` | — | Databricks classic (opt-in) / serverless (default) | Initial count from plan + input size |
| `CLUSTER BY` / `.clusterBy()` | Write | Delta tables (Databricks: all new tables) | Keys can change later without rewrite |
| `CLUSTER BY AUTO` | Write | UC managed tables with predictive optimization (DBR 15.4+) | Keys chosen from query history |
| `.partitionBy("col")` | Write | Non-Delta file outputs; legacy Delta tables | One folder per value |
| `.bucketBy(n, "col")` | Write | Non-Delta catalog tables written with `saveAsTable` | Not the recommended layout for Delta |

## Common Mistakes

### Wrong

```python
# Hard-coded shuffle partitions tuned once for yesterday's volume
spark.conf.set("spark.sql.shuffle.partitions", "2000")

# coalesce(1) on large data — single task, OOM risk
df.coalesce(1).write.parquet("output/")

# New Delta table partitioned by a high-cardinality column
df.write.format("delta").partitionBy("customer_id").saveAsTable("main.sales.orders")
```

### Correct

```python
spark.conf.set("spark.sql.shuffle.partitions", "auto")   # Databricks; AQE handles the rest
df.write.clusterBy("customer_id").saveAsTable("main.sales.orders")
# Existing partitioned table (DBR 18.1+): ALTER TABLE main.sales.orders
#   REPLACE PARTITIONED BY WITH CLUSTER BY (customer_id);
```

## Related

- [catalyst-optimizer](../concepts/catalyst-optimizer.md)
- [performance-tuning](../patterns/performance-tuning.md)
- [delta-integration](../patterns/delta-integration.md)

## Sources

- https://docs.databricks.com/aws/en/optimizations/aqe
- https://docs.databricks.com/aws/en/spark/conf
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/tables/tune-file-size
- https://docs.delta.io/latest/delta-clustering.html
- https://spark.apache.org/docs/latest/sql-performance-tuning.html
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-hints.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
