# ANSI Mode

> **Purpose**: ANSI SQL mode is on by default in Spark 4.0+ and Databricks Runtime 17.0+ — what now fails, the `try_*` escape hatches, how to opt out, and the type-coercion trap in `union`/`when`/`coalesce`
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

`spark.sql.ansi.enabled` defaults to `true` in Apache Spark 4.0 and Databricks Runtime 17.0+
(so DBR 17.3 LTS, 18 LTS and 19 all run ANSI by default; serverless also defaults to `true`).
Invalid operations that used to return `NULL` or a wrapped value now **fail the query at runtime**,
and implicit type coercion follows a type-precedence list instead of Hive rules.
Store assignment (`spark.sql.storeAssignmentPolicy`, default `ANSI`) is a separate setting that
governs casts on `INSERT`.

| Runtime | ANSI default |
|---------|--------------|
| Spark 3.x / DBR ≤ 16.x | `false` (Hive-compatible: NULL / overflow wrap) |
| Spark 4.0+ / DBR 17.0+ (17.3 LTS, 18 LTS, 19) | `true` |
| Databricks SQL warehouses (`ANSI_MODE`) | `TRUE` for accounts created on/after 2022-10-19 |

## The Concept

```python
from pyspark.sql import functions as F

df = spark.createDataFrame([("10", 0), ("abc", 2)], "qty_txt STRING, boxes INT")

# ANSI ON (default): each of these raises at runtime when a bad row is reached
df.select(F.col("qty_txt").cast("int"))                    # [CAST_INVALID_INPUT] 'abc'
df.select(F.col("qty_txt").cast("int") / F.col("boxes"))   # [DIVIDE_BY_ZERO]
df.select(F.lit(2147483647) + F.lit(1))                     # [ARITHMETIC_OVERFLOW]
df.select(F.element_at(F.array(F.lit(1)), 5))               # index out of bounds

# Tolerant equivalents: return NULL instead of failing
df.select(
    F.col("qty_txt").try_cast("int").alias("qty"),                          # Column.try_cast: PySpark 4.0+
    F.try_divide(F.col("qty_txt").try_cast("int"), F.col("boxes")).alias("per_box"),
    F.try_element_at(F.array(F.lit(1)), F.lit(5)).alias("fifth"),
    F.try_to_number(F.lit("$78.12"), F.lit("$99.99")).alias("amount"),
)
```

```sql
SELECT try_cast(qty_txt AS INT), try_divide(a, b), try_add(a, b), try_to_timestamp(ts_txt)
FROM raw.orders;
```

## What Changes Under ANSI

| Operation | ANSI off (legacy) | ANSI on (Spark 4.0+ / DBR 17.0+) | Tolerant alternative |
|-----------|-------------------|----------------------------------|----------------------|
| `CAST('a' AS INT)` | `NULL` | `CAST_INVALID_INPUT` error | `try_cast` |
| `CAST(2147483648L AS INT)` | wraps to `-2147483648` | `CAST_OVERFLOW` error | `try_cast` |
| `INT + INT` overflow | wraps | `ARITHMETIC_OVERFLOW` | `try_add`, `try_subtract`, `try_multiply` |
| `x / 0`, `x % 0` | `NULL` | `DIVIDE_BY_ZERO` | `try_divide`, `try_mod` |
| `SUM`/`AVG` overflow | wraps / `NULL` | error | `try_sum`, `try_avg` |
| `arr[i]`, `element_at(arr, i)` out of range | `NULL` | error | `try_element_at` |
| `to_date`/`to_timestamp` unparsable | `NULL` | error | `try_to_timestamp` |
| `to_number` format mismatch | — | error | `try_to_number` |
| `make_date`/`make_timestamp`/`make_interval` invalid | `NULL` | error | `try_make_timestamp*`, `try_make_interval` |
| `size(NULL)` | `-1` | `NULL` | — |

All functions in the last column exist in Spark 4.0 SQL (DBR 17.3 LTS); PySpark 4.0 exposes them
in `pyspark.sql.functions` plus `Column.try_cast`.

## Type Coercion Trap: `union`, `when`/`CASE`, `coalesce`, `IN`

Under ANSI, the result type of `UNION`/`INTERSECT`/`EXCEPT`, `CASE`/`when`, `coalesce`, `least`,
`greatest` and `IN` is the **least common type** of the inputs. When one side is `STRING` and the
other numeric/date, the **string is cast to the other type** (BIGINT for integral types, DOUBLE for
DECIMAL/FLOAT, DATE/TIMESTAMP for dates) — not the other way round. A single non-numeric string then
fails the whole query.

```sql
SELECT typeof(coalesce(5, '6'));   -- BIGINT
SELECT coalesce('6.1', 5);         -- CAST_INVALID_INPUT ('6.1' is not a BIGINT)
SELECT typeof(coalesce(1BD, '6')); -- DOUBLE
```

```python
# Branch A: id is BIGINT; branch B: id is STRING with codes like 'A-17'
a.unionByName(b)                                   # resolves id to BIGINT -> fails on 'A-17'
F.when(F.col("src") == "a", F.col("id_num")).otherwise(F.col("id_txt"))  # same trap

# Fix: conform types explicitly in every branch, to the type you actually want
a.withColumn("id", F.col("id").cast("string")).unionByName(b)
```

## Opting Out (and Why Not To)

```python
spark.conf.set("spark.sql.ansi.enabled", "false")   # session / notebook (also settable on serverless)
```

```sql
SET spark.sql.ansi.enabled = false;   -- Databricks Runtime / Spark SQL session
SET ANSI_MODE = false;                -- Databricks SQL warehouse session
```

Prefer fixing the expression: legacy mode silently turns bad input into `NULL` or wrapped numbers,
which hides data-quality defects downstream. Use `try_*` only where a `NULL` is the intended
business outcome, and count/log the rows that became `NULL`.

## Common Mistakes

### Wrong

```python
# Blanket opt-out to "make the job pass" after upgrading to DBR 17.x
spark.conf.set("spark.sql.ansi.enabled", "false")
df = df.withColumn("qty", F.col("qty_txt").cast("int"))
```

### Correct

```python
df = df.withColumn("qty", F.col("qty_txt").try_cast("int"))
bad = df.filter(F.col("qty").isNull() & F.col("qty_txt").isNotNull()).count()
if bad:
    print(f"WARN: {bad} rows with non-numeric qty_txt")   # surface, don't hide
```

## Related

- [dataframe-api](../concepts/dataframe-api.md)
- [spark-connect](../concepts/spark-connect.md)
- [version-matrix](../reference/version-matrix.md)
- [cross-dialect SQL](../../sql-patterns/patterns/cross-dialect.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-datatype-rules
- https://docs.databricks.com/aws/en/sql/language-manual/parameters/ansi_mode
- https://docs.databricks.com/aws/en/release-notes/runtime/17.0
- https://docs.databricks.com/aws/en/spark/conf
- https://kb.databricks.com/dbsql/databricks-sql-ansi-enablement-guide
- https://spark.apache.org/docs/4.0.0/sql-ref-ansi-compliance.html
- https://spark.apache.org/docs/latest/sql-ref-ansi-compliance.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.Column.try_cast.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.functions.try_to_number.html
