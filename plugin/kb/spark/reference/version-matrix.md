# Spark / Databricks Runtime Version Matrix

> **Purpose**: Which Spark SQL / PySpark feature is available in which Apache Spark release and Databricks Runtime (DBR 17.3 LTS, 18 LTS, 19) — check here before using a feature
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Runtime Baseline

| Line | Release | Bundled Spark | Notes |
|------|---------|---------------|-------|
| Apache Spark 4.2.0 | 2026-07-14 | — | Current `latest` docs; Java 17/21/25, Python 3.10+ |
| Apache Spark 4.1.x | 4.1.0 (2025-12-16); 4.1.3 (2026-07-15) | — | Maintenance line |
| Apache Spark 4.0.x | 4.0.0 (2025-05-23); 4.0.4 (2026-07-15) | — | Scala 2.13 only from 4.0 |
| DBR 17.3 LTS | 2025-10-22 (EoS 2028-10-22) | 4.0.0 | Python 3.12.3, Java 17, Delta 4.0.0 |
| DBR 18 LTS | GA 2026-06-10 (support to 2029-06-10) | 4.1.0 | Unified release notes: includes everything shipped as 18.0, 18.1, 18.2 and later dated updates |
| DBR 19 | 2026-06-15 | 4.2.0 | Not LTS at the time of writing |

Rule of thumb: an **OSS Spark 4.1+ API is not in DBR 17.3** unless Databricks ships it earlier — it often
does (QUALIFY, recursive CTE, Python row-based `transformWithState`), so check both columns.

## Feature Matrix

`Yes` = available; `No` = not available; versions are the minimum. For DBR 18 LTS / DBR 19, rows marked
`Spark 4.x` are OSS features of the bundled Spark release — confirm in the DBR release notes before relying on them.

| Feature | Apache Spark (min) | Databricks (min) | DBR 17.3 LTS | DBR 18 LTS | DBR 19 |
|---------|--------------------|------------------|--------------|------------|--------|
| ANSI mode on by default | 4.0 | DBR 17.0 | Yes | Yes | Yes |
| `try_cast`, `try_divide`, `try_add`, `try_element_at`, `try_to_number`… | in 4.0 SQL | — | Yes | Yes | Yes |
| `Column.try_cast` (PySpark) | 4.0 | — | Yes | Yes | Yes |
| Pipe syntax `\|>` | 4.0 | DBR 16.2 | Yes | Yes | Yes |
| `\|` alias of `\|>`, aggregates inside `\|> SELECT` | 4.2 | DBR 18.0 | No | Yes | Yes |
| SQL UDF (`CREATE FUNCTION … RETURN`) | 4.0 | — | Yes | Yes | Yes |
| SQL scripting (`BEGIN … END`) | 4.0 behind a flag; GA 4.1 | DBR 16.3 | Yes (Databricks) | Yes | Yes |
| Scripting cursors / CONTINUE handler | 4.2 (cursors) | DBR 18.1 | No | Yes | Yes |
| `CREATE PROCEDURE` | — | DBR 17.0 | Yes | Yes | Yes |
| Recursive CTE (`WITH RECURSIVE`) | 4.1 | DBR 17.0 (`LIMIT ALL` 17.2) | Yes (Databricks) | Yes | Yes |
| `QUALIFY` | 4.2 | DBR 10.4 LTS | Yes (Databricks) | Yes | Yes |
| `listagg` / `string_agg` | 4.0 | — | Yes | Yes | Yes |
| `UNPIVOT` clause, `DataFrame.unpivot/melt` | 3.4 | — | Yes | Yes | Yes |
| `DataFrame.lateralJoin` | 4.0 | — | Yes | Yes | Yes |
| Collations (`COLLATE`, `UTF8_LCASE`) | 4.0 | DBR 16.1 | Yes | Yes | Yes |
| VARIANT type, `parse_json`, `variant_get` | 4.0 (GA/default 4.1) | DBR 15.4 | Yes | Yes | Yes |
| `:` path operator on VARIANT | 4.1 | DBR 15.4 (SQL) | Yes (Databricks SQL) | Yes | Yes |
| `time_bucket` | 4.2 | — | No | No | Spark 4.2 |
| `GEOMETRY` / `GEOGRAPHY` types | 4.2 | see Databricks spatial docs | — | — | Spark 4.2 |
| `CHANGES` clause (CDC read) | 4.2 | — | No | No | Spark 4.2 |
| `udf(..., useArrow=True)` (Arrow-optimized Python UDF) | 3.5 (default on in 4.2) | — | Yes (opt-in) | Yes (opt-in) | Yes (default) |
| `@arrow_udf` (operates on `pyarrow.Array`) | 4.1 | — | **No** | Spark 4.1 | Spark 4.2 |
| Python Data Source API | 4.0 (filter pushdown 4.1) | — | Yes | Yes | Yes |
| `DataFrame.plot` (Plotly, experimental) | 4.0 | — | Yes | Yes | Yes |
| `transformWithState` (Scala/Java), `transformWithStateInPandas` | 4.0 | DBR 16.2 | Yes | Yes | Yes |
| `transformWithState` Python Row API | 4.1 | DBR 16.2 | Yes (Databricks) | Yes | Yes |
| Real-time mode trigger | 4.1 (Scala, stateless); PySpark trigger 4.2 | DBR 16.4 LTS (Lakeflow pipelines: Public Preview, DBR 18.1.3 preview channel) | Yes (Databricks) | Yes | Yes |
| `spark.api.mode` (classic/connect) | 4.0 | — | Yes | Yes | Yes |
| `MERGE … WITH SCHEMA EVOLUTION` | 4.0 | DBR 15.2 | Yes | Yes | Yes |
| `DataFrameWriter(V2).clusterBy` | 4.0 | liquid clustering GA DBR 15.4 | Yes | Yes | Yes |
| `CLUSTER BY AUTO` (UC managed + predictive optimization) | — | DBR 15.4 | Yes | Yes | Yes |
| `ALTER TABLE … REPLACE PARTITIONED BY WITH CLUSTER BY` | — | DBR 18.1 | **No** | Yes | Yes |
| `OPTIMIZE … FULL WHERE` (partial recluster) | — | DBR 18.1 | No | Yes | Yes |
| `spark.sql.shuffle.partitions = auto` | — (Databricks only) | classic: opt-in; serverless: default | Yes | Yes | Yes |
| Spark Declarative Pipelines (`pyspark.pipelines`) | 4.1 | Lakeflow pipelines | see lakeflow KB | see lakeflow KB | see lakeflow KB |

## How to Check at Runtime

```python
print(spark.version)                                   # e.g. '4.0.0' on DBR 17.3 LTS
print(spark.conf.get("spark.sql.ansi.enabled"))        # 'true' on DBR 17.0+
```

```sql
SELECT version();          -- Spark version string
```

## Related

- [ansi-mode](../concepts/ansi-mode.md)
- [spark-connect](../concepts/spark-connect.md)
- [quick-reference](../quick-reference.md)

## Sources

- https://spark.apache.org/downloads.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://spark.apache.org/releases/spark-release-3-4-0.html
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://docs.databricks.com/aws/en/release-notes/runtime/18
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-pipeline
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-scripting
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-cte
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-collation
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/optimizations/aqe
- https://docs.databricks.com/aws/en/spark/conf
- https://docs.databricks.com/aws/en/stateful-applications/
- https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store
- https://docs.databricks.com/aws/en/ldp/real-time
- https://docs.databricks.com/aws/en/structured-streaming/real-time/setup
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.arrow_udf.html
- https://spark.apache.org/docs/4.0.0/app-dev-spark-connect.html
