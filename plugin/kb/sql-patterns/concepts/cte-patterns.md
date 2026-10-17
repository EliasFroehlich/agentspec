# CTE Patterns

> **Purpose**: Chained and recursive CTEs, how each engine treats a CTE referenced more than once, and what to do instead when it gets recomputed
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Common Table Expressions (CTEs) name the steps of a query. Chained CTEs replace nested subqueries;
recursive CTEs walk hierarchies and graphs. Whether a CTE that is referenced twice is computed once
depends on the engine: PostgreSQL materializes it by default and lets you override that, while
Spark/Databricks have no materialization hint — check the plan.

## The Concept

```sql
-- Chained CTEs: each step builds on the previous
WITH raw_orders AS (
    SELECT order_id, customer_id, amount, order_date
    FROM orders
    WHERE order_date >= DATE '2026-01-01'
),
enriched AS (
    SELECT o.*, c.customer_name, c.segment
    FROM raw_orders o
    JOIN customers c ON o.customer_id = c.customer_id
)
SELECT segment, COUNT(*) AS order_count, SUM(amount) AS total_revenue
FROM enriched
GROUP BY segment
ORDER BY total_revenue DESC;

-- Recursive CTE: org hierarchy (Postgres, DuckDB, Snowflake, BigQuery;
-- Databricks DBR 17.0+; OSS Spark 4.1+ — not OSS Spark 4.0)
WITH RECURSIVE org_tree AS (
    SELECT employee_id, name, manager_id, 1 AS depth
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.name, e.manager_id, t.depth + 1
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.employee_id
    WHERE t.depth < 10                      -- explicit safety limit
)
SELECT * FROM org_tree ORDER BY depth, name;
```

```sql
-- Databricks: recursion limit is part of the CTE definition (default 100 levels)
WITH RECURSIVE org_tree MAX RECURSION LEVEL 50 AS (
    SELECT employee_id, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, t.depth + 1
    FROM employees e JOIN org_tree t ON e.manager_id = t.employee_id
)
SELECT * FROM org_tree;
-- Result capped at 1,000,000 rows unless the driving SELECT has a LIMIT; DBR 17.2+: LIMIT ALL lifts it.
-- Not supported inside UPDATE / DELETE / MERGE.
```

## Quick Reference

| Engine | CTE referenced twice | Recursive | Notes |
|--------|----------------------|-----------|-------|
| PostgreSQL | Materialized by default (inlined only if referenced once and side-effect free) | Yes | Override with `AS MATERIALIZED` / `AS NOT MATERIALIZED` |
| Snowflake | Optimizer decides | Yes | No materialization hint |
| BigQuery | Optimizer decides | Yes (`WITH RECURSIVE`) | |
| DuckDB | Optimizer decides | Yes | |
| Spark SQL / Databricks | No hint — verify in the plan whether the CTE's scans/aggregations appear twice | OSS 4.1+; Databricks DBR 17.0+ | Single-pass rewrite or persist the intermediate |

## Common Mistakes

### Wrong

```sql
-- Expensive CTE consumed by two branches: may be computed twice (Spark/Databricks)
WITH expensive AS (
    SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id
)
SELECT customer_id, total, 'big' AS tier FROM expensive WHERE total > 1000
UNION ALL
SELECT customer_id, total, 'small' AS tier FROM expensive WHERE total <= 1000;
```

### Correct

```sql
-- Any engine: single pass, no second consumer
SELECT customer_id,
       SUM(amount) AS total,
       CASE WHEN SUM(amount) > 1000 THEN 'big' ELSE 'small' END AS tier
FROM orders
GROUP BY customer_id;

-- PostgreSQL: make the intent explicit
WITH expensive AS MATERIALIZED (
    SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id
)
SELECT * FROM expensive WHERE total > 1000;

-- Spark/Databricks when two consumers are unavoidable: write the intermediate to a table
-- (works everywhere), or CACHE TABLE on classic compute (cache APIs are not supported on serverless).
CREATE OR REPLACE TABLE main.tmp.customer_totals AS
SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id;
```

## Related

- [window-functions](../concepts/window-functions.md)
- [cross-dialect](../patterns/cross-dialect.md)
- [spark performance-tuning](../../spark/patterns/performance-tuning.md)

## Sources

- https://www.postgresql.org/docs/current/queries-with.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-cte
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-cte.html
- https://docs.databricks.com/aws/en/compute/serverless/limitations
