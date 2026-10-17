# Gold Aggregation Patterns

> **Purpose**: Business aggregates, denormalized views and time series as materialized views that refresh incrementally
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- KPIs by day, month, region or product consumed by BI, apps or other pipelines
- Denormalized "360" tables that join silver entities
- Rolling and period-over-period metrics

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Daily sales by store and channel
@dp.materialized_view(
    name="daily_sales",
    comment="Daily sales metrics",
    table_properties={"quality": "gold"},
    cluster_by=["sale_date"],
)
@dp.expect_all_or_fail({"non_negative_total": "total_amount >= 0"})
def daily_sales():
    return (spark.read.table("orders_silver")
            .groupBy(F.col("order_date").alias("sale_date"), "store_id", "channel")
            .agg(F.count("*").alias("order_count"),
                 F.sum("amount").alias("total_amount"),          # amount is DECIMAL(18,2)
                 F.countDistinct("customer_id").alias("unique_customers")))

# Customer 360: join entities with lifetime aggregates
@dp.materialized_view(name="customer_360", cluster_by_auto=True)
def customer_360():
    customers = spark.read.table("customers_silver")
    lifetime = (spark.read.table("orders_silver")
                .groupBy("customer_id")
                .agg(F.count("*").alias("lifetime_orders"),
                     F.sum("amount").alias("lifetime_value"),
                     F.max("order_date").alias("last_order_date")))
    return customers.join(lifetime, "customer_id", "left")

# Rolling 7-day average per store (window with PARTITION BY)
@dp.materialized_view(name="store_rolling_7d")
def store_rolling_7d():
    w = Window.partitionBy("store_id").orderBy("sale_date").rowsBetween(-6, 0)
    return (spark.read.table("daily_sales")
            .withColumn("rolling_7d_amount", F.avg("total_amount").over(w)))

# Month over month
@dp.materialized_view(name="store_mom")
def store_mom():
    monthly = (spark.read.table("daily_sales")
               .groupBy(F.date_trunc("month", "sale_date").alias("month"), "store_id")
               .agg(F.sum("total_amount").alias("amount")))
    w = Window.partitionBy("store_id").orderBy("month")
    return (monthly.withColumn("prev_amount", F.lag("amount").over(w))
            .withColumn("mom_pct", (F.col("amount") - F.col("prev_amount")) / F.col("prev_amount") * 100))

# Intermediate logic that does not need to be published
@dp.temporary_view
def active_stores():
    return spark.read.table("main.ref.stores").where("status = 'ACTIVE'")
```

## Configuration

| Goal | Choose | Decorator / SQL |
|------|--------|-----------------|
| Pre-computed aggregate | Materialized view | `@dp.materialized_view` / `CREATE OR REFRESH MATERIALIZED VIEW` |
| Append-only incremental output | Streaming table | `@dp.table` + `spark.readStream` |
| Intermediate, unpublished | Temporary view | `@dp.temporary_view` |
| Persisted but unpublished | Private MV | `@dp.materialized_view(private=True)` |

| Incremental refresh tip | Detail |
|-------------------------|--------|
| Serverless only | Classic-compute pipelines always fully recompute MVs |
| Decimal, not float | `SUM`/`AVG` over `FLOAT`/`DOUBLE` can force full refresh; cast to `DECIMAL` |
| Window functions | Need `PARTITION BY` to be incremental |
| Date filters | `>= :start` stays incremental; a two-sided range forces full refresh |
| Source tables | Enable deletion vectors, row tracking and change data feed |
| Guarantee | `refresh_policy="incremental_strict"` fails instead of falling back to full |

## Example Usage

```python
# Wrong: a temporary view is not a materialized view (not persisted, not queryable outside)
@dp.temporary_view(name="current_store_status")
def current_store_status():
    return spark.read.table("main.ref.stores")

# Correct: persist and publish with a materialized view
@dp.materialized_view(name="current_store_status")
def current_store_status():
    return spark.read.table("main.ref.stores").where("status = 'ACTIVE'")
```

Gold quality: fail only on invariants that would make the published numbers wrong.

## See Also

- [Silver cleansing](silver-cleansing.md)
- [Materialized views reference](../reference/materialized-views.md)
- [DABs deployment](dabs-deployment.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-materialized-view
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-view
- https://docs.databricks.com/aws/en/ldp/concepts/materialized-views
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/concepts
