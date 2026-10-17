# DataFrame API

> **Purpose**: Core PySpark transformations — select, filter, groupBy, join, Column expressions, UDF avoidance, plus Spark 4.x additions (VARIANT, Python Data Source API, plotting) with version notes
> **Confidence**: 0.93
> **MCP Validated**: 2026-10-17

## Overview

DataFrames are lazily evaluated: transformations build a logical plan that Catalyst optimizes before
execution. Column expressions (`F.col`, `F.lit`, `F.when`) are preferred over Python UDFs. Two Spark 4
defaults change how code behaves: **ANSI mode** (invalid casts/arithmetic fail instead of returning
`NULL` — see [ansi-mode](ansi-mode.md)) and, on Databricks standard/serverless compute, **Spark
Connect** (analysis errors surface at the action — see [spark-connect](spark-connect.md)).

## The Concept

```python
from pyspark.sql import DataFrame
from pyspark.sql import functions as F
from pyspark.sql.types import DecimalType

def transform_orders(df: DataFrame) -> DataFrame:
    """Clean and enrich raw orders (ANSI-safe casts)."""
    return (
        df
        .filter(F.col("status").isin("completed", "shipped"))
        .withColumn("amount", F.col("amount_txt").try_cast(DecimalType(12, 2)))   # PySpark 4.0+
        .withColumn("amount_usd", F.col("amount") * F.col("exchange_rate"))
        .withColumn("order_date", F.to_date("created_at"))
        .select("order_id", "customer_id", "amount_usd", "order_date", "status")
        .dropDuplicates(["order_id"])
    )

# On Databricks the `spark` session already exists; outside Databricks:
# spark = SparkSession.builder.appName("orders_transform").getOrCreate()
orders = transform_orders(spark.read.table("main.sales.orders_raw"))
```

## Quick Reference

| Operation | Method | Notes |
|-----------|--------|-------|
| Select columns | `.select("col1", F.col("col2"))` | Strings or Column objects |
| Filter rows | `.filter(F.col("x") > 10)` | Alias: `.where()` |
| Add column | `.withColumn("new", expr)` | Many columns at once: `.withColumns({...})` |
| Rename | `.withColumnRenamed("old", "new")` | |
| Group + agg | `.groupBy("key").agg(F.sum("val"))` | Chain multiple aggs |
| Join | `.join(other, on="key", how="left")` | inner, left, right, full, semi, anti |
| Lateral join | `df.lateralJoin(...)` | Spark 4.0+ DataFrame form of SQL `LATERAL` |
| Sort | `.orderBy(F.col("x").desc_nulls_last())` | |
| Distinct | `.dropDuplicates(["col"])` | `.distinct()` for all columns |
| Safe cast | `F.col("c").try_cast("int")` | `NULL` on bad input instead of an ANSI error |

## Common Mistakes

### Wrong

```python
# Python UDF for a simple string operation — row-at-a-time Python, opaque to Catalyst,
# and forces a Photon fallback on Databricks
@F.udf("string")
def upper_name(name):
    return name.upper() if name else None

df = df.withColumn("name_upper", upper_name("name"))
```

### Correct

```python
df = df.withColumn("name_upper", F.upper(F.col("name")))   # built-in, optimized
```

## Spark 4.x Additions

### VARIANT (semi-structured data)

Spark 4.0 adds the VARIANT type (GA/default in 4.1); Databricks supports it from DBR 15.4.
In PySpark, use `parse_json` + `variant_get`/`try_variant_get` (Spark 4.0+). `F.col("data:name")`
does **not** parse a path — `F.col` takes a column name.

```python
df = spark.sql("""SELECT parse_json('{"name": "Alice", "scores": [90, 85]}') AS data""")

df.select(
    F.variant_get("data", "$.name", "string").alias("name"),
    F.variant_get("data", "$.scores[0]", "int").alias("first_score"),
    F.try_variant_get("data", "$.missing", "int").alias("missing"),   # NULL, no error
)
```

```sql
-- Databricks SQL / DBR 15.4+ (OSS Spark 4.1+): colon path syntax with :: cast
SELECT data:name::string AS name, data:scores[0]::int AS first_score FROM events;
```

VARIANT columns cannot be clustering/partition/Z-order keys and cannot be used in comparisons,
grouping, ordering or set operations (Databricks).

### Python Data Source API (4.0+)

```python
from pyspark.sql.datasource import DataSource, DataSourceReader

class MyAPISource(DataSource):
    @classmethod
    def name(cls): return "my_api"
    def schema(self): return "data string"
    def reader(self, schema): return MyAPIReader()   # implements read(partition)

spark.dataSource.register(MyAPISource)
df = spark.read.format("my_api").load()
# Spark 4.1 adds filter pushdown for Python data sources (not in DBR 17.3 / Spark 4.0).
```

### Plotting API (4.0+, experimental)

```python
df.plot.bar(x="category", y="revenue")   # Plotly backend
```

`DataFrame.plot` is **not distributed**: it collects the top N rows
(`spark.sql.pyspark.plotting.max_rows`, default 1000) or a sample to the driver and plots with pandas.
Aggregate first, then plot.

## Related

- [ansi-mode](../concepts/ansi-mode.md)
- [spark-connect](../concepts/spark-connect.md)
- [partitioning](../concepts/partitioning.md)
- [python-udfs](../patterns/python-udfs.md)
- [window-functions](../patterns/window-functions.md)
- [version-matrix](../reference/version-matrix.md)

## Sources

- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.functions.variant_get.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.Column.try_cast.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.plot.html
- https://spark.apache.org/docs/latest/api/python/_modules/pyspark/sql/plot/core.html
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://docs.databricks.com/aws/en/sql/language-manual/functions/colonsign
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://docs.databricks.com/aws/en/compute/photon
