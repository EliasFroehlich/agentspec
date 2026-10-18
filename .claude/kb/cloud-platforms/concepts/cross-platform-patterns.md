# Cross-Platform SQL Patterns

> **Purpose**: SQL dialect differences across Snowflake, Databricks, and BigQuery for migration and multi-cloud
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Snowflake, Databricks (Spark SQL), and BigQuery each implement ANSI SQL with vendor-specific extensions. Understanding dialect differences is critical for multi-cloud pipelines, platform migrations, and writing portable SQL. Key divergence areas include date functions, semi-structured data, MERGE syntax, array handling, and type systems.

## The Concept

### DATE_TRUNC Syntax

```sql
-- Snowflake: DATE_TRUNC(part, expr)
SELECT DATE_TRUNC('month', order_date) AS month_start FROM orders;

-- Databricks (Spark SQL): DATE_TRUNC(part, expr) -- same as Snowflake
SELECT DATE_TRUNC('month', order_date) AS month_start FROM orders;

-- BigQuery: DATE_TRUNC(expr, part) -- reversed argument order
SELECT DATE_TRUNC(order_date, MONTH) AS month_start FROM orders;
```

### Semi-Structured Data (JSON)

```sql
-- Snowflake: VARIANT type with colon notation
SELECT
  raw_data:customer.name::STRING AS customer_name,
  raw_data:items[0].price::FLOAT AS first_item_price
FROM events;

-- Databricks: colon path works on JSON STRING and on VARIANT (VARIANT: DBR 15.4 LTS+)
SELECT
  raw_data:customer.name::STRING AS customer_name,
  raw_data:items[0].price::DOUBLE AS first_item_price
FROM events;

-- BigQuery: JSON type with JSON_VALUE / JSON_QUERY
SELECT
  JSON_VALUE(raw_data, '$.customer.name') AS customer_name,
  CAST(JSON_VALUE(raw_data, '$.items[0].price') AS FLOAT64) AS first_item_price
FROM events;
```

### ARRAY and STRUCT Handling

```sql
-- Snowflake: ARRAY_CONSTRUCT + FLATTEN
SELECT f.value::STRING AS tag
FROM products, LATERAL FLATTEN(input => tags) f;

-- Databricks: ARRAY() + EXPLODE
SELECT EXPLODE(tags) AS tag FROM products;

-- BigQuery: ARRAY + UNNEST
SELECT tag FROM products, UNNEST(tags) AS tag;
```

### MERGE (Upsert) Syntax

```sql
-- Snowflake
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.value = s.value, t.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT (id, value, updated_at) VALUES (s.id, s.value, CURRENT_TIMESTAMP());

-- Databricks (Delta Lake)
MERGE INTO target t USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- BigQuery: no UPDATE wildcard; ROW exists only in the INSERT branch
MERGE INTO `project.dataset.target` t USING `project.dataset.source` s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET value = s.value, updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT ROW;
```

### QUALIFY (filter on window functions)

```sql
-- Same syntax on all three: Snowflake, Databricks (DBR 10.4 LTS+ / Databricks SQL), BigQuery
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_ts DESC) = 1;
-- Apache Spark OSS has QUALIFY only from 4.2.0 (SPARK-31561).
```

## Quick Reference

| Feature | Snowflake | Databricks | BigQuery |
|---------|-----------|------------|----------|
| Date truncate | `DATE_TRUNC('month', d)` | `DATE_TRUNC('month', d)` | `DATE_TRUNC(d, MONTH)` |
| JSON access | `col:path::TYPE` | `col:path::TYPE` | `JSON_VALUE(col, '$.path')` |
| Flatten array | `LATERAL FLATTEN()` | `EXPLODE()` | `UNNEST()` |
| Create array | `ARRAY_CONSTRUCT()` | `ARRAY()` | `[ ]` literal |
| Regex extract | `REGEXP_SUBSTR()` | `REGEXP_EXTRACT()` | `REGEXP_EXTRACT()` |
| String agg | `LISTAGG()` | `LISTAGG()` / `STRING_AGG()` (DBR 16.4+); `concat_ws(',', collect_list(x))` before | `STRING_AGG()` |
| Upsert wildcard | Not supported | `UPDATE SET *` / `INSERT *` | Not supported (`INSERT ROW` only) |
| QUALIFY | Supported | DBR 10.4 LTS+ | Supported |
| Temp table | `CREATE TEMP TABLE` | `CREATE TEMP VIEW` | `CREATE TEMP TABLE` |
| Type: string | `VARCHAR` / `STRING` | `STRING` | `STRING` |
| Type: integer | `NUMBER(38,0)` | `BIGINT` / `INT` | `INT64` |
| Type: float | `FLOAT` / `NUMBER(x,y)` | `DOUBLE` | `FLOAT64` |

## Common Mistakes

### Wrong
```sql
-- Assuming DATE_TRUNC argument order is the same everywhere
-- This fails in BigQuery:
SELECT DATE_TRUNC('month', order_date) FROM orders;
```

### Correct
```sql
-- BigQuery requires expression first, then part as keyword
SELECT DATE_TRUNC(order_date, MONTH) FROM orders;
```

### Migration Considerations

- **Snowflake to Databricks:** VARIANT maps to Databricks VARIANT (DBR 15.4 LTS+); FLATTEN to EXPLODE / `LATERAL VIEW`; QUALIFY ports unchanged
- **Snowflake to BigQuery:** Reverse DATE_TRUNC args; FLATTEN to UNNEST; VARIANT to JSON type
- **BigQuery to Snowflake:** UNNEST to FLATTEN; STRUCT to OBJECT; backtick paths to dot notation
- **QUALIFY clause:** Supported in Snowflake, Databricks (DBR 10.4 LTS+) and BigQuery; only open-source Spark before 4.2 lacks it

## Related

- [Snowflake Cortex](snowflake-cortex.md) -- Snowflake-specific AI and platform features
- [Databricks Lakeflow](databricks-lakeflow.md) -- Databricks-specific platform features
- [BigQuery AI](bigquery-ai.md) -- BigQuery-specific ML and AI capabilities
- [SQL Patterns KB](../../sql-patterns/index.md) -- General cross-dialect SQL patterns

## Sources

- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/dml-syntax
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/sql/language-manual/functions/listagg
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://docs.snowflake.com/en/sql-reference/constructs/qualify
