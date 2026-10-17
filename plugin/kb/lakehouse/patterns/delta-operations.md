# Delta Operations

> **Purpose**: Production Delta Lake operations — CREATE with liquid clustering, MERGE (with schema evolution), INSERT REPLACE, change data feed, OPTIMIZE/VACUUM vs predictive optimization, time travel, feature management
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Upserts and idempotent partial replaces on Delta tables (Databricks or Delta Lake OSS)
- Incremental downstream processing with change data feed
- Deciding which maintenance you still run by hand once predictive optimization is on

## Implementation

```sql
-- 1. Create (UC managed): CLUSTER BY AUTO needs predictive optimization; CDF set once at creation
CREATE TABLE main.silver.orders (
    order_id      STRING NOT NULL,
    customer_id   STRING NOT NULL,
    order_date    DATE NOT NULL,
    total_amount  DECIMAL(12,2),
    status        STRING,
    _updated_at   TIMESTAMP
)
CLUSTER BY AUTO
TBLPROPERTIES (
    'delta.enableChangeDataFeed' = 'true',            -- legacy CDF; see automatic CDF below
    'delta.logRetentionDuration' = 'interval 30 days',
    'delta.deletedFileRetentionDuration' = 'interval 7 days'
);

-- 2. MERGE (WHEN NOT MATCHED BY SOURCE and UPDATE SET * EXCEPT: DBR 12.2 LTS+)
MERGE INTO main.silver.orders AS t
USING main.bronze.orders_batch AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s._updated_at > t._updated_at THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
WHEN NOT MATCHED BY SOURCE AND t.order_date < current_date() - INTERVAL 90 DAYS THEN DELETE;

-- 2b. Same MERGE, adding new source columns to the target (DBR 15.2+)
MERGE WITH SCHEMA EVOLUTION INTO main.silver.orders AS t
USING main.bronze.orders_batch AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- 3. Atomic partial replace (one commit, unlike DELETE + INSERT)
INSERT INTO main.gold.daily_sales
REPLACE WHERE sale_date >= DATE'2026-10-01'          -- DBR 12.2 LTS+
SELECT * FROM main.silver.daily_sales_v WHERE sale_date >= DATE'2026-10-01';

INSERT INTO main.gold.daily_sales
REPLACE USING (sale_date, store_id)                   -- DBR 16.3+; unpartitioned/clustered tables 17.2+
SELECT * FROM main.silver.daily_sales_recomputed;

-- 4. Change data feed reads
SELECT _change_type, order_id, total_amount, _commit_version, _commit_timestamp
FROM table_changes('main.silver.orders', 10, 20)
WHERE _change_type IN ('insert', 'update_postimage', 'delete');

-- 5. Maintenance: only when predictive optimization does NOT cover the table
OPTIMIZE main.silver.orders;                -- incremental clustering
OPTIMIZE main.silver.orders FULL;           -- after changing clustering keys (DBR 16.4 LTS+)
VACUUM main.silver.orders;                  -- honors delta.deletedFileRetentionDuration (min 7 days)
VACUUM main.silver.orders LITE;             -- log-based, faster (DBR 16.4 LTS+ Public Preview)

-- 6. Time travel and rollback
SELECT * FROM main.silver.orders VERSION AS OF 5;
SELECT * FROM main.silver.orders TIMESTAMP AS OF '2026-10-01';
RESTORE TABLE main.silver.orders TO VERSION AS OF 5;
DESCRIBE HISTORY main.silver.orders;
```

## Delta 4.x and Recent Databricks Operations

```sql
-- Variant (DBR 15.4+ / Delta 4.0+)
INSERT INTO main.silver.events
SELECT event_id, parse_json(raw_payload) AS event_data FROM main.bronze.events_raw;
SELECT event_data:user.name::STRING FROM main.silver.events;

-- Type widening (DBR 15.4 LTS+ / Delta 4.0 GA)
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');
ALTER TABLE main.silver.orders ALTER COLUMN quantity TYPE BIGINT;

-- Enabling deletion vectors no longer blocks concurrent writers (Delta 4.1 conflict-free enablement)
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('delta.enableDeletionVectors' = 'true');

-- INSERT with schema evolution (DBR 18.1+; Delta OSS 4.3+), instead of overwriteSchema
INSERT WITH SCHEMA EVOLUTION INTO main.silver.orders SELECT * FROM main.bronze.orders_batch;

-- Identity surrogate key: disables concurrent transactions on the table
CREATE TABLE main.gold.dim_product (
    product_sk BIGINT GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1),
    product_id STRING NOT NULL,
    product_name STRING NOT NULL
);
```

## Configuration

| Setting | Default | Description |
|---|---|---|
| `delta.deletedFileRetentionDuration` | 7 days | VACUUM window; predictive optimization keeps at least 7 days |
| `delta.logRetentionDuration` | 30 days | Time travel over history |
| `delta.enableChangeDataFeed` | false | Legacy CDF; set at creation (changing properties later conflicts with concurrent writes) |
| Automatic CDF | — | DBR 19+, UC-registered tables with row tracking (Delta) or Iceberg v3; no table property; Databricks readers only; cannot coexist with legacy CDF |
| Predictive optimization | Account default | Runs OPTIMIZE, VACUUM, ANALYZE on UC managed tables; never runs ZORDER |

## Example Usage

| Operation | Command | When |
|---|---|---|
| Upsert | `MERGE INTO ... WHEN MATCHED / NOT MATCHED / NOT MATCHED BY SOURCE` | Silver dedup/upsert |
| Partial replace | `INSERT INTO ... REPLACE WHERE / USING / ON` (`REPLACE ON` DBR 17.1+) | Recompute a slice atomically |
| Schema change | `ALTER TABLE ADD COLUMN`, type widening, `MERGE/INSERT WITH SCHEMA EVOLUTION` | Never `overwriteSchema` on shared tables |
| Maintenance | Nothing on UC managed + predictive optimization; else scheduled `OPTIMIZE`/`VACUUM` | External or legacy tables |
| Remove a feature | `ALTER TABLE t DROP FEATURE <feature>` (DBR 16.3+) | Restore compatibility with older clients |

```python
# Wrong: replaces the schema and partitioning under every consumer in one commit; streaming
# readers of the table fail on the schema change; column comments not carried by the
# DataFrame (e.g. on a recomputed column) are lost.
df.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable("main.gold.orders")

# Correct: change the schema explicitly (DDL), then write with the table's schema.
spark.sql("ALTER TABLE main.gold.orders ADD COLUMN channel STRING")
df.write.mode("overwrite").saveAsTable("main.gold.orders")
```

## See Also

- [delta-lake concept](../concepts/delta-lake.md)
- [liquid-clustering concept](../concepts/liquid-clustering.md)
- [iceberg-operations](iceberg-operations.md)
- [migration-to-open-formats](migration-to-open-formats.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-dml-insert-into
- https://docs.databricks.com/aws/en/tables/update-schema
- https://docs.databricks.com/aws/en/tables/features/change-data-feed
- https://docs.databricks.com/aws/en/tables/table-properties
- https://docs.databricks.com/aws/en/tables/operations/vacuum
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- https://docs.databricks.com/aws/en/tables/features/generated-columns
- https://docs.databricks.com/aws/en/structured-streaming/delta-lake
- https://docs.delta.io/delta-batch/
- https://github.com/delta-io/delta/releases/tag/v4.1.0
