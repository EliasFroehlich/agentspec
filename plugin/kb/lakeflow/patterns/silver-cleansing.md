# Silver Cleansing Patterns

> **Purpose**: Typed, validated, de-duplicated silver datasets: casting, PII handling, bounded-state dedup, keep-latest
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Turning bronze rows (often strings) into typed columns with explicit rules
- Removing or masking PII before data reaches consumers
- De-duplicating a stream or keeping only the latest version of each key

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F
from pyspark.sql.types import DecimalType
from pyspark.sql.window import Window

# Typed silver stream with drop rules
@dp.table(name="orders_silver", comment="Typed, validated orders",
          table_properties={"quality": "silver"}, cluster_by=["order_date"])
@dp.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
@dp.expect_or_drop("valid_status", "status IN ('NEW', 'PAID', 'SHIPPED', 'CANCELLED')")
@dp.expect("has_customer", "customer_id IS NOT NULL")
def orders_silver():
    return (spark.readStream.table("orders_bronze")
            .select(
                F.col("order_id").cast("bigint"),
                F.col("customer_id").cast("bigint"),
                F.upper(F.trim("status")).alias("status"),
                F.col("amount").cast(DecimalType(18, 2)),
                F.to_timestamp("order_ts", "yyyy-MM-dd HH:mm:ss").alias("order_ts"),
                F.to_date("order_ts").alias("order_date"),
                "_source_file", "_ingested_at",
            ))

# PII: drop or mask before publishing
@dp.table(name="customers_silver", comment="Customers without raw PII")
def customers_silver():
    return (spark.readStream.table("customers_bronze")
            .withColumn("tax_id_masked",
                        F.concat(F.lit("***-**-"), F.substring("tax_id", -4, 4)))
            .withColumn("email_hash", F.sha2(F.lower("email"), 256))
            .drop("tax_id", "email", "phone"))

# Streaming dedup with bounded state: watermark + dropDuplicatesWithinWatermark
@dp.table(name="events_dedup")
def events_dedup():
    return (spark.readStream.table("events_bronze")
            .withWatermark("event_ts", "1 hour")
            .dropDuplicatesWithinWatermark(["event_id"]))

# Keep the latest row per key: batch window in a materialized view
@dp.materialized_view(name="customers_latest")
def customers_latest():
    w = Window.partitionBy("customer_id").orderBy(F.desc("_ingested_at"))
    return (spark.read.table("customers_silver")
            .withColumn("_rn", F.row_number().over(w))
            .where("_rn = 1").drop("_rn"))

```

## Configuration

| Need | Pattern | Why |
|------|---------|-----|
| Cast strings | `.cast(...)`, `to_timestamp`, `to_date` | Explicit types; ANSI mode errors on bad casts (use `try_cast` when bad values are expected) |
| Validate | `@dp.expect_or_drop` / `@dp.expect` | Drop technical garbage, warn on soft rules |
| Dedup a stream | `withWatermark` + `dropDuplicatesWithinWatermark` | `dropDuplicates` on a stream without a watermark keeps unbounded state |
| Latest per key, append feed | Materialized view with `row_number()` | Window functions refresh incrementally only with `PARTITION BY` |
| Latest per key, CDC feed | `AUTO CDC ... STORED AS SCD TYPE 1` | Handles out-of-order events and deletes |
| Column masking for readers | UC column masks in the table definition | Applied at query time with the reader's identity |

## Example Usage

```python
# Wrong: unbounded streaming state
spark.readStream.table("events_bronze").dropDuplicates(["event_id"])

# Correct
(spark.readStream.table("events_bronze")
    .withWatermark("event_ts", "1 hour")
    .dropDuplicatesWithinWatermark(["event_id"]))
```

| Source type | Target type | Expression |
|-------------|-------------|------------|
| String amount | Decimal | `F.col("amount").cast(DecimalType(18, 2))` |
| String date | Date | `F.to_date("d", "yyyy-MM-dd")` |
| String timestamp | Timestamp | `F.to_timestamp("ts", "yyyyMMddHHmmss")` |
| Null default | Same | `F.coalesce("c", F.lit(0))` |

## See Also

- [Bronze ingestion](bronze-ingestion.md)
- [Gold aggregation](gold-aggregation.md)
- [Stateful processing](../reference/stateful-processing.md)
- [Advanced expectations](expectations-advanced.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/stateful-processing
- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
