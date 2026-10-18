# Data Quality Expectations

> **Purpose**: Apply warn/drop/fail expectations to pipeline datasets in Python and SQL
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Tracking data quality without blocking ingestion (warn)
- Removing records that break technical rules before they reach consumers (drop)
- Stopping an update when a critical invariant is violated (fail)

## Implementation

```python
from pyspark import pipelines as dp

# WARN (default): invalid rows are written, metrics are recorded
@dp.table
@dp.expect("valid_age", "age >= 0")
def customers_raw():
    return spark.readStream.table("main.raw.customers")

# DROP: invalid rows are removed before the write; drop count is logged
@dp.table
@dp.expect_or_drop("valid_email", "email LIKE '%@%.%'")
def customers_clean():
    return spark.readStream.table("customers_raw")

# FAIL: the update fails and the transaction rolls back
@dp.materialized_view
@dp.expect_or_fail("has_customer_id", "customer_id IS NOT NULL")
def customers_gold():
    return spark.read.table("customers_clean")

# Several decorators stack on one dataset
@dp.table
@dp.expect_or_drop("valid_id", "customer_id IS NOT NULL")
@dp.expect_or_drop("valid_age", "age > 0 AND age < 150")
@dp.expect("recent", "updated_at >= '2020-01-01'")
def customers_checked():
    return spark.readStream.table("customers_raw")
```

```sql
CREATE OR REFRESH STREAMING TABLE customers_checked (
  CONSTRAINT valid_id   EXPECT (customer_id IS NOT NULL)  ON VIOLATION DROP ROW,
  CONSTRAINT valid_age  EXPECT (age > 0 AND age < 150)    ON VIOLATION DROP ROW,
  CONSTRAINT recent     EXPECT (updated_at >= '2020-01-01')
) AS SELECT * FROM STREAM(customers_raw);

CREATE OR REFRESH MATERIALIZED VIEW customers_gold (
  CONSTRAINT has_customer_id EXPECT (customer_id IS NOT NULL) ON VIOLATION FAIL UPDATE
) AS SELECT * FROM customers_checked;
```

## Configuration

| Action | Python | SQL | Result |
|--------|--------|-----|--------|
| warn | `@dp.expect(name, expr)` | `CONSTRAINT n EXPECT (expr)` | Rows kept, metrics recorded |
| drop | `@dp.expect_or_drop(name, expr)` | `... ON VIOLATION DROP ROW` | Rows dropped, count logged |
| fail | `@dp.expect_or_fail(name, expr)` | `... ON VIOLATION FAIL UPDATE` | Update fails; no metrics recorded |

Constraint expressions can use comparisons, built-in SQL functions, `CASE`, and boolean logic. They
cannot call Python UDFs or external services, or run subqueries against other tables.

## Example Usage

```python
@dp.table
@dp.expect("valid_order_status", """
    CASE
      WHEN type = 'ORDER'  THEN status IN ('PENDING', 'COMPLETED', 'CANCELLED')
      WHEN type = 'REFUND' THEN status IN ('PENDING', 'APPROVED', 'REJECTED')
      ELSE false
    END
""")
@dp.expect("valid_dates", "start_date <= end_date AND end_date <= current_date()")
def orders_checked():
    return spark.readStream.table("main.raw.orders")
```

Error text for a failed update looks like
`[EXPECTATION_VIOLATION.VERBOSITY_ALL] Flow '...' failed to meet the expectation. Violated
expectations: '...'. Input data: '{...}'`. Fix the code or data before re-running.

## See Also

- [Advanced expectations](expectations-advanced.md)
- [Expectations model](../concepts/expectations-model.md)
- [Python decorators](python-decorators.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/developer/python-ref
