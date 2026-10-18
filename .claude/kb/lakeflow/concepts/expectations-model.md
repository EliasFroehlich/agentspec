# Expectations Model

> **Purpose**: Data quality constraints on Lakeflow pipeline datasets: components, actions, scope, limits
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

Expectations are optional clauses on streaming tables, materialized views and temporary views that
evaluate a SQL boolean per record. The action decides what happens on violation: warn (default),
drop the row, or fail the update. Metrics for warn/drop appear in the Data quality tab and the
event log. Standalone streaming tables and materialized views in Databricks SQL also accept
`CONSTRAINT ... EXPECT`. Expectations need the `Advanced` edition (serverless pipelines always are).

## The Concept

```python
from pyspark import pipelines as dp

@dp.table
@dp.expect("valid_ts", "event_ts IS NOT NULL")                 # warn: keep row, log metric
@dp.expect_or_drop("valid_email", "email LIKE '%@%.%'")        # drop row
@dp.expect_or_fail("valid_pk", "id IS NOT NULL")               # fail the update
def customers():
    return spark.readStream.table("main.raw.customers")

checks = {"valid_id": "id IS NOT NULL", "positive_amount": "amount > 0"}

@dp.materialized_view
@dp.expect_all_or_drop(checks)        # also expect_all, expect_all_or_fail
def orders_clean():
    return spark.read.table("main.raw.orders")
```

```sql
CREATE OR REFRESH STREAMING TABLE orders (
  CONSTRAINT valid_amount   EXPECT (amount > 0)              ON VIOLATION DROP ROW,
  CONSTRAINT valid_customer EXPECT (customer_id IS NOT NULL) ON VIOLATION FAIL UPDATE
) AS SELECT * FROM STREAM(main.raw.orders);
```

## Quick Reference

| Action | Python | SQL | Invalid rows | Metrics |
|--------|--------|-----|--------------|---------|
| warn (default) | `@dp.expect` | `EXPECT (...)` | Written | Yes |
| drop | `@dp.expect_or_drop` | `... ON VIOLATION DROP ROW` | Dropped | Yes |
| fail | `@dp.expect_or_fail` | `... ON VIOLATION FAIL UPDATE` | Update fails (atomic rollback) | No |

| Rule | Detail |
|------|--------|
| Supported on | Streaming tables, materialized views, temporary views |
| Not supported on | Sinks, `AUTO CDC FROM SNAPSHOT` flows |
| Constraint content | Valid SQL; no Python UDFs, external calls, or subqueries on other tables |
| Names | Unique per dataset; reusable across datasets |
| Grouping | Python only (`expect_all*` take a dict) |
| Failure scope | Triggered: other parallel flows keep running. Continuous: the flow and its dependents stop |

## Layering Convention

A common convention (not a platform rule): warn in bronze, drop in silver, fail on critical gold
invariants. Keep FAIL for conditions that make downstream results wrong.

## Common Mistakes

### Wrong

```python
@dp.table
@dp.expect_or_fail("fk_customer", "customer_id IN (SELECT id FROM customers)")  # subquery
def orders():
    return spark.readStream.table("raw_orders")
```

### Correct

```python
@dp.materialized_view
@dp.expect_or_drop("has_customer", "c_id IS NOT NULL")   # check after a left join
def orders_with_customer():
    o = spark.read.table("raw_orders")
    c = spark.read.table("customers").selectExpr("id AS c_id")
    return o.join(c, o.customer_id == c.c_id, "left")
```

## Related

- [Expectations pattern](../patterns/expectations.md)
- [Advanced expectations](../patterns/expectations-advanced.md)
- [Core concepts](core-concepts.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/serverless
- https://docs.databricks.com/aws/en/ldp/developer/python-ref
