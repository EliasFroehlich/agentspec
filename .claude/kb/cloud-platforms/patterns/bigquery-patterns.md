# BigQuery Implementation Patterns

> **Purpose**: Production patterns for idempotent scheduled loads, BQML forecasting, remote functions on Cloud Run, Gemini functions in SQL, and Iceberg managed tables
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Recurring batch transforms that must be safe to re-run (scheduled queries, Dataform, orchestrators)
- LLM enrichment of rows (classification, extraction, embeddings) without leaving SQL
- Calling external code (geocoding, decryption, custom APIs) from SQL via remote functions
- Open-format tables in your own bucket that other engines read through the Iceberg REST catalog

## Implementation

### Idempotent partition reload (scheduled query)

```sql
-- @run_date is supplied by the scheduled query runtime
DECLARE run_date DATE DEFAULT @run_date;

BEGIN TRANSACTION;
DELETE FROM analytics.daily_revenue WHERE revenue_date = run_date;
INSERT INTO analytics.daily_revenue
SELECT DATE(order_ts) AS revenue_date, region,
       COUNT(DISTINCT order_id) AS orders,
       SUM(order_total) AS gross_revenue
FROM raw.orders
WHERE DATE(order_ts) = run_date          -- prunes the date-partitioned source
GROUP BY 1, 2;
COMMIT TRANSACTION;
```

### BQML forecasting (ARIMA_PLUS)

```sql
CREATE OR REPLACE MODEL analytics.demand_forecast
OPTIONS (
  model_type = 'ARIMA_PLUS',
  time_series_timestamp_col = 'order_date',
  time_series_data_col = 'daily_orders',
  time_series_id_col = 'product_category',
  auto_arima = TRUE,
  holiday_region = 'US'
) AS
SELECT DATE(order_ts) AS order_date, product_category, COUNT(*) AS daily_orders
FROM raw.orders
GROUP BY 1, 2;

SELECT * FROM ML.EVALUATE(MODEL analytics.demand_forecast);
SELECT * FROM ML.FORECAST(MODEL analytics.demand_forecast,
                          STRUCT(30 AS horizon, 0.95 AS confidence_level));
SELECT * FROM ML.EXPLAIN_FORECAST(MODEL analytics.demand_forecast, STRUCT(30 AS horizon));
```

### Remote function on Cloud Run functions

```sql
-- 1. Cloud resource connection (no CREATE OR REPLACE; options are connection_type/friendly_name/description)
CREATE CONNECTION IF NOT EXISTS `us.cloud_run_conn`
OPTIONS (connection_type = "CLOUD_RESOURCE", friendly_name = "Remote functions");
-- Grant the connection's service account the Cloud Run Invoker role on the service.

-- 2. Remote function; endpoint is the Cloud Run URL (a route path is allowed)
CREATE OR REPLACE FUNCTION analytics.geocode(address STRING)
RETURNS STRING
REMOTE WITH CONNECTION `us.cloud_run_conn`
OPTIONS (
  endpoint = 'https://geocode-<project-hash>-uc.a.run.app/geocode',
  max_batching_rows = 100
);

SELECT store_id, analytics.geocode(address) AS geo_json FROM raw.stores;
```

### Gemini in SQL: scalar functions (no remote model)

```sql
-- Structured extraction; your output_schema fields replace `result`
SELECT
  ticket_id,
  AI.GENERATE(
    CONCAT('Analyze this support ticket: ', ticket_text),
    endpoint      => 'gemini-3.5-flash',
    output_schema => """category STRING OPTIONS(description = 'billing, technical, account or shipping'),
                        urgency STRING, sentiment STRING"""
  ).* EXCEPT (full_response, status)
FROM raw.support_tickets
WHERE created_date = CURRENT_DATE() - 1;

-- Managed functions: cheaper syntax for routine tasks
SELECT movie_id,
       AI.AGG(review, 'Summarize the overall sentiment towards the movie.') AS sentiment
FROM raw.reviews
GROUP BY movie_id;
```

### Gemini in SQL: table-valued functions (remote model)

```sql
-- The default connection avoids managing a named connection
CREATE OR REPLACE MODEL analytics.gemini_flash
  REMOTE WITH CONNECTION DEFAULT
  OPTIONS (ENDPOINT = 'gemini-3.8-flash');

-- AI.GENERATE_TABLE: the input column must be named `prompt`
SELECT *
FROM AI.GENERATE_TABLE(
  MODEL analytics.gemini_flash,
  (SELECT CONCAT('Extract the person: ', bio) AS prompt FROM raw.people),
  STRUCT('name STRING, age INT64, city STRING' AS output_schema));

-- Batch embeddings (AI.GENERATE_EMBEDDING replaces ML.GENERATE_EMBEDDING)
CREATE OR REPLACE MODEL analytics.embedder
  REMOTE WITH CONNECTION DEFAULT OPTIONS (ENDPOINT = 'gemini-embedding-001');
SELECT *
FROM AI.GENERATE_EMBEDDING(
  MODEL analytics.embedder,
  (SELECT doc_id, content FROM knowledge.documents));
```

### Apache Iceberg managed table

```sql
CREATE TABLE lake.orders_iceberg (
  order_id STRING, customer_id STRING, order_ts TIMESTAMP, amount NUMERIC
)
CLUSTER BY customer_id
WITH CONNECTION DEFAULT
OPTIONS (
  file_format  = 'PARQUET',
  table_format = 'ICEBERG',
  storage_uri  = 'gs://my-bucket/lake/orders_iceberg'
);
-- Create in an empty prefix and never modify its files from other engines.
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `AI.GENERATE` `endpoint` | `gemini-3.5-flash-lite` | Short model ID or full Agent Platform URL |
| `AI.GENERATE` `connection_id` | none (end-user credentials) | `PROJECT.LOCATION.CONNECTION` for service-account auth |
| `request_type` | `UNSPECIFIED` | `SHARED` (DSQ) or `DEDICATED` (Provisioned Throughput) |
| `max_batching_rows` | chosen by BigQuery | Rows per remote-function HTTP call |
| Iceberg `CLUSTER BY` | none | Up to four top-level, non-repeated columns |

## Example Usage

```bash
# Estimate bytes before running, then cap the bill for the real run
bq query --use_legacy_sql=false --dry_run 'SELECT ... FROM analytics.daily_revenue'
bq query --use_legacy_sql=false --maximum_bytes_billed=10737418240 'SELECT ...'
```

### Wrong

```sql
SELECT * FROM raw.orders LIMIT 10;   -- LIMIT does not reduce bytes billed on non-clustered tables
```

### Correct

```sql
SELECT order_id, amount FROM raw.orders
WHERE DATE(order_ts) = CURRENT_DATE() - 1;   -- select columns, filter the partition column
```

## See Also

- [BigQuery AI](../concepts/bigquery-ai.md) — function catalog, model IDs, renames
- [Cost Optimization](cost-optimization.md) — on-demand vs editions, maximum bytes billed
- [Cross-Platform Patterns](../concepts/cross-platform-patterns.md) — QUALIFY, MERGE, dialects

## Sources

- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-agg
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-table
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-generate-embedding
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-remote-model
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-time-series
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-forecast
- https://docs.cloud.google.com/bigquery/docs/remote-functions
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/data-definition-language
- https://docs.cloud.google.com/bigquery/docs/iceberg-tables
- https://docs.cloud.google.com/bigquery/docs/best-practices-costs
- https://cloud.google.com/blog/products/serverless/google-cloud-functions-is-now-cloud-run-functions
