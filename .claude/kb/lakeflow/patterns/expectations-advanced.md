# Advanced Data Quality Expectations

> **Purpose**: Grouped and reusable expectations, multi-layer strategy, referential checks, and expectation metrics from the event log
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- The same rule set applies to several datasets
- You need pass/fail counts per expectation for monitoring or alerting
- Referential integrity must be checked (expectations cannot run subqueries)

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

ORDER_RULES = {
    "valid_id": "order_id IS NOT NULL",
    "valid_date": "order_date IS NOT NULL",
    "positive_amount": "amount > 0",
    "valid_status": "status IN ('PENDING', 'COMPLETED', 'CANCELLED')",
}

# Bronze: warn only
@dp.table(comment="Raw orders", table_properties={"quality": "bronze"})
@dp.expect("no_rescued_data", "_rescued_data IS NULL")
def orders_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load("/Volumes/main/landing/files/orders/"))

# Silver: grouped drop rules (Python-only grouping)
@dp.table(comment="Clean orders", table_properties={"quality": "silver"})
@dp.expect_all_or_drop(ORDER_RULES)
def orders_silver():
    return spark.readStream.table("orders_bronze")

# Referential check: join first, then expect on the joined column
@dp.temporary_view
@dp.expect_or_drop("known_customer", "c_id IS NOT NULL")
def orders_with_customer():
    orders = spark.read.table("orders_silver")
    customers = spark.read.table("main.crm.customers").select(F.col("customer_id").alias("c_id"))
    return orders.join(customers, orders.customer_id == customers.c_id, "left")

# Gold: fail on critical invariants
@dp.materialized_view(comment="Daily revenue", table_properties={"quality": "gold"})
@dp.expect_all_or_fail({"non_negative_revenue": "revenue >= 0"})
def daily_revenue():
    return (spark.read.table("orders_with_customer")
            .groupBy(F.to_date("order_date").alias("day"))
            .agg(F.sum("amount").alias("revenue")))

# Same rules as a parameter of create_streaming_table (targets of flows)
dp.create_streaming_table("orders_archive", expect_all_or_drop=ORDER_RULES)
```

## Configuration

| Function | Action | Input |
|----------|--------|-------|
| `@dp.expect_all(dict)` | warn | `{name: expr}` |
| `@dp.expect_all_or_drop(dict)` | drop | `{name: expr}` |
| `@dp.expect_all_or_fail(dict)` | fail | `{name: expr}` |
| `dp.create_streaming_table(..., expect_all*=dict)` | per parameter | Same semantics as decorators |

Expectations are supported on streaming tables, materialized views and temporary views. Metrics may
be missing when no expectation is defined, the flow type does not support them (sinks), there were
no updates, or the dataset is a view that was not queried (views queried by several datasets can
report several metric sets). A materialized view that reads from a view with expectations is not
incrementally refreshed, so put heavy gold aggregations on a table, not on such a view.

## Example Usage

```sql
-- Expectation metrics for the latest update (event_log_raw is a view over the pipeline event log,
-- e.g. CREATE VIEW event_log_raw AS SELECT * FROM event_log('<pipeline-id>'))
WITH latest_update AS (
  SELECT origin.update_id AS id FROM event_log_raw
  WHERE event_type = 'create_update' ORDER BY timestamp DESC LIMIT 1
)
SELECT e.dataset, e.name AS expectation,
       SUM(e.passed_records) AS passed, SUM(e.failed_records) AS failed
FROM (
  SELECT explode(from_json(details:flow_progress:data_quality:expectations,
         'array<struct<name: string, dataset: string, passed_records: int, failed_records: int>>')) AS e
  FROM event_log_raw, latest_update
  WHERE event_type = 'flow_progress' AND origin.update_id = latest_update.id
)
GROUP BY e.dataset, e.name;
```

Dropped-row totals are in `details:flow_progress.data_quality.dropped_records`. The event log is also
available as the Beta system table `system.lakeflow_pipeline_events_preview.pipeline_events`.

Practices: name rules after the business rule, keep FAIL rare, review warn metrics regularly, and keep
expressions simple. Avoid reading downstream tables from upstream rules (cycles).

## See Also

- [Expectations](expectations.md)
- [Expectations model](../concepts/expectations-model.md)
- [Silver cleansing](silver-cleansing.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/monitor-event-logs
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-streaming-table
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
