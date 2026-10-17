# Cross-Dialect Patterns

> **Purpose**: Translate SQL between Snowflake, BigQuery, DuckDB, Spark SQL (OSS 4.x) and Databricks SQL — functions, QUALIFY/LATERAL support, JSON/VARIANT, ASOF, time bucketing, pipe syntax, DuckDB specifics
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Migrating queries between platforms (e.g. Snowflake → Databricks)
- Writing SQL that must run on more than one engine
- Checking whether a construct exists in the target engine **and version** (OSS Spark 4.0 ≠ Databricks DBR 17.3)

## Implementation

```sql
-- === DATE/TIME ===
-- Date truncation
-- Snowflake / DuckDB:     DATE_TRUNC('MONTH', order_date)
-- BigQuery:               DATE_TRUNC(order_date, MONTH)
-- Spark SQL / Databricks: DATE_TRUNC('MONTH', order_date)   -- or TRUNC(order_date, 'MM')

-- Difference in days
-- Snowflake:              DATEDIFF('DAY', start_date, end_date)
-- BigQuery:               DATE_DIFF(end_date, start_date, DAY)
-- DuckDB:                 DATE_DIFF('DAY', start_date, end_date)
-- Spark SQL / Databricks: DATEDIFF(end_date, start_date)            -- two-argument form = days

-- Difference in other units
-- Databricks (DBR 10.4+): timestampdiff(SECOND, start_ts, end_ts)  -- also datediff(unit, start, end)
-- PySpark 4.0+:           F.timestamp_diff("SECOND", "start_ts", "end_ts")
-- Portable Spark:         unix_timestamp(end_ts) - unix_timestamp(start_ts)

-- === STRING AGGREGATION ===
-- Snowflake:              LISTAGG(col, ', ') WITHIN GROUP (ORDER BY col)
-- BigQuery / DuckDB:      STRING_AGG(col, ', ' ORDER BY col)
-- Spark 4.0+ / DBR 17.3:  listagg(col, ', ') WITHIN GROUP (ORDER BY col)   -- alias string_agg
-- Spark 3.x:              array_join(collect_list(col), ', ')               -- order not guaranteed

-- === ARRAYS ===
-- Snowflake:  ARRAY_CONSTRUCT(1, 2, 3)        | SELECT f.VALUE FROM t, LATERAL FLATTEN(arr) f
-- BigQuery:   [1, 2, 3]                       | SELECT v FROM t, UNNEST(arr) AS v
-- DuckDB:     [1, 2, 3]                       | SELECT UNNEST(arr) FROM t
-- Spark SQL:  ARRAY(1, 2, 3)                  | SELECT explode(arr) FROM t

-- === QUALIFY (filter on window functions) ===
-- Supported: Snowflake, BigQuery, DuckDB, Databricks (DBR 10.4 LTS+), OSS Spark SQL 4.2+
-- Not supported: PostgreSQL, OSS Spark 4.0/4.1 (e.g. plain Spark 4.0 outside Databricks)
SELECT * FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;

-- Portable workaround
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) sub WHERE rn = 1;

-- === SEMI-STRUCTURED ===
-- Snowflake:   PARSE_JSON('{"a":1}'):a::INT
-- BigQuery:    JSON_VALUE('{"a":1}', '$.a')
-- DuckDB:      '{"a":1}'::JSON->>'a'
-- Databricks:  '{"a":1}':a::int                   -- `:` works on JSON STRING and VARIANT (DBR 15.4+ for VARIANT)
-- Spark 4.0+:  variant_get(parse_json('{"a":1}'), '$.a', 'int')   -- `:` on VARIANT only from OSS 4.1
-- Legacy:      get_json_object('{"a":1}', '$.a')
```

## Configuration

| Feature | Snowflake | BigQuery | DuckDB | Spark SQL (OSS) | Databricks |
|---------|-----------|----------|--------|-----------------|------------|
| QUALIFY | Yes | Yes | Yes | 4.2+ | DBR 10.4+ |
| LATERAL subquery | Yes | Correlated subquery | Yes | Yes | Yes |
| Recursive CTE | Yes | Yes | Yes | 4.1+ | DBR 17.0+ |
| `UNPIVOT` | Yes | Yes | Yes | 3.4+ | Yes |
| MERGE | Yes | Yes | Yes | Only for row-level-capable tables (Delta, Iceberg); `WITH SCHEMA EVOLUTION` grammar 4.0+ | Delta (`WITH SCHEMA EVOLUTION` DBR 15.2+) |
| Pipe syntax | — | Yes | — | 4.0+ | DBR 16.2+ |
| Regex match | `REGEXP_LIKE` | `REGEXP_CONTAINS` | `REGEXP_MATCHES` | `RLIKE` / `regexp_like` | same as Spark |
| Implicit STRING↔number coercion | check dialect docs | check dialect docs | check dialect docs | ANSI (4.0+): string cast to the numeric type, errors on bad values | same (DBR 17.0+) |

## Pipe Syntax (Spark 4.0+, Databricks DBR 16.2+, BigQuery)

```sql
-- Spark / Databricks
FROM orders
|> WHERE order_date >= DATE '2026-01-01'
|> AGGREGATE SUM(amount) AS revenue GROUP BY region
|> WHERE revenue > 10000              -- replaces HAVING
|> ORDER BY revenue DESC
|> LIMIT 10;

-- BigQuery uses the same operators (|> WHERE, |> AGGREGATE ... GROUP BY, |> ORDER BY, |> LIMIT)
```

## ASOF JOIN (Temporal Lookups)

```sql
-- DuckDB
SELECT o.order_id, o.amount, r.rate
FROM orders o
ASOF JOIN exchange_rates r ON o.currency = r.currency AND o.order_ts >= r.effective_ts;

-- Snowflake (MATCH_CONDITION required; ON optional)
SELECT o.order_id, o.amount, r.rate
FROM orders o
ASOF JOIN exchange_rates r
    MATCH_CONDITION (o.order_ts >= r.effective_ts)
    ON o.currency = r.currency;

-- Spark / Databricks / BigQuery (no ASOF JOIN): join on key + time bound, keep the latest
SELECT * FROM (
    SELECT o.order_id, o.amount, r.rate,
           ROW_NUMBER() OVER (PARTITION BY o.order_id ORDER BY r.effective_ts DESC) AS rn
    FROM orders o
    JOIN exchange_rates r ON r.currency = o.currency AND r.effective_ts <= o.order_ts
) x WHERE rn = 1;
```

## Time Bucketing

```sql
-- DuckDB
SELECT time_bucket(INTERVAL '15 minutes', reading_ts) AS bucket, AVG(value) FROM sensor_readings GROUP BY bucket;
-- Snowflake
SELECT TIME_SLICE(reading_ts, 15, 'MINUTE') AS bucket, AVG(value) FROM sensor_readings GROUP BY bucket;
-- BigQuery
SELECT TIMESTAMP_BUCKET(reading_ts, INTERVAL 15 MINUTE) AS bucket, AVG(value) FROM sensor_readings GROUP BY bucket;
-- Spark SQL / Databricks (all versions): window() returns a struct with start/end
SELECT window.start AS bucket, AVG(value)
FROM sensor_readings GROUP BY window(reading_ts, '15 minutes');
-- OSS Spark 4.2+ also adds time_bucket (not on DBR 17.3 LTS / 18 LTS)
```

## Example Usage

```sql
-- DuckDB power features
SELECT [x * 2 FOR x IN [1, 2, 3, 4] IF x > 1];                -- list comprehension: [4, 6, 8]
SELECT * FROM 's3://bucket/data/*.parquet' WHERE year = 2026;  -- query files directly
SELECT COLUMNS('revenue_.*') FROM quarterly_report;            -- dynamic column selection
SELECT * EXCLUDE (internal_id) FROM customers;

-- Lambdas: DuckDB 1.3 deprecated the single arrow; use the Python-style form
SELECT list_transform([1, 2, 3], lambda x: x + 1);             -- [2, 3, 4]
-- Old: list_transform([1, 2, 3], x -> x + 1)  -- disabled by default in DuckDB 2.0, removed in 2.1
```

## See Also

- [pivot-unpivot](../patterns/pivot-unpivot.md)
- [set-operations](../concepts/set-operations.md)
- [ANSI mode (Spark)](../../spark/concepts/ansi-mode.md)

## Sources

- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.functions.listagg.html
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://spark.apache.org/docs/latest/sql-pipe-syntax.html
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-lateral-subquery.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-select-qualify
- https://docs.databricks.com/aws/en/sql/language-manual/functions/colonsign
- https://docs.databricks.com/aws/en/sql/language-manual/functions/timestampdiff
- https://docs.databricks.com/aws/en/sql/language-manual/functions/datediff3
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-datatype-rules
- https://docs.snowflake.com/en/sql-reference/constructs/asof-join
- https://docs.snowflake.com/en/sql-reference/functions/time_slice
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/pipe-syntax
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/time-series-functions
- https://duckdb.org/docs/lts/sql/functions/lambda
- https://duckdb.org/docs/current/sql/expressions/star
- https://spark.apache.org/docs/4.0.0/api/sql/index.html
