# Deduplication

> **Purpose**: Keep-latest dedup with ROW_NUMBER / QUALIFY, dedup inside MERGE, and cross-source priority — with the dialect differences that break "portable" SQL (`* EXCEPT`, QUALIFY)
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## When to Use

- Raw data contains duplicates from CDC, retries or event replay
- Several sources feed the same entity with overlapping keys
- Late-arriving data creates duplicate rows in append-only tables
- You must keep exactly one (the most recent) version per key

## Implementation

```sql
-- === EXACT DEDUP (portable: every engine with window functions) ===
WITH ranked AS (
    SELECT order_id, customer_id, amount, updated_at, _loaded_at,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY updated_at DESC, _loaded_at DESC      -- deterministic tiebreaker
        ) AS rn
    FROM raw_orders
)
SELECT order_id, customer_id, amount, updated_at, _loaded_at
FROM ranked
WHERE rn = 1;

-- Dropping the helper column without listing columns is NOT standard SQL:
--   Databricks (DBR 11.3+) / Spark 4.x / BigQuery:  SELECT * EXCEPT (rn) FROM ranked WHERE rn = 1
--   Snowflake / DuckDB:                             SELECT * EXCLUDE (rn) FROM ranked WHERE rn = 1
--   PostgreSQL:                                     list the columns

-- === QUALIFY shorthand ===
-- Snowflake, BigQuery, DuckDB, Databricks (DBR 10.4 LTS+), OSS Spark SQL 4.2+
SELECT *
FROM raw_orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC, _loaded_at DESC) = 1;

-- PostgreSQL / DuckDB alternative
SELECT DISTINCT ON (order_id) *
FROM raw_orders
ORDER BY order_id, updated_at DESC, _loaded_at DESC;


-- === DEDUP WITH MERGE (incremental, Delta on Databricks) ===
-- Each target row may match at most one source row, so dedup the source first.
MERGE INTO main.silver.orders AS t
USING (
    SELECT *
    FROM main.bronze.raw_orders
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s.updated_at > t.updated_at THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
-- New source columns: MERGE WITH SCHEMA EVOLUTION INTO ... (Databricks DBR 15.2+), if your
-- schema-change policy allows evolving the target from the source.


-- === CROSS-SOURCE DEDUP (priority, then recency) ===
WITH all_sources AS (
    SELECT email, name, updated_at, 1 AS source_priority FROM crm_customers
    UNION ALL
    SELECT email, name, updated_at, 2 AS source_priority FROM web_customers
    UNION ALL
    SELECT email, name, updated_at, 3 AS source_priority FROM manual_uploads
)
SELECT *
FROM all_sources
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY lower(trim(email))
    ORDER BY source_priority ASC, updated_at DESC
) = 1;
-- Under ANSI (Spark 4 / DBR 17+), UNION branches with different column types are coerced to the
-- least common type (e.g. STRING -> BIGINT) and can fail: cast every branch to the same type.
```

## Configuration

| Strategy | Use when | Notes |
|----------|----------|-------|
| `ROW_NUMBER` + filter | Exact key, keep latest | Portable; one sort per partition key |
| `QUALIFY` | Same, shorter | Not in PostgreSQL or OSS Spark < 4.2 |
| `DISTINCT ON` | Keep first row per key | PostgreSQL and DuckDB |
| `GROUP BY` + `MAX`/`max_by` | Need only a few columns | `max_by(col, updated_at)` in Spark/Databricks/DuckDB avoids a window |
| `MERGE` with deduplicated source | Incremental upsert | Dedup the source first to avoid multiple-match errors |
| PySpark `dropDuplicates(["key"])` | Any row per key is acceptable | Not "latest" — use a window when order matters |

## Example Usage

```sql
-- How many duplicate keys exist?
SELECT order_id, COUNT(*) AS cnt
FROM raw_orders
GROUP BY order_id
HAVING COUNT(*) > 1
ORDER BY cnt DESC
LIMIT 20;
```

## See Also

- [window-functions](../concepts/window-functions.md)
- [gap-and-island](../patterns/gap-and-island.md)
- [Delta MERGE / SCD2 (Spark)](../../spark/patterns/delta-integration.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-star
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-datatype-rules
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://spark.apache.org/docs/latest/sql-pipe-syntax.html
- https://docs.snowflake.com/en/sql-reference/sql/select
- https://duckdb.org/docs/current/sql/query_syntax/select
- https://www.postgresql.org/docs/current/sql-select.html
- https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax
