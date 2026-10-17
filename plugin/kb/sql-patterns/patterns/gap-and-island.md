# Gap and Island

> **Purpose**: Date-gap detection with a date spine, consecutive-run (island) detection, and sessionization — with dialect-labelled variants, including Spark SQL / Databricks
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Detecting missing days in daily data (SLA / completeness monitoring)
- Grouping events into sessions separated by inactivity gaps
- Finding streaks and contiguous ranges (consecutive active days, uninterrupted subscriptions)

## Implementation

```sql
-- === DATE SPINE + GAP DETECTION ===
-- Snowflake
WITH date_spine AS (
    SELECT DATEADD(DAY, seq, '2026-01-01'::DATE) AS expected_date
    FROM (SELECT ROW_NUMBER() OVER (ORDER BY 1) - 1 AS seq
          FROM TABLE(GENERATOR(ROWCOUNT => 365)))
)
SELECT d.expected_date AS missing_date
FROM date_spine d
LEFT JOIN (SELECT DISTINCT order_date FROM orders) a ON d.expected_date = a.order_date
WHERE a.order_date IS NULL
ORDER BY missing_date;

-- Spark SQL / Databricks
WITH date_spine AS (
    SELECT explode(sequence(DATE'2026-01-01', DATE'2026-12-31', INTERVAL 1 DAY)) AS expected_date
)
SELECT d.expected_date AS missing_date
FROM date_spine d
LEFT ANTI JOIN orders a ON d.expected_date = a.order_date     -- Spark: LEFT ANTI JOIN keeps unmatched
ORDER BY missing_date;


-- === ISLAND DETECTION (consecutive days per customer) ===
-- Standard SQL idea: date minus row number is constant within a run
-- Postgres / DuckDB style (adapt the date arithmetic per dialect)
WITH numbered AS (
    SELECT customer_id, order_date,
           order_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)
                        * INTERVAL '1 DAY' AS grp
    FROM (SELECT DISTINCT customer_id, order_date FROM orders) d
)
SELECT customer_id, MIN(order_date) AS island_start, MAX(order_date) AS island_end,
       COUNT(*) AS consecutive_days
FROM numbered
GROUP BY customer_id, grp
ORDER BY customer_id, island_start;

-- Spark SQL / Databricks: date_sub(date, int) instead of date - interval * n
--   date_sub(order_date, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)) AS grp


-- === SESSIONIZATION (30-minute inactivity gap) ===
-- Snowflake / DuckDB: DATEDIFF('SECOND', prev_ts, ts)
-- Databricks (DBR 10.4+): timestampdiff(SECOND, prev_ts, ts)
-- Portable Spark: unix_timestamp(ts) - unix_timestamp(prev_ts)   <- used below (Spark SQL / Databricks)
WITH with_gap AS (
    SELECT *,
        unix_timestamp(event_ts)
          - unix_timestamp(LAG(event_ts) OVER (PARTITION BY user_id ORDER BY event_ts)) AS gap_seconds
    FROM clickstream
),
with_boundary AS (
    SELECT *, CASE WHEN gap_seconds > 1800 OR gap_seconds IS NULL THEN 1 ELSE 0 END AS new_session
    FROM with_gap
)
SELECT *,
    SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_ts
                           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS session_id
FROM with_boundary;
```

## Configuration

| Pattern | Key technique | Dialect notes |
|---------|---------------|---------------|
| Date spine | Generate expected dates, anti-join actuals | `generate_series` (Postgres/DuckDB), `GENERATOR` (Snowflake), `UNNEST(GENERATE_DATE_ARRAY(...))` (BigQuery), `explode(sequence(...))` (Spark/Databricks) |
| Islands | `date - ROW_NUMBER()` is constant within a run | Spark/Databricks: `date_sub(date, rn)` |
| Sessions | `LAG` gap + running `SUM` of boundaries | Seconds between timestamps differ per dialect (see above) |
| Streaming sessions | — | Spark Structured Streaming: `session_window(ts, '30 minutes')` |

## Example Usage

```sql
-- Spark SQL / Databricks: quick completeness check for the current month
SELECT COUNT(DISTINCT order_date)                                   AS actual_days,
       datediff(current_date(), trunc(current_date(), 'MM')) + 1    AS expected_days
FROM orders
WHERE order_date >= trunc(current_date(), 'MM');
```

## See Also

- [window-functions](../concepts/window-functions.md)
- [deduplication](../patterns/deduplication.md)
- [Spark window functions](../../spark/patterns/window-functions.md)

## Sources

- https://spark.apache.org/docs/4.0.0/api/sql/index.html
- https://docs.databricks.com/aws/en/sql/language-manual/functions/timestampdiff
- https://docs.databricks.com/aws/en/sql/language-manual/functions/datediff3
- https://spark.apache.org/docs/latest/sql-ref-syntax-qry-select-window.html
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/functions.html
