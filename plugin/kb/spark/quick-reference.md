# Spark Quick Reference

> **Purpose**: Fast lookup for Apache Spark 4.0–4.2 and Databricks Runtime 17.3 LTS (Spark 4.0.0) / 18 LTS (4.1.0) / 19 (4.2.0). Per-feature versions: `reference/version-matrix.md`.
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## DataFrame API Cheat Sheet

| Operation | PySpark | Spark SQL |
|-----------|---------|-----------|
| Select | `df.select("col_a", "col_b")` | `SELECT col_a, col_b` |
| Filter | `df.filter(F.col("age") > 21)` | `WHERE age > 21` |
| Group | `df.groupBy("dept").agg(F.sum("sal"))` | `GROUP BY dept` |
| Join | `df1.join(df2, "key", "inner")` | `JOIN t2 USING (key)` |
| Window | `Window.partitionBy("dept").orderBy("sal")` | `OVER (PARTITION BY dept ORDER BY sal)` |
| Safe cast | `F.col("c").try_cast("int")` (4.0+) | `try_cast(c AS INT)` |
| Top-1 per key | `row_number()` + `filter(rn == 1)` | `QUALIFY` (Databricks DBR 10.4+; OSS Spark 4.2+) |

## What Changed in Spark 4.x (and Where It Runs)

| Feature | OSS min | DBR 17.3 LTS | Notes |
|---------|---------|--------------|-------|
| ANSI mode on by default | 4.0 | Yes (DBR 17.0+) | Bad casts/overflow/÷0 fail → `try_*`; see `concepts/ansi-mode.md` |
| VARIANT + `variant_get` | 4.0 (GA 4.1) | Yes (DBR 15.4+) | `:` path syntax is SQL: Databricks 15.4+, OSS 4.1+ |
| Pipe syntax `\|>` | 4.0 | Yes (DBR 16.2+) | `\|` alias + aggregates in `\|> SELECT`: 4.2 / DBR 18.0+ |
| SQL scripting | 4.0 flag, GA 4.1 | Yes (DBR 16.3+) | |
| Recursive CTE | 4.1 | Yes (DBR 17.0+) | Default max recursion 100; 1M-row limit |
| `listagg` / `string_agg` | 4.0 | Yes | |
| `udf(useArrow=True)` | 3.5 | Yes | Still row-at-a-time Python; Arrow default in 4.2 |
| `@arrow_udf` (`pyarrow.Array`) | 4.1 | **No** | DBR 18 LTS+ |
| `transformWithStateInPandas` | 4.0 | Yes (DBR 16.2+) | Row-based Python `transformWithState`: OSS 4.1, Databricks DBR 16.2+ |
| Real-time mode trigger | 4.1 (Scala) / 4.2 (PySpark) | Yes (DBR 16.4 LTS+) | Photon off, update mode only |
| Plotting `df.plot` | 4.0 | Yes | Experimental; collects top 1000 rows / a sample to the driver |
| `pyspark-client`, `spark.api.mode` | 4.0 | n/a | Mode is chosen at session creation |

## Partitioning and Layout

| Method | Use when | Effect |
|--------|----------|--------|
| `repartition(n, "col")` | Need hash distribution by key | Full shuffle |
| `coalesce(n)` | Reduce partitions only | No shuffle |
| `/*+ REBALANCE(col) */` | Even output files before a write | AQE-sized (needs AQE) |
| `CLUSTER BY` / `.clusterBy()` | Delta tables (Databricks: all new tables) | Replaces partitioning + ZORDER |
| `CLUSTER BY AUTO` | UC managed + predictive optimization (DBR 15.4+) | Keys picked from query history |
| `.partitionBy("col")` | Plain Parquet/CSV outputs | Hive-style folders |

## Spark Connect Quick Reference

| Aspect | Detail |
|--------|--------|
| Where | Databricks standard access mode (Python DBR 14.3+), serverless, Databricks Connect; dedicated = Classic |
| Behaviour | Analysis is lazy: bad column names fail at the action or at `df.columns` |
| Not available | RDD APIs, JVM handles (`_jsc`, `_jdf`); temp views resolved by name at run time |
| OSS client | `pip install pyspark-client`; `SparkSession.builder.remote("sc://host:15002")` |

## Performance Settings Checklist

| Setting | OSS default | Databricks |
|---------|-------------|------------|
| `spark.sql.adaptive.enabled` | `true` (3.2+) | on (`spark.databricks.optimizer.adaptive.enabled`) |
| `spark.sql.shuffle.partitions` | `200` | `auto` recommended (default on serverless) |
| `spark.sql.autoBroadcastJoinThreshold` | `10MB` | static plan; AQE runtime switch up to `30MB` |
| `spark.sql.adaptive.skewJoin.enabled` | `true` | `true` (skew hints not needed) |
| `spark.sql.optimizer.dynamicPartitionPruning.enabled` | `true` | `true` |
| Photon | n/a | On by default (UI); API needs `runtime_engine: PHOTON` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `df.collect()` on large data | `df.take(10)`, `df.show()`, or write to a table |
| Python UDFs for simple transforms | Built-ins (`F.when`, `F.regexp_replace`) — UDFs also disable Photon |
| `cast` on dirty strings under ANSI | `try_cast` / `try_to_number`, then count the NULLs |
| `union`/`coalesce` of BIGINT and STRING columns | Cast every branch to the intended type first |
| Hard-coded `shuffle.partitions` | `auto` on Databricks; AQE coalescing on OSS |
| `.cache()` on serverless | Not supported there — materialize to a table or single-pass |
| `spark.sparkContext._jsc...` | `DESCRIBE DETAIL`, `df.explain()` — works on Spark Connect |
| `F.col("payload:a.b")` | `F.variant_get("payload", "$.a.b", "string")` |
| New Delta tables with `partitionBy` + ZORDER | Liquid clustering (`CLUSTER BY` / `AUTO`) |
| `input_file_name()` | `_metadata.file_path` (DBR 17.3 LTS+ no longer supports it) |
| `mapGroupsWithState` / `applyInPandasWithState` for new code | `transformWithState(InPandas)` |

## Related Documentation

| Topic | Path |
|-------|------|
| ANSI mode | `concepts/ansi-mode.md` |
| Version matrix | `reference/version-matrix.md` |
| Full Index | `index.md` |

## Sources

- https://spark.apache.org/releases/spark-release-4-0-0.html · https://spark.apache.org/releases/spark-release-4.1.0.html · https://spark.apache.org/releases/spark-release-4-2-0.html
- https://docs.databricks.com/aws/en/release-notes/runtime/ · https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://docs.databricks.com/aws/en/optimizations/aqe · https://docs.databricks.com/aws/en/spark/conf · https://docs.databricks.com/aws/en/compute/photon
- https://docs.databricks.com/aws/en/spark/connect-vs-classic · https://docs.databricks.com/aws/en/compute/serverless/limitations
- https://docs.databricks.com/aws/en/tables/clustering · https://docs.databricks.com/aws/en/sql/language-manual/functions/input_file_name
- https://spark.apache.org/docs/latest/configuration.html · https://spark.apache.org/docs/latest/sql-performance-tuning.html
