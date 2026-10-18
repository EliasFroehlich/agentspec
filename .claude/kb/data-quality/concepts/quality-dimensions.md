# Quality Dimensions

> **Purpose**: Six dimensions of data quality — completeness, accuracy, consistency, timeliness, uniqueness, validity — with measurement SQL
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Data quality is measured across six dimensions. Each dimension maps to specific SQL checks that can be automated in dbt tests, Great Expectations suites, Soda contracts, Databricks expectations/constraints or ODCS `quality` rules. Measuring all six gives a comprehensive health score for any dataset. The SQL below is Databricks SQL (`COUNT(*) FILTER (WHERE …)` is supported; `EXTRACT(EPOCH …)` is not — use `timestampdiff`). With ANSI mode (default on DBR 17.0+), division by zero raises an error — guard
ratios with `NULLIF(COUNT(*), 0)` or `try_divide` ([ANSI mode](../../spark/concepts/ansi-mode.md)).

## The Concept

```sql
-- Measure all six quality dimensions for an orders table (Databricks SQL)
WITH quality_report AS (
    SELECT
        -- 1. COMPLETENESS: share of non-null values in required fields
        1.0 - COUNT(*) FILTER (WHERE order_id IS NULL) / NULLIF(COUNT(*), 0) AS completeness_order_id,
        1.0 - COUNT(*) FILTER (WHERE amount IS NULL) / NULLIF(COUNT(*), 0)   AS completeness_amount,

        -- 2. UNIQUENESS: distinct share of the primary key
        COUNT(DISTINCT order_id) / NULLIF(COUNT(*), 0)             AS uniqueness_order_id,

        -- 3. VALIDITY: share of values matching business rules
        COUNT(*) FILTER (WHERE amount >= 0) / NULLIF(COUNT(*), 0)  AS validity_amount_positive,
        COUNT(*) FILTER (WHERE status IN ('pending','completed','cancelled'))
            / NULLIF(COUNT(*), 0)                                  AS validity_status_enum,

        -- 4. TIMELINESS: hours since the most recent record
        timestampdiff(SECOND, MAX(updated_at), current_timestamp()) / 3600.0 AS staleness_hours,

        -- 5. CONSISTENCY: cross-table referential integrity
        (SELECT COUNT(*) FROM orders o
         LEFT JOIN customers c ON o.customer_id = c.customer_id
         WHERE c.customer_id IS NULL)                              AS orphan_orders,

        -- 6. ACCURACY: requires ground truth comparison
        COUNT(*)                                                   AS total_rows
    FROM orders
)
SELECT * FROM quality_report;
```

## Quick Reference

| Dimension | Question | SQL Pattern | Threshold |
|-----------|----------|-------------|-----------|
| **Completeness** | Are required fields populated? | `COUNT(*) FILTER (WHERE col IS NULL)` | > 99% |
| **Uniqueness** | Are primary keys unique? | `COUNT(DISTINCT pk) = COUNT(*)` | 100% |
| **Validity** | Do values match business rules? | `CASE WHEN` + regex/enum checks | > 99.5% |
| **Timeliness** | Is data fresh enough? | `MAX(updated_at)` vs SLA | Within SLA |
| **Consistency** | Do related tables agree? | `LEFT JOIN` orphan check | 0 orphans |
| **Accuracy** | Does data match reality? | Spot-check vs source of truth | > 99% |

## Common Mistakes

### Wrong

```sql
-- Only checking completeness — missing 5 other dimensions
SELECT COUNT(*) FILTER (WHERE email IS NULL) AS null_emails FROM customers;
-- This tells you nothing about uniqueness, validity, freshness, etc.
```

### Correct

```sql
-- Check at least completeness + uniqueness + validity together
SELECT
    COUNT(*) FILTER (WHERE email IS NULL) / NULLIF(COUNT(*), 0)        AS null_rate,
    COUNT(DISTINCT email) / NULLIF(COUNT(*), 0)                        AS unique_rate,
    COUNT(*) FILTER (WHERE email LIKE '%@%.%') / NULLIF(COUNT(*), 0)   AS valid_rate
FROM customers;
```

## Related

- [observability](../concepts/observability.md)
- [databricks-data-quality](../concepts/databricks-data-quality.md)
- [dbt-testing](../patterns/dbt-testing.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/functions/count
- https://docs.databricks.com/aws/en/sql/language-manual/functions/extract
- https://docs.databricks.com/aws/en/sql/language-manual/functions/timestampdiff
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://bitol-io.github.io/open-data-contract-standard/latest/data-quality/
