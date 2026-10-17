# SQL Streaming, Flows and CDC

> **Purpose**: Pipeline SQL for CDC (`CREATE FLOW ... AS AUTO CDC INTO`), append/backfill flows, joins, windows, watermarks and layout
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Applying a change feed to SCD Type 1/2 tables in SQL
- Feeding one streaming table from several sources or backfilling it once
- Joins, window functions and aggregations as materialized views

## Implementation

```sql
-- SCD Type 1 (current state)
CREATE OR REFRESH STREAMING TABLE customers;

CREATE FLOW customers_cdc AS AUTO CDC INTO customers
FROM STREAM(customers_cdc_feed)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
APPLY AS TRUNCATE WHEN operation = 'TRUNCATE'
SEQUENCE BY event_ts
COLUMNS * EXCEPT (operation, event_ts)
STORED AS SCD TYPE 1;

-- SCD Type 2 (history in __START_AT / __END_AT)
CREATE OR REFRESH STREAMING TABLE customers_history;

CREATE FLOW customers_history_cdc AS AUTO CDC INTO customers_history
FROM STREAM(customers_cdc_feed)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY STRUCT(event_ts, event_id)
COLUMNS * EXCEPT (operation)
STORED AS SCD TYPE 2
TRACK HISTORY ON * EXCEPT (last_seen_at);

-- Two append flows into one target, plus a one-time backfill
CREATE OR REFRESH STREAMING TABLE events;

CREATE FLOW events_web AS INSERT INTO events BY NAME
SELECT * FROM STREAM(main.raw.web_events);

CREATE FLOW events_mobile AS INSERT INTO events BY NAME
SELECT * FROM STREAM(main.raw.mobile_events);

CREATE FLOW events_backfill AS INSERT ONCE INTO events BY NAME
SELECT * FROM main.archive.events_2024;
```

### Materialized views: joins, windows, aggregates

```sql
CREATE OR REFRESH MATERIALIZED VIEW order_details
CLUSTER BY AUTO
AS SELECT o.order_id, o.order_date, c.customer_name
FROM orders o INNER JOIN customers c ON o.customer_id = c.customer_id;

CREATE OR REFRESH MATERIALIZED VIEW latest_order_per_customer AS
SELECT * FROM orders
QUALIFY row_number() OVER (PARTITION BY customer_id ORDER BY order_ts DESC) = 1;
```

### Streaming aggregates need a watermark

```sql
CREATE OR REFRESH STREAMING TABLE clicks_5min AS
SELECT ad_id, window(click_ts, '5 minutes') AS w, count(*) AS clicks
FROM STREAM(clicks_raw)
WATERMARK click_ts DELAY OF INTERVAL 3 MINUTES
GROUP BY ad_id, window(click_ts, '5 minutes');
```

## Configuration

| Item | Rule |
|------|------|
| `AUTO CDC` target | Must be a streaming table; only other AUTO CDC flows may also target it |
| `AUTO CDC` source | Streaming source (`STREAM(...)`), even with `ONCE` |
| `APPLY CHANGES INTO` | Still accepted; replaced by `AUTO CDC` with the same clauses |
| Inline form | `CREATE OR REFRESH STREAMING TABLE t FLOW AUTO CDC ...` is also valid |
| `INSERT ONCE` | Runs once; re-runs on full refresh |
| `REPLACE WHERE` / `REPLACE USING` flows | `REPLACE USING` is Beta and needs DBR 18.2+ |
| Window functions in MVs | Incremental only with `PARTITION BY` |
| `PIVOT` | Not supported in pipelines |
| Layout | Prefer `CLUSTER BY` / `CLUSTER BY AUTO`; Databricks recommends liquid clustering over Z-order |

## Example Usage

```sql
-- Wrong: legacy CDC statement and Z-order tuning
APPLY CHANGES INTO customers FROM STREAM(feed) KEYS (id) SEQUENCE BY ts;  -- Wrong: legacy
-- TBLPROPERTIES ("pipelines.autoOptimize.zOrderCols" = "customer_id")

-- Correct
CREATE FLOW customers_cdc AS AUTO CDC INTO customers
FROM STREAM(feed) KEYS (id) SEQUENCE BY ts;
-- CLUSTER BY AUTO on the target table
```

## See Also

- [SQL tables](sql-tables.md)
- [CDC with AUTO CDC](cdc-apply-changes.md)
- [Flows and sinks](flows-and-sinks.md)
- [Stateful processing](../reference/stateful-processing.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/sql-dev
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-flow
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-apply-changes-into
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-streaming-table
- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/concepts/flows
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/ldp/stateful-processing
