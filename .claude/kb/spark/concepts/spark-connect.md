# Spark Connect

> **Purpose**: Client-server Spark (gRPC) — where it is used (Databricks standard access mode, serverless, Databricks Connect), and what changes in your code versus Spark Classic
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Spark Connect separates the client from the Spark driver: the client builds **unresolved** logical
plans and sends them over gRPC to a Spark Connect server, which analyzes and executes them. It
exists since Spark 3.4 (Python) / 3.5 (Scala); Spark 4.0 added the 1.5 MB `pyspark-client`
package, the `spark.api.mode` switch and ML on Connect. Connect supports most of the DataFrame,
Column and Functions APIs, but **SparkContext and RDD APIs are not supported**. Official clients are
PySpark and Scala; Go, Swift and Rust clients are separate Apache subprojects (the Go README calls it
highly experimental and not for production).

| Where your code runs | Engine mode |
|----------------------|-------------|
| Databricks Python notebooks on **standard** access mode, DBR 14.3+ (Scala: 13.3+) | Spark Connect |
| Databricks **serverless** notebooks/jobs | Spark Connect |
| Databricks Connect (IDE / local process) | Spark Connect |
| Lakeflow pipelines with environment versions configured | Spark Connect |
| Databricks **dedicated** access mode | Spark Classic |
| OSS `pyspark` with `.remote(...)` or `spark.api.mode=connect` | Spark Connect |

## The Concept

```python
# OSS: thin client (pip install pyspark-client==<server version>)
from pyspark.sql import SparkSession
spark = SparkSession.builder.remote("sc://spark-server:15002").getOrCreate()

# OSS full pyspark: choose the mode when the session is CREATED (not with spark.conf.set later)
spark = (SparkSession.builder
    .config("spark.api.mode", "connect")      # or "classic"; Spark 4.0+
    .master("local[*]")
    .getOrCreate())
# spark-submit --conf spark.api.mode=connect ...
```

## Lazy Analysis: The Behaviour Change That Bites

| Aspect | Spark Classic | Spark Connect |
|--------|---------------|---------------|
| Query execution | Lazy | Lazy |
| Schema analysis (name/type resolution) | Eager, at each transformation | Deferred until an action or schema access |
| `df.columns`, `df.schema`, `df.isStreaming` | Local | RPC to the server (cached per DataFrame) |
| Temporary views referenced by a DataFrame | Plan embedded at creation | Looked up **by name** at execution |
| Python UDF serialization | At creation | At execution (captures variables' *current* value) |

```python
df = spark.sql("select 1 as a, 2 as b").filter("c > 1")   # Classic: error here
df.columns                                                  # Connect: error only here (or on show())
```

## Quick Reference

| Need | Do on Spark Connect |
|------|---------------------|
| Catch analysis errors early | Touch `df.columns`/`df.schema` inside the `try` |
| Loop adding many columns | Track names in a Python `set`; don't call `df.columns` each iteration |
| Temp views in helper functions | Use unique names (e.g. `f"tmp_{uuid4().hex}"`) |
| UDF depending on an outer variable | Bind it via a factory function (`make_udf(value)`) |
| Data size / partition count | Use SQL/Delta metadata (`DESCRIBE DETAIL`), not `sparkContext` internals |
| RDD code (`sc.parallelize`, `df.rdd`) | Rewrite with DataFrame APIs (RDD APIs unsupported on standard mode and serverless) |
| Local data → DataFrame | `spark.createDataFrame(...)`; on standard mode each row must be < 128 MB |

## Common Mistakes

### Wrong

```python
# py4j/JVM internals do not exist on Spark Connect
n = spark.sparkContext._jsc.sc().getExecutorMemoryStatus().size()
plan = df._jdf.queryExecution().toString()

# Switching mode on an already-running session
spark.conf.set("spark.api.mode", "connect")

# UDF silently picks up the latest value of x on Connect
x = 123
@F.udf("int")
def foo(): return x
df = spark.range(1).select(foo()); x = 456; df.show()   # prints 456 on Connect
```

### Correct

```python
size_bytes = spark.sql("DESCRIBE DETAIL main.sales.orders").first()["sizeInBytes"]
df.explain(mode="formatted")                 # plan inspection without JVM handles

def make_udf(value):
    return F.udf(lambda: value, "int")
foo = make_udf(123)                          # value bound at creation on both engines
```

## Related

- [dataframe-api](../concepts/dataframe-api.md)
- [ansi-mode](../concepts/ansi-mode.md)
- [performance-tuning](../patterns/performance-tuning.md)
- [version-matrix](../reference/version-matrix.md)

## Sources

- https://spark.apache.org/docs/latest/spark-connect-overview.html
- https://spark.apache.org/docs/4.0.0/app-dev-spark-connect.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://docs.databricks.com/aws/en/spark/connect-vs-classic
- https://docs.databricks.com/aws/en/compute/standard-limitations
- https://docs.databricks.com/aws/en/compute/serverless/limitations
- https://docs.databricks.com/aws/en/delta/table-details
