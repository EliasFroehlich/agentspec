# Migration to Open Formats

> **Purpose**: Hive/Parquet to Iceberg, Parquet to Delta, external Delta to Unity Catalog managed, partitioned Delta to liquid clustering, Delta to Iceberg readers — plus validation queries
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Moving legacy Hive or raw Parquet tables to an open table format
- On Databricks: turning external/legacy Delta tables into UC managed tables with liquid clustering
- Exposing Delta tables to Iceberg engines without copying data

## Implementation

```sql
-- ============================================================
-- Strategy 1: Hive/Parquet -> Iceberg in place (metadata only; Iceberg Spark extensions)
-- ============================================================
CALL catalog.system.migrate('spark_catalog.analytics.orders');   -- replaces the Hive table
-- or keep the source table and register files into an existing Iceberg table:
-- CALL spark_catalog.system.add_files(table => 'db.orders_iceberg', source_table => 'db.orders_parquet');
DESCRIBE EXTENDED catalog.analytics.orders;                      -- Provider = iceberg
ALTER TABLE catalog.analytics.orders SET TBLPROPERTIES ('format-version' = '3');

-- ============================================================
-- Strategy 2: Parquet -> Delta (adds _delta_log, no rewrite)
-- ============================================================
CONVERT TO DELTA parquet.`s3://data-lake/raw/orders/` PARTITIONED BY (order_date DATE);
-- The result is an external (path-based) Delta table. On Databricks, register it in UC,
-- then continue with strategies 3 and 4.

-- ============================================================
-- Strategy 3 (Databricks): external Delta -> UC managed (DBR 17.3 LTS+ or serverless)
-- Keeps name, settings, permissions, views and history; path-based access is redirected
-- ============================================================
ALTER TABLE main.bronze.orders SET MANAGED;               -- external table: no MOVE/COPY
-- Foreign Delta table (from a federated catalog): ALTER TABLE ... SET MANAGED MOVE | COPY;

-- ============================================================
-- Strategy 4 (Databricks): partitioned Delta -> liquid clustering (DBR 18.1+, DBR 18 LTS)
-- ============================================================
ALTER TABLE main.bronze.orders REPLACE PARTITIONED BY WITH CLUSTER BY AUTO;  -- UC managed only
-- External table or explicit keys: ... WITH CLUSTER BY (order_date, customer_id); then OPTIMIZE
-- Not on DBR 17.3 LTS: there, recreate with CLUSTER BY + INSERT, validate, and swap.

-- ============================================================
-- Strategy 5 (Databricks): let Iceberg clients read a Delta table (Iceberg reads / UniForm)
-- ============================================================
ALTER TABLE main.gold.orders SET TBLPROPERTIES (
    'delta.columnMapping.mode' = 'name',
    'delta.enableIcebergCompatV2' = 'true',
    'delta.universalFormat.enabledFormats' = 'iceberg');   -- DBR 15.4 LTS+ on existing tables
-- Table has deletion vectors? Use:
-- REORG TABLE main.gold.orders APPLY (UPGRADE UNIFORM(ICEBERG_COMPAT_VERSION = 2));

-- ============================================================
-- Validation: compare source vs target
-- ============================================================
SELECT 'source' AS dataset, COUNT(*) AS row_count FROM legacy.analytics.orders
UNION ALL
SELECT 'target', COUNT(*) FROM main.analytics.orders;

SELECT 'source' AS dataset, COUNT(*) AS rows, SUM(HASH(order_id)) AS id_checksum,
       SUM(total_amount) AS amount_sum, COUNT(DISTINCT customer_id) AS unique_customers
FROM legacy.analytics.orders
UNION ALL
SELECT 'target', COUNT(*), SUM(HASH(order_id)), SUM(total_amount), COUNT(DISTINCT customer_id)
FROM main.analytics.orders;

-- Row-level diff (both directions)
(SELECT * FROM legacy.analytics.orders EXCEPT SELECT * FROM main.analytics.orders)
UNION ALL
(SELECT * FROM main.analytics.orders EXCEPT SELECT * FROM legacy.analytics.orders);
```

## Configuration

| Migration path | Method | Data rewrite? | Notes |
|---|---|---|---|
| Hive -> Iceberg | `system.migrate()` | No | Replaces the source table |
| Parquet -> Iceberg | `system.add_files()` | No | Imports files into an existing Iceberg table |
| Parquet -> Delta | `CONVERT TO DELTA` | No | Creates an external table |
| External Delta -> UC managed | `ALTER TABLE ... SET MANAGED` | Files moved to managed storage | Brief writer downtime; `UNSET MANAGED` rolls back within 14 days |
| Partitioned Delta -> liquid clustering | `REPLACE PARTITIONED BY WITH CLUSTER BY` | No full rewrite; run `OPTIMIZE` | DBR 18.1+; not for Lakeflow ST/MV (change `PARTITIONED BY` to `CLUSTER BY` in the definition) |
| Delta -> Iceberg readers | Iceberg reads (IcebergCompatV2/V3) | No (REORG rewrites if DVs exist) | V3 with deletion vectors: DBR 18 LTS+ |
| Hive -> Delta | CTAS or streaming copy | Yes | Prefer CTAS into a UC managed table |
| Any -> Any (large) | Dual-write + cutover | Yes | Blue-green with validation |

## Example Usage

```sql
-- Check the result on Databricks
DESCRIBE EXTENDED main.bronze.orders;   -- Type = MANAGED, clustering columns, clusterByAuto
DESCRIBE HISTORY main.bronze.orders;    -- REORG / UPGRADE PROTOCOL / REPLACE PARTITIONED BY steps
```

## See Also

- [iceberg-operations](iceberg-operations.md)
- [delta-operations](delta-operations.md)
- [liquid-clustering concept](../concepts/liquid-clustering.md)
- [unity-catalog-iceberg concept](../concepts/unity-catalog-iceberg.md)
- [catalog-setup](catalog-setup.md)

## Sources

- https://iceberg.apache.org/docs/latest/spark-procedures/
- https://docs.databricks.com/aws/en/tables/convert-to-managed
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/delta/iceberg-reads
- https://docs.databricks.com/aws/en/tables/managed
