# Spark Testing Patterns

> **Purpose**: Test PySpark 4.x transformations with a local SparkSession, `pyspark.testing` assertions and ANSI mode pinned to the target runtime
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Unit-testing DataFrame transformations, aggregations, joins and window logic
- Asserting output schemas (names, types, nullability) before a table write
- Locking in behavior that changed with Spark 4 ANSI mode (casts, division, overflow)
- Running the same tests locally and on Databricks Runtime 17.3 LTS (Spark 4.0.0)

## Implementation

```python
# tests/conftest.py
import pytest
from pyspark.sql import SparkSession


@pytest.fixture(scope="session")
def spark():
    session = (
        SparkSession.builder.master("local[2]")
        .appName("unit-tests")
        .config("spark.sql.ansi.enabled", "true")       # pin it: default since Spark 4.0 / DBR 17
        .config("spark.sql.session.timeZone", "UTC")    # deterministic timestamps
        .config("spark.sql.shuffle.partitions", "2")
        .config("spark.ui.enabled", "false")
        .getOrCreate()
    )
    yield session
    session.stop()
```

```python
# tests/test_invoices.py
import pytest
from pyspark.errors import PySparkException
from pyspark.sql import Row, functions as F
from pyspark.testing import assertDataFrameEqual, assertSchemaEqual

SCHEMA = "invoice_id STRING NOT NULL, vendor STRING, amount DOUBLE, status STRING"


def active_by_vendor(df):
    return (
        df.filter(F.col("status").isin("paid", "pending") & (F.col("amount") > 0))
        .groupBy("vendor")
        .agg(F.sum("amount").alias("total_amount"), F.count("*").alias("invoice_count"))
    )


def parse_amount(df):
    return df.withColumn("amount", F.col("raw").try_cast("double"))   # Column.try_cast: 4.0+


@pytest.fixture
def invoices(spark):
    rows = [
        ("INV-1", "Acme", 1500.0, "paid"),
        ("INV-2", "Beta", 2500.0, "pending"),
        ("INV-3", "Acme", 750.0, "paid"),
        ("INV-4", "Gamma", 0.0, "cancelled"),
    ]
    return spark.createDataFrame(rows, schema=SCHEMA)   # explicit schema, no inference


def test_active_by_vendor(invoices):
    expected = [
        Row(vendor="Acme", total_amount=2250.0, invoice_count=2),
        Row(vendor="Beta", total_amount=2500.0, invoice_count=1),
    ]
    # Row order is ignored by default (checkRowOrder=False); floats compared with rtol/atol
    assertDataFrameEqual(active_by_vendor(invoices), expected)


def test_output_schema(spark, invoices):
    expected = spark.createDataFrame(
        [], "vendor STRING, total_amount DOUBLE, invoice_count BIGINT NOT NULL"
    ).schema
    assertSchemaEqual(active_by_vendor(invoices).schema, expected)   # nullability ignored by default


def test_invalid_cast_fails_under_ansi(spark):
    df = spark.createDataFrame([("1.5",), ("n/a",)], "raw STRING")
    with pytest.raises(PySparkException) as exc:
        df.select(F.col("raw").cast("double")).collect()
    assert exc.value.getCondition() == "CAST_INVALID_INPUT"


def test_try_cast_returns_null(spark):
    df = spark.createDataFrame([("1.5",), ("n/a",)], "raw STRING")
    assertDataFrameEqual(parse_amount(df).select("amount"), [Row(amount=1.5), Row(amount=None)])
```

## Configuration

| Setting | Value | Purpose |
|---------|-------|---------|
| `spark.sql.ansi.enabled` | same as the target (`true` on Spark 4 / DBR 17.3 LTS) | casts, `/ 0` and overflow raise instead of returning NULL |
| `spark.sql.session.timeZone` | `UTC` | stable timestamp tests across machines |
| `spark.sql.shuffle.partitions` | `2` | fast local shuffles |
| `spark.ui.enabled` | `false` | no UI threads/ports in CI |
| `master` | `local[2]` | two local cores |

## pyspark.testing Reference (PySpark 3.5+, extra flags 4.0+)

| Call | Key options |
|------|-------------|
| `assertDataFrameEqual(actual, expected)` | `checkRowOrder=False`, `rtol=1e-05`, `atol=1e-08`, `ignoreNullable=True`; 4.0+: `ignoreColumnOrder`, `ignoreColumnName`, `ignoreColumnType`, `maxErrors`, `showOnlyDiff`, `includeDiffRows` |
| `assertSchemaEqual(actual, expected)` | 4.0+: `ignoreNullable=True`, `ignoreColumnOrder=False`, `ignoreColumnName=False` |

`expected` may be a DataFrame, a pandas/pandas-on-Spark DataFrame or a list of `Row`s; the schema
is compared only when `expected` is a DataFrame. Failures print a `difflib` diff. `chispa` (0.12+
supports Spark 4) is an optional third-party alternative. When pandas is installed the assertion
imports `pyspark.pandas`: pin pandas like the runtime (DBR 17.3 LTS ships 2.2.3). PySpark 4.1.1 with
pandas 3.0.1 failed here with `ImportError` (`_builtin_table`); PySpark 4.0.4 + pandas 2.2.3 passed.

## ANSI Mode in Spark 4

| Expression | ANSI on (Spark 4 default) | ANSI off (Spark 3 default) | Tolerant form |
|------------|---------------------------|----------------------------|---------------|
| `cast("n/a" AS DOUBLE)` | error `CAST_INVALID_INPUT` | NULL | `try_cast` / `Column.try_cast` (4.0+) |
| `1 / 0` | error `DIVIDE_BY_ZERO` | NULL | `F.try_divide` (3.5+) |

Errors surface on the **action** (`collect`, `count`, write), so assert inside the
`pytest.raises` block. Details: [ANSI mode](../../spark/concepts/ansi-mode.md).

## Example Usage

```bash
# Local runtime: PySpark 4.x needs Java 17+ (DBR 17.3 LTS: Java 17); Python 3.9+ (4.0), 3.10+ (latest)
export JAVA_HOME=/path/to/jdk-17
uv add --group test "pyspark==4.0.*" "pandas==2.2.3" pyarrow "pytest>=8.3"   # match DBR 17.3 LTS
uv run pytest tests/ -m "not slow" -ra

# Inside a Databricks notebook (docs: skip .pyc and pytest cache on the read-only filesystem)
#   import sys, pytest; sys.dont_write_bytecode = True
#   pytest.main([".", "-v", "-p", "no:cacheprovider"])
```

## See Also

- [Fixtures](../concepts/fixtures.md)
- [Fixture Factories](../patterns/fixture-factories.md)
- [Spark ANSI mode](../../spark/concepts/ansi-mode.md)

## Sources

- https://spark.apache.org/docs/latest/api/python/reference/pyspark.testing.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/api/pyspark.testing.assertDataFrameEqual.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/api/pyspark.testing.assertSchemaEqual.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.Column.try_cast.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/api/pyspark.errors.PySparkException.getCondition.html
- https://spark.apache.org/docs/latest/sql-migration-guide.html
- https://spark.apache.org/docs/4.0.0/api/python/getting_started/install.html
- https://spark.apache.org/docs/latest/api/python/getting_started/install.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://docs.databricks.com/aws/en/notebooks/testing
- https://github.com/MrPowers/chispa/releases
