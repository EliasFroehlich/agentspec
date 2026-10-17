# Snowflake Implementation Patterns

> **Purpose**: Production SQL patterns for Snowpipe auto-ingest, dynamic-table pipelines, streams + tasks CDC, Gen2/Adaptive warehouse sizing and Cortex AI Functions
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Continuous file ingestion from cloud storage into raw tables
- Declarative multi-hop transformations with freshness targets (dynamic tables)
- Imperative CDC with `MERGE` and deletes (streams + tasks)
- Sizing compute per workload; LLM enrichment directly in SQL

## Implementation

### Snowpipe auto-ingest (files to raw table)

```sql
-- S3: event notifications go to the pipe's SQS queue (see SHOW PIPES -> notification_channel).
-- INTEGRATION is required only for GCS/Azure stages or an S3 Multi-Queue Notification Integration.
CREATE OR REPLACE PIPE landing.events_pipe
  AUTO_INGEST = TRUE
AS
COPY INTO raw.events_landing
FROM @landing_stage/events/
FILE_FORMAT = (TYPE = 'PARQUET')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT SYSTEM$PIPE_STATUS('landing.events_pipe');
```

### Dynamic tables (declarative silver and gold)

```sql
-- Sources must be tables, views, Iceberg/Delta Direct tables, other dynamic tables or VALUES
-- (not stages, external tables, directory tables or streams). Minimum TARGET_LAG is 1 minute.
CREATE OR REPLACE DYNAMIC TABLE silver.events
  TARGET_LAG = '5 minutes'
  WAREHOUSE  = transform_wh          -- REFRESH_MODE defaults to AUTO; check SHOW DYNAMIC TABLES
AS
SELECT event_id, event_type, user_id, properties, event_ts, loaded_at
FROM raw.events_landing
WHERE event_id IS NOT NULL
QUALIFY ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY loaded_at DESC) = 1;

CREATE OR REPLACE DYNAMIC TABLE gold.daily_event_metrics
  TARGET_LAG = '1 hour'
  WAREHOUSE  = analytics_wh
AS
SELECT DATE_TRUNC('day', event_ts)     AS event_date,
       event_type,
       COUNT(*)                        AS event_count,
       COUNT(DISTINCT user_id)         AS unique_users,
       AVG(properties:duration::FLOAT) AS avg_duration
FROM silver.events
GROUP BY 1, 2;

-- Fixed clock schedule instead of a lag: no TARGET_LAG with SCHEDULER = DISABLE; refresh from a task
CREATE OR REPLACE DYNAMIC TABLE gold.daily_snapshot
  SCHEDULER = DISABLE
  WAREHOUSE = analytics_wh
AS SELECT event_date, SUM(event_count) AS events FROM gold.daily_event_metrics GROUP BY 1;

CREATE OR REPLACE TASK refresh_gold_daily
  WAREHOUSE = task_wh
  SCHEDULE  = 'USING CRON 0 5 * * * UTC'
AS ALTER DYNAMIC TABLE gold.daily_snapshot REFRESH;
```

### Streams + tasks (CDC with deletes)

```sql
CREATE OR REPLACE STREAM orders_stream ON TABLE raw.orders APPEND_ONLY = FALSE;

-- Triggered task: no SCHEDULE, runs when the stream has data
CREATE OR REPLACE TASK process_order_changes
  WAREHOUSE = task_wh
  WHEN SYSTEM$STREAM_HAS_DATA('orders_stream')
AS
MERGE INTO curated.orders t
USING (
  -- an UPDATE appears as DELETE + INSERT with METADATA$ISUPDATE = TRUE; keep only the INSERT half
  SELECT * FROM orders_stream
  WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE)
) s ON t.order_id = s.order_id
WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' THEN DELETE
WHEN MATCHED AND s.METADATA$ACTION = 'INSERT' THEN
  UPDATE SET t.amount = s.amount, t.status = s.status, t.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND s.METADATA$ACTION = 'INSERT' THEN
  INSERT (order_id, amount, status, created_at)
  VALUES (s.order_id, s.amount, s.status, CURRENT_TIMESTAMP());

ALTER TASK process_order_changes RESUME;
-- Polling alternative: SCHEDULE = '5 MINUTES' plus the same WHEN clause.
```

### Warehouse sizing (Gen2, multi-cluster, Adaptive)

```sql
-- Standard warehouses default to Gen2 where available; set it explicitly in IaC
CREATE OR REPLACE WAREHOUSE transform_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  GENERATION     = '2'
  AUTO_SUSPEND   = 60
  AUTO_RESUME    = TRUE;

-- BI concurrency: multi-cluster scale-out
ALTER WAREHOUSE analytics_wh SET
  MIN_CLUSTER_COUNT = 1  MAX_CLUSTER_COUNT = 6  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300;

-- Adaptive Warehouse (Enterprise Edition+): Snowflake sizes each query; billed per query
CREATE ADAPTIVE WAREHOUSE adhoc_wh
  WITH MAX_QUERY_PERFORMANCE_LEVEL = XLARGE
       QUERY_THROUGHPUT_MULTIPLIER = 4;
-- Convert in place: ALTER WAREHOUSE my_wh SET WAREHOUSE_TYPE = 'ADAPTIVE';
```

### Cortex AI Functions in SQL

```sql
SELECT ticket_id,
       AI_CLASSIFY(description, ['billing', 'technical', 'account', 'other']):labels[0]::STRING AS category,
       AI_COMPLETE(
         model    => 'claude-sonnet-5',
         prompt   => CONCAT('Draft a one-line reply to: ', description),
         model_parameters => {'temperature': 0.2, 'max_tokens': 120}) AS reply
FROM support.tickets
WHERE created_at >= CURRENT_DATE() - 1
  AND AI_FILTER(PROMPT('Is this ticket a complaint? {0}', description));

-- Embeddings for semantic search (output dimension comes from the model)
SELECT doc_id, AI_EMBED('snowflake-arctic-embed-l-v2.0', content) AS embedding
FROM knowledge.documents;
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `TARGET_LAG` | required with the scheduler on; not allowed with `SCHEDULER = DISABLE` | Max staleness (minimum 1 minute) or `DOWNSTREAM` |
| `REFRESH_MODE` | `AUTO` | `INCREMENTAL` fails fast if the query cannot refresh incrementally |
| `GENERATION` | `'2'` where available | `'1'` = original standard warehouses |
| `AUTO_SUSPEND` | 600 s | Seconds idle before suspend; 60 is a common floor |
| `MAX_QUERY_PERFORMANCE_LEVEL` | `XLARGE` | Adaptive: per-query performance ceiling |
| `QUERY_THROUGHPUT_MULTIPLIER` | `2` | Adaptive: aggregate throughput budget |

## Example Usage

### Wrong

```sql
-- Legacy namespace functions (BCR 2026_08 removes them in inactive accounts) and a legacy model
SELECT SNOWFLAKE.CORTEX.COMPLETE('mistral-large2', prompt) FROM t;
SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_1024('snowflake-arctic-embed-l-v2.0', content) FROM docs;
-- Dynamic table reading a stage: not a supported source
CREATE DYNAMIC TABLE bronze.raw TARGET_LAG = '1 minute' WAREHOUSE = wh AS SELECT $1 FROM @stage;
```

### Correct

```sql
SELECT AI_COMPLETE('claude-sonnet-5', prompt) FROM t;
SELECT AI_EMBED('snowflake-arctic-embed-l-v2.0', content) FROM docs;
-- Land files with Snowpipe/COPY into a table, then build dynamic tables on that table.
```

## See Also

- [Snowflake Cortex](../concepts/snowflake-cortex.md) — function catalog, models, platform map
- [Cross-Platform Patterns](../concepts/cross-platform-patterns.md) — dialect differences
- [Cost Optimization](cost-optimization.md) — credit prices, resource monitors

## Sources

- https://docs.snowflake.com/en/sql-reference/sql/create-pipe
- https://docs.snowflake.com/en/sql-reference/sql/create-dynamic-table
- https://docs.snowflake.com/en/user-guide/dynamic-tables/supported-queries
- https://docs.snowflake.com/en/user-guide/dynamic-tables/decision-guide
- https://docs.snowflake.com/en/sql-reference/sql/create-stream
- https://docs.snowflake.com/en/sql-reference/sql/create-task
- https://docs.snowflake.com/en/sql-reference/sql/create-warehouse
- https://docs.snowflake.com/en/user-guide/warehouses-gen2
- https://docs.snowflake.com/en/user-guide/warehouses-adaptive
- https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-migrate-legacy-functions
- https://docs.snowflake.com/en/sql-reference/functions/ai_filter
- https://docs.snowflake.com/en/sql-reference/functions/prompt
- https://docs.snowflake.com/en/release-notes/bcr-bundles/2026_08/bcr-2432
