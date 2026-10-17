# Python Decorators for Lakeflow Pipelines

> **Purpose**: Define streaming tables, materialized views and views with `pyspark.pipelines` decorators and their parameters
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Writing pipeline source files in Python (`transformations/*.py`)
- Choosing between `@dp.table`, `@dp.materialized_view` and `@dp.temporary_view`
- Setting clustering, schema, privacy, refresh policy or row filters on a dataset
- Generating many datasets in a loop

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

# Streaming table: function returns a streaming DataFrame
@dp.table(
    name="orders_bronze",
    comment="Raw orders",
    table_properties={"quality": "bronze"},
    cluster_by_auto=True,                      # automatic liquid clustering
)
def orders_bronze():
    return spark.readStream.table("main.raw.orders")

# Materialized view: function returns a batch DataFrame
@dp.materialized_view(
    comment="Daily revenue",
    cluster_by=["order_date"],
    refresh_policy="incremental",              # auto | incremental | incremental_strict | full
)
def daily_revenue():
    return (spark.read.table("orders_bronze")
            .groupBy("order_date")
            .agg(F.sum("amount").alias("revenue")))

# Temporary view: computed when queried, not published
@dp.temporary_view
def orders_open():
    return spark.read.table("orders_bronze").where("status = 'OPEN'")

# Private table: persisted for the pipeline lifetime, not published to the catalog
@dp.materialized_view(private=True)
def orders_dedup_keys():
    return spark.read.table("orders_bronze").select("order_id").distinct()

# Explicit schema with a generated column (DDL string or StructType)
@dp.table(schema="""
    order_id BIGINT, amount DECIMAL(18,2), order_ts TIMESTAMP,
    order_date DATE GENERATED ALWAYS AS (CAST(order_ts AS DATE))
""")
def orders_typed():
    return spark.readStream.table("orders_bronze").select("order_id", "amount", "order_ts")

# Fully qualified name writes outside the pipeline default catalog/schema
@dp.materialized_view(name="main.reporting.regional_sales")
def regional_sales():
    return spark.read.table("daily_revenue")
```

### Loops: bind the loop variable

```python
tables = ["orders", "customers", "products"]

for t in tables:
    @dp.materialized_view(name=f"{t}_snapshot")
    def snapshot(source=t):                  # default argument captures the current value
        return spark.read.table(f"main.raw.{source}")
```

Keep the list additive: a dataset removed from the list is removed from the pipeline graph.

## Configuration

| Parameter | `@dp.table` | `@dp.materialized_view` | Notes |
|-----------|-------------|-------------------------|-------|
| `name` | yes | yes | Defaults to function name; may be fully qualified |
| `comment`, `table_properties`, `spark_conf` | yes | yes | `spark_conf` is per dataset |
| `schema` | yes | yes | DDL string or `StructType` |
| `cluster_by`, `cluster_by_auto` | yes | yes | Preferred over `partition_cols` |
| `partition_cols` | yes | yes | Mutually exclusive with clustering |
| `private` | yes | yes | Replaces legacy `temporary` |
| `row_filter` | yes | yes | Public Preview |
| `refresh_policy` | - | yes | Default `auto` |
| `replace_using`, `sequence_by` | yes | - | Beta, `REPLACE USING` flow |
| `path` | yes | yes | Leave unset in Unity Catalog pipelines: tables use managed storage and `LOCATION` is not supported |

`@dp.temporary_view` takes `name` and `comment`. Expectation decorators (`@dp.expect*`) stack on
all three. `@dp.table` / `@dp.materialized_view` also work outside a pipeline in a serverless
notebook to define standalone tables (Beta, Sep 2026).

## Example Usage

```python
# Coding rules from the Python reference
# - Return a DataFrame; no side effects in dataset functions.
# - Never call collect(), count(), toPandas(), save(), saveAsTable(), start(), toTable().
# - Read pipeline datasets with spark.read.table / spark.readStream.table.

@dp.materialized_view
def bad():
    df = spark.read.table("orders_bronze")
    print(df.count())          # Wrong: triggers execution during graph planning
    return df
```

## See Also

- [Python streaming](python-streaming.md)
- [Flows and sinks](flows-and-sinks.md)
- [Python module migration](../concepts/python-module-migration.md)
- [Expectations](expectations.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/developer/python-dev
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-table
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-materialized-view
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-view
- https://docs.databricks.com/aws/en/ldp/target-schema
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
