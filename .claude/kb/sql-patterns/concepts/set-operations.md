# Set Operations

> **Purpose**: UNION/INTERSECT/EXCEPT, LATERAL subqueries, UNNEST/FLATTEN/explode and ASOF JOIN across dialects — plus ANSI type coercion of set operations in Spark 4 / Databricks
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Set operations combine result sets vertically (UNION, INTERSECT, EXCEPT); LATERAL subqueries and
UNNEST/FLATTEN/explode expand per-row arrays or correlated lookups. Column types of each branch are
resolved to a **least common type** — under ANSI mode (Spark 4.0+, Databricks DBR 17.0+) a STRING
branch combined with a numeric branch is cast to the numeric type, so one bad value fails the query.

## The Concept

```sql
-- UNION ALL keeps duplicates (no sort/dedup); UNION removes them
SELECT order_id, amount FROM online_orders
UNION ALL
SELECT order_id, amount FROM store_orders;

SELECT customer_id FROM web_customers INTERSECT SELECT customer_id FROM mobile_customers;
SELECT customer_id FROM all_customers EXCEPT SELECT customer_id FROM churned_customers;
-- BigQuery spells these INTERSECT DISTINCT / EXCEPT DISTINCT

-- LATERAL: a subquery that can reference columns of preceding FROM items
-- Postgres / DuckDB / Snowflake
SELECT c.customer_id, recent.order_id, recent.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, amount FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY order_date DESC LIMIT 3
) recent;

-- Spark SQL / Databricks: LATERAL subquery in the FROM list
SELECT * FROM t1,
  LATERAL (SELECT * FROM t2 WHERE t1.c1 = t2.c1);
-- For top-N per key in Spark, prefer ROW_NUMBER() + QUALIFY (Databricks) / filter (OSS)
```

## Quick Reference

| Operation | Snowflake | BigQuery | DuckDB | Spark SQL / Databricks |
|-----------|-----------|----------|--------|------------------------|
| `UNION ALL` | Standard | Standard | Standard | Standard (`unionByName` in DataFrames) |
| `INTERSECT` | Standard | `INTERSECT DISTINCT` | Standard | Standard |
| `EXCEPT` | Standard | `EXCEPT DISTINCT` | Standard | Standard |
| `LATERAL` | `LATERAL` | Correlated subquery / `UNNEST` | `LATERAL` | `LATERAL` subquery; PySpark `DataFrame.lateralJoin` (4.0+) |
| Array unnest | `LATERAL FLATTEN(col)` | `UNNEST(col)` | `UNNEST(col)` | `explode(col)` / `LATERAL VIEW explode` / `inline` |
| Struct access | `col:key` (VARIANT) | `col.key` | `col.key` | `col.key`; Databricks JSON/VARIANT `col:key` |

## Common Mistakes

### Wrong

```sql
-- UNION when duplicates are impossible or acceptable: needless sort + dedup
SELECT * FROM table_a
UNION
SELECT * FROM table_b;

-- Spark 4 / DBR 17+ (ANSI): branch types differ — id is BIGINT in a, STRING in b
SELECT id FROM a
UNION ALL
SELECT id FROM b;          -- least common type BIGINT: 'A-17' raises CAST_INVALID_INPUT
```

### Correct

```sql
SELECT * FROM table_a
UNION ALL
SELECT * FROM table_b;

SELECT CAST(id AS STRING) AS id FROM a
UNION ALL
SELECT id FROM b;          -- conform each branch to the type you intend
```

## ASOF JOIN (Temporal Lookup)

```sql
-- DuckDB: most recent exchange rate at or before each order
SELECT o.order_id, o.amount, r.rate
FROM orders o
ASOF JOIN exchange_rates r
    ON o.currency = r.currency
   AND o.order_ts >= r.effective_ts;

-- Snowflake: MATCH_CONDITION is required, ON is optional
SELECT o.order_id, o.amount, r.rate
FROM orders o
ASOF JOIN exchange_rates r
    MATCH_CONDITION (o.order_ts >= r.effective_ts)
    ON o.currency = r.currency;
```

| Dialect | ASOF JOIN | Alternative |
|---------|-----------|-------------|
| DuckDB | Native | — |
| Snowflake | Native (`MATCH_CONDITION`) | — |
| BigQuery | No | Correlated subquery or window function |
| Spark SQL / Databricks | No | Window function with `LAST_VALUE` (ignoring nulls) over the time order, or range join + `ROW_NUMBER` |

## Related

- [cte-patterns](../concepts/cte-patterns.md)
- [cross-dialect](../patterns/cross-dialect.md)
- [ANSI mode (Spark)](../../spark/concepts/ansi-mode.md)

## Sources

- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-lateral-subquery.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-datatype-rules
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
- https://docs.snowflake.com/en/sql-reference/constructs/asof-join
- https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax
