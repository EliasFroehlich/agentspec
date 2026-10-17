# SQL Patterns Quick Reference

> **Purpose**: Fast cross-dialect lookup — Snowflake, BigQuery, DuckDB, PostgreSQL, Spark SQL (OSS) and Databricks SQL, with minimum versions
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Window / Filtering Features by Dialect

| Feature | Snowflake | BigQuery | DuckDB | Postgres | Spark SQL (OSS) | Databricks |
|---------|-----------|----------|--------|----------|-----------------|------------|
| `ROW_NUMBER`, `NTILE`, `FIRST_VALUE`, `NTH_VALUE` | Yes | Yes | Yes | Yes | Yes | Yes |
| `QUALIFY` | Yes | Yes | Yes | No (subquery) | **4.2+** | DBR 10.4 LTS+ |
| `DISTINCT ON` | use `QUALIFY` | use `QUALIFY` | Yes | Yes | use `ROW_NUMBER` | use `QUALIFY` |
| `SELECT * EXCEPT/EXCLUDE (col)` | `EXCLUDE` | `EXCEPT` | `EXCLUDE` | No | `* EXCEPT` (4.x) | `* EXCEPT` (DBR 11.3+) |
| Pipe syntax (`\|>`) | — | Yes | — | — | 4.0+ | DBR 16.2+ |

## CTE Support Matrix

| Feature | Snowflake | BigQuery | DuckDB | Postgres | Spark SQL (OSS) | Databricks |
|---------|-----------|----------|--------|----------|-----------------|------------|
| Standard CTE | Yes | Yes | Yes | Yes | Yes | Yes |
| `WITH RECURSIVE` | Yes | Yes | Yes | Yes | **4.1+** | DBR 17.0+ (`MAX RECURSION LEVEL`, default 100; 1M-row cap; `LIMIT ALL` 17.2+) |
| Materialization hint | No | No | No | `MATERIALIZED` / `NOT MATERIALIZED` | No | No |

## Joins and Set Operations

| Feature | Snowflake | BigQuery | DuckDB | Postgres | Spark SQL / Databricks |
|---------|-----------|----------|--------|----------|------------------------|
| `LATERAL` subquery | Yes | Correlated subquery / `UNNEST` | Yes | Yes | Yes (`FROM t1, LATERAL (...)`); PySpark `lateralJoin` 4.0+ |
| `ASOF JOIN` | Yes (`MATCH_CONDITION`) | No | Yes | No | No — window workaround |
| `EXCEPT` | Standard | `EXCEPT DISTINCT` | Standard | Standard | Standard |
| `UNPIVOT` clause | Yes | Yes | `UNPIVOT … ON` | No clause (`UNNEST`) | Spark 3.4+ |

## Functions by Dialect

| Concept | Snowflake | BigQuery | DuckDB | Spark SQL / Databricks |
|---------|-----------|----------|--------|------------------------|
| Date trunc | `DATE_TRUNC('month', d)` | `DATE_TRUNC(d, MONTH)` | `DATE_TRUNC('month', d)` | `DATE_TRUNC('MONTH', d)` |
| Seconds between | `DATEDIFF('second', a, b)` | `TIMESTAMP_DIFF(b, a, SECOND)` | `DATE_DIFF('second', a, b)` | Databricks: `timestampdiff(SECOND, a, b)` (DBR 10.4+); portable: `unix_timestamp(b) - unix_timestamp(a)` |
| String aggregation | `LISTAGG(c, ', ')` | `STRING_AGG(c, ', ')` | `STRING_AGG(c, ', ')` | `listagg(c, ', ')` / `string_agg` (Spark 4.0+) |
| JSON field | `v:a::INT` | `JSON_VALUE(j, '$.a')` | `j->>'a'` | Databricks `j:a::int` (STRING JSON or VARIANT); `variant_get(v, '$.a', 'int')` |
| Time bucket | `TIME_SLICE(ts, 15, 'MINUTE')` | `TIMESTAMP_BUCKET(ts, INTERVAL 15 MINUTE)` | `time_bucket(INTERVAL '15 min', ts)` | `window(ts, '15 minutes')`; `time_bucket` only in Spark 4.2+ |
| Case-insensitive compare | — | — | — | Collations (Spark 4.0+, Databricks DBR 16.1+): `c COLLATE UTF8_LCASE` |
| Date spine | `GENERATOR(ROWCOUNT => n)` | `GENERATE_DATE_ARRAY` | `generate_series` | `explode(sequence(d1, d2, INTERVAL 1 DAY))` |

## ANSI Behaviour (Spark 4.0+ / DBR 17.0+)

| Expression | Result | Safe form |
|------------|--------|-----------|
| `CAST('a' AS INT)` | error | `try_cast('a' AS INT)` |
| `x / 0` | error | `try_divide(x, 0)` |
| `coalesce(5, 'abc')`, `UNION` of BIGINT and STRING | STRING cast to BIGINT → error on bad values | Cast every branch explicitly |

## DuckDB Version Notes

| Version | Note |
|---------|------|
| 1.3.0 | Single-arrow lambda `x -> x + 1` deprecated in favour of `lambda x: x + 1`; Linux binaries need glibc 2.28+ |
| 1.4.x (LTS 1.4.5) | Both lambda syntaxes accepted |
| 1.5.x (stable 1.5.6) | Last release accepting single-arrow lambdas without opting in |
| 2.0.0 (scheduled 2026-10-21) | Single-arrow syntax disabled by default; removed in 2.1 |

## Anti-Pattern Checklist

| Anti-Pattern | Fix |
|-------------|-----|
| `SELECT *` in production | Explicit column list |
| `DISTINCT` masking duplication | Fix the upstream join fan-out |
| Correlated subquery per row | Rewrite as JOIN or window |
| Implicit type coercion (joins, UNION, `coalesce`) | Explicit `CAST` / `try_cast` |
| `ORDER BY` without `LIMIT` in subqueries | Order only the final result |
| `NOT IN (subquery with NULLs)` | `NOT EXISTS` |
| `ROW_NUMBER()` without deterministic `ORDER BY` | Add a tiebreaker column |

## Sources

- https://spark.apache.org/releases/spark-release-4-2-0.html · https://spark.apache.org/releases/spark-release-4.1.0.html · https://spark.apache.org/releases/spark-release-3-4-0.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify · https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-cte
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-star · https://docs.databricks.com/aws/en/sql/language-manual/functions/timestampdiff
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-collation · https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-datatype-rules
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-lateral-subquery.html · https://spark.apache.org/docs/latest/sql-pipe-syntax.html
- https://docs.snowflake.com/en/sql-reference/constructs/asof-join · https://docs.snowflake.com/en/sql-reference/functions/time_slice · https://docs.snowflake.com/en/sql-reference/sql/select
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/pipe-syntax · https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/time-series-functions
- https://duckdb.org/docs/lts/sql/functions/lambda · https://duckdb.org/2025/05/21/announcing-duckdb-130 · https://duckdb.org/release_calendar
- https://www.postgresql.org/docs/current/queries-with.html · https://www.postgresql.org/docs/current/sql-select.html
