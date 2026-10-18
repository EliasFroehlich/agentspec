# Performance Tuning

> **Purpose**: Diagnose and fix slow Spark jobs — AQE and auto-optimized shuffle, broadcast joins, skew (AQE first, salting when needed), caching rules, write sizing, Photon — for OSS Spark 4.x and Databricks
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- A stage dominates runtime (shuffle, skewed tasks, spill) or the job fails with OOM
- Many tiny tasks or tiny output files, or a few huge ones
- Cost per run is too high on Databricks (check Photon, cluster sizing, layout)

Measure first: Spark UI (classic compute) or the query profile (serverless / SQL warehouses).

## Implementation

```python
from pyspark.sql import functions as F

# --- 1. AQE + SHUFFLE SIZING ---------------------------------------------------
# AQE is on by default (OSS since 3.2; Databricks too). Don't re-set defaults.
# Databricks: let auto-optimized shuffle choose the initial partition count (serverless default).
spark.conf.set("spark.sql.shuffle.partitions", "auto")      # Databricks only
# OSS Spark: keep an integer and let AQE coalesce:
# spark.conf.set("spark.sql.shuffle.partitions", "400")

# --- 2. BROADCAST JOINS -----------------------------------------------------------
# Static planning uses spark.sql.autoBroadcastJoinThreshold (OSS default 10 MB).
# Databricks AQE also converts sort-merge -> broadcast at runtime below
# spark.databricks.adaptive.autoBroadcastJoinThreshold (default 30 MB).
# If you know a dimension is small, hint it: a planned broadcast beats a runtime switch,
# because AQE may only switch after both sides were already shuffled.
result = orders_df.join(F.broadcast(dim_country), "country_code", "left")
# Note: the left side of a LEFT OUTER JOIN cannot be broadcast.

# --- 3. SKEW -------------------------------------------------------------------------
# AQE splits skewed partitions of sort-merge / shuffle-hash joins automatically when a partition
# is > skewedPartitionFactor (5) x median AND > skewedPartitionThresholdInBytes (256 MB).
# Databricks: prefer AQE over skew hints (skew-hint docs are retired).
# Salting is still needed where AQE does not help: skewed groupBy/aggregations, FULL OUTER joins,
# the non-optimizable side of LEFT/RIGHT joins.
SALT = 16
hot = (orders_df
    .withColumn("salt", (F.rand(seed=7) * SALT).cast("int"))
    .groupBy("customer_id", "salt").agg(F.sum("amount").alias("partial"))   # spread the hot key
    .groupBy("customer_id").agg(F.sum("partial").alias("amount")))          # merge partials

# --- 4. CACHING ----------------------------------------------------------------------
# Only when a DataFrame is reused AND recomputing it is expensive. Always release it.
# NOT supported on Databricks serverless: df.cache()/persist()/unpersist(), CACHE TABLE raise errors.
filtered = orders_df.filter(F.col("status") == "active").cache()
try:
    summary = filtered.groupBy("region").agg(F.sum("amount"))
    detail = filtered.filter(F.col("amount") > 1000)
    summary.write.mode("overwrite").saveAsTable("main.sales.region_summary")
    detail.write.mode("overwrite").saveAsTable("main.sales.big_orders")
finally:
    filtered.unpersist()
# Portable alternative (serverless, Spark Connect): materialize the intermediate as a table,
# or restructure into a single pass.

# --- 5. WRITE SIZING -------------------------------------------------------------------
# Delta on Databricks (UC managed): file size is auto-tuned; use liquid clustering +
# predictive optimization instead of manual repartition/OPTIMIZE schedules.
# Elsewhere: REBALANCE asks AQE for evenly sized output partitions (ignored if AQE is off).
spark.sql("""
    INSERT OVERWRITE TABLE main.sales.orders_curated
    SELECT /*+ REBALANCE(order_date) */ * FROM main.sales.orders_stage
""")
# Table size without JVM internals (works on Spark Connect):
size_bytes = spark.sql("DESCRIBE DETAIL main.sales.orders").first()["sizeInBytes"]
```

## Configuration

| Setting | OSS Spark default | Databricks | Recommendation |
|---------|-------------------|------------|----------------|
| `spark.sql.adaptive.enabled` / `spark.databricks.optimizer.adaptive.enabled` | `true` (3.2+) | `true` | Leave on |
| `spark.sql.shuffle.partitions` | `200` | `200` classic, `auto` serverless | Databricks: `auto`; OSS: size to data, AQE coalesces |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | follows `targetPostShuffleInputSize` | `64MB` | Leave unless tasks are clearly too big/small |
| `spark.sql.autoBroadcastJoinThreshold` | `10MB` | static planning | Raise cautiously — broadcast tables are collected through the driver |
| `spark.databricks.adaptive.autoBroadcastJoinThreshold` | — | `30MB` | Runtime SMJ→BHJ switch |
| `spark.sql.adaptive.skewJoin.enabled` | `true` | `true` | Leave on |
| `spark.sql.files.maxPartitionBytes` | `128MB` | `128MB` | Input split size, not output file size |
| `spark.sql.optimizer.dynamicPartitionPruning.enabled` | `true` | `true` | Leave on |

On Databricks, executor memory/cores come from the node type and cluster policy; tune by choosing
the instance type and worker count, not `spark.executor.memory`. On standard access mode some Spark
properties cannot be set at all.

## Photon (Databricks)

On by default for classic jobs/all-purpose compute created in the UI (API: `runtime_engine:
PHOTON`), always on for serverless and SQL warehouses. UDFs, RDD/Dataset APIs and stateful
streaming fall back to the JVM engine. See [photon](../concepts/photon.md).

## Example Usage

```python
# Diagnose: rows per partition after a shuffle
df.groupBy(F.spark_partition_id().alias("pid")).count().orderBy(F.desc("count")).show(10)

# Diagnose: hot join keys
df.groupBy("join_key").count().orderBy(F.desc("count")).show(10)

# Did AQE change the plan? Run the action, then inspect the final plan
df.explain(mode="formatted")   # look for AdaptiveSparkPlan isFinalPlan=true, Coalesced, isSkew=true
```

## See Also

- [partitioning](../concepts/partitioning.md)
- [catalyst-optimizer](../concepts/catalyst-optimizer.md)
- [photon](../concepts/photon.md)
- [python-udfs](../patterns/python-udfs.md)
- [delta-integration](../patterns/delta-integration.md)
- [spark-connect](../concepts/spark-connect.md)

## Sources

- https://docs.databricks.com/aws/en/optimizations/aqe
- https://docs.databricks.com/aws/en/optimizations/skew-join
- https://docs.databricks.com/aws/en/spark/conf
- https://docs.databricks.com/aws/en/compute/photon
- https://docs.databricks.com/aws/en/compute/serverless/limitations
- https://docs.databricks.com/aws/en/compute/standard-limitations
- https://docs.databricks.com/aws/en/delta/tune-file-size
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- https://spark.apache.org/docs/latest/sql-performance-tuning.html
- https://spark.apache.org/docs/latest/configuration.html
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-hints.html
- https://docs.databricks.com/aws/en/delta/table-details
