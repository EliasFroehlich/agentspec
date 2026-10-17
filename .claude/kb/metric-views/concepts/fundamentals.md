# Metric View Fundamentals

> **Purpose**: What a Unity Catalog metric view is and why it exists — the core mental model before writing YAML
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

A metric view is the core implementation of **Unity Catalog semantics**: a YAML definition registered in Unity Catalog (`CREATE VIEW ... WITH METRICS LANGUAGE YAML`) that separates **fields** (also called dimensions: scalar attributes to group and filter by) from **measures** (aggregate expressions with no fixed grain). A metric is defined once and the engine computes it correctly for whatever fields the caller groups by — the opposite of a standard SQL view, which locks its `GROUP BY` shape at creation time.

## The Concept

```sql
CREATE OR REPLACE VIEW main.sales.orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
version: 1.1                          # required: YAML spec version (1.1 or 0.1)
source: main.sales.orders             # table, view, metric view, or SQL query
fields:                               # preferred keyword; `dimensions:` is an accepted synonym
  - name: Order Month
    expr: DATE_TRUNC('MONTH', order_date)   # scalar expression, no aggregates
measures:
  - name: Total Revenue
    expr: SUM(total_price)                  # aggregate, independent of any GROUP BY
  - name: Unique Customers
    expr: COUNT(DISTINCT customer_id)
  - name: Revenue per Customer
    expr: MEASURE(`Total Revenue`) / MEASURE(`Unique Customers`)   # composed from other measures
$$
```

```sql
-- Queried like a view, but every measure is wrapped in MEASURE()
SELECT `Order Month`, MEASURE(`Total Revenue`) AS revenue
FROM main.sales.orders_metrics
GROUP BY ALL
ORDER BY ALL
```

## Quick Reference

| Aspect | Standard View | Metric View |
|--------|---------------|-------------|
| Aggregation grain | Locked at `CREATE VIEW` time | Chosen per query via `GROUP BY` |
| Ratio re-aggregation | Unsafe — averaging an average | Safe — engine recomputes from the aggregates |
| Star/snowflake joins | Hand-written SQL joins | Declarative `joins:` block in YAML |
| Query syntax | Plain `SELECT column` | Fields plain, measures via `MEASURE(name)` (`agg(name)` on DBR 18.1+) |
| Reuse of logic | Copy the SQL | Composed measures (`MEASURE(a) / MEASURE(b)`), metric view as `source` of another |
| Business context for AI/BI and Genie Agents | Column comments only | `comment`, `display_name`, `format`, `synonyms` per field/measure (spec 1.1) |

Field types: a field is either a categorical column (region, status) or an unaggregated numeric column (price, quantity) that callers can aggregate at query time. String-like fields are always `STRING`, even when the source column is `CHAR`/`VARCHAR`, so `CHAR(n)` space padding is lost and equality comparisons can differ from the source table.

## Common Mistakes

### Wrong

```sql
-- Standard view exposing a pre-divided ratio column
CREATE VIEW dept_summary AS
SELECT dept, SUM(revenue) revenue, SUM(revenue)/SUM(cost) AS margin
FROM transactions GROUP BY dept;
-- Rolling this up to company level: AVG(margin) is WRONG (averages an average)
```

### Correct

```yaml
# Metric view: margin is a composed measure, correct at any grain
measures:
  - name: Revenue
    expr: SUM(revenue)
  - name: Cost
    expr: SUM(cost)
  - name: Margin
    expr: MEASURE(Revenue) / NULLIF(MEASURE(Cost), 0)
# SELECT MEASURE(Margin) FROM view GROUP BY ALL  -- correct at dept, region, or company grain
```

## Related

- [Requirements & Versions](requirements-versions.md)
- [Limitations](limitations.md)
- [Create and Query Pattern](../patterns/create-and-query.md)
- [Ratio and Filtered Measures](../patterns/ratio-and-filtered-measures.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/basic-modeling
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/advanced-techniques (composability)
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure
