# Delta Lake

> **Purpose**: Delta Lake 4.x (OSS 4.4.1) and Delta on Databricks — table features by version, Variant, type widening, identity columns, row tracking, catalog-managed tables, UniForm
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Delta Lake is an open table format: Parquet data files plus a transaction log (`_delta_log/`) that gives ACID commits, time travel and table features negotiated through reader/writer protocol versions. Current OSS release is **4.4.1 (2026-10-05, patch on 4.4.0 of 2026-08-20; builds for Spark 4.0.1/4.1/4.2, Scala 2.13 only)**. The 4.x line: 4.0.0 (2025-06-09: Variant GA, Delta Connect, DROP FEATURE without history truncation), 4.1.0 (2026-02-26: catalog-managed tables and server-side planning in preview, conflict-free enablement of deletion vectors and column mapping), 4.2.0 (2026-04-16: Kernel-based Flink connector, experimental), 4.3.0 (2026-06-18: Unity Catalog Delta REST API by default for managed tables), 4.4.0 (2026-08-20: identity columns through SQL DDL, Spark 4.2). On Databricks, UC managed Delta tables add predictive optimization, `CLUSTER BY AUTO` and catalog commits — see [liquid-clustering](liquid-clustering.md) and [unity-catalog-iceberg](unity-catalog-iceberg.md).

## The Concept

```sql
-- UC managed Delta table (Databricks): platform-managed layout and maintenance
CREATE TABLE main.analytics.events (
  event_id   STRING NOT NULL,
  user_id    STRING,
  event_ts   TIMESTAMP,
  event_data VARIANT                      -- DBR 15.4+ / Delta OSS 4.0+
)
CLUSTER BY AUTO                           -- DBR 15.4 LTS+, UC managed only
TBLPROPERTIES ('delta.enableRowTracking' = 'true');   -- DBR 14.0+ / OSS 3.2+

SELECT event_data:user_agent::STRING FROM main.analytics.events;

-- Type widening without rewriting files (DBR 15.4 LTS+ / OSS GA 4.0)
ALTER TABLE main.analytics.orders SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');
ALTER TABLE main.analytics.orders ALTER COLUMN quantity TYPE BIGINT;   -- INT -> BIGINT

-- Catalog-managed table (OSS 4.0.1+ / Databricks "catalog commits", DBR 16.4+)
CREATE TABLE main.analytics.sales (sale_id BIGINT, amount DECIMAL(10,2))
TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');

-- Remove a table feature and downgrade the protocol (DBR 16.3+; conflicts with all writes)
ALTER TABLE main.analytics.legacy DROP FEATURE deletionVectors;
```

## Quick Reference

| Feature | Delta OSS | Databricks Runtime |
|---|---|---|
| Change data feed (legacy, `delta.enableChangeDataFeed`) | 2.0+ | Supported; automatic CDF (row tracking, no table property) DBR 19+ |
| Deletion vectors | 2.3+ read; DELETE 2.4, UPDATE 3.0 | Read 12.2 LTS+; all write optimizations 14.3 LTS+ |
| UniForm / Iceberg reads (IcebergCompatV2) | 3.1+ (Hudi 3.2+, preview) | 14.3 LTS+; not with deletion vectors |
| Liquid clustering | `CLUSTER BY` 3.1+; `ALTER`/`OPTIMIZE FULL` 3.3+ | GA 15.4 LTS+; `CLUSTER BY AUTO` 15.4 LTS+ (UC managed) |
| Row tracking (`_metadata.row_id`, `row_commit_version`) | 3.2+ (backfill on existing tables 3.3+) | 14.0+ |
| Type widening | Preview 3.2, GA 4.0 | 15.4 LTS+ (`VOID` -> any type 18.2+) |
| Identity columns | 3.3+ (DeltaTable API); SQL DDL in Delta Spark 4.4+ | SQL 10.4 LTS+; Python/Scala API 16.0+ |
| Variant | 4.0+ (shredding preview) | 15.4+ |
| Catalog-managed tables / catalog commits | 4.0.1+ (preview 4.1) | 16.4+ create/read/write; 18.0+ enable on existing |
| `VACUUM ... LITE` | 3.3+ | 16.4 LTS+ (Public Preview) |
| Predictive optimization (OPTIMIZE, VACUUM, ANALYZE) | No | UC managed tables only |

| Identity column rule (Databricks docs) | Consequence |
|---|---|
| Declaring one **disables concurrent transactions** on the table | Do not use on tables with concurrent writers (parallel MERGE/INSERT jobs) |
| BIGINT only; values unique but not contiguous | Never rely on gap-free keys |
| Cannot be added, replaced or changed with `ALTER TABLE`; not in CTAS; not a partition column | Define in `CREATE TABLE`, then `INSERT` |

## Common Mistakes

### Wrong

```sql
OPTIMIZE orders ZORDER BY (region, order_date);   -- error on a liquid-clustered table
VACUUM orders RETAIN 0 HOURS;                      -- breaks time travel and running readers
TBLPROPERTIES ('delta.catalogManaged' = 'true');   -- not a real property
```

### Correct

```sql
OPTIMIZE orders;                                   -- incremental clustering
VACUUM orders RETAIN 168 HOURS;                    -- default/minimum 7 days
TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');
-- The older "coordinated commits" preview is superseded by catalog-managed tables
-- (Delta 4.0 `catalogOwned-preview`, then `catalogManaged` from 4.0.1).
```

## Related

- [liquid-clustering](liquid-clustering.md)
- [unity-catalog-iceberg](unity-catalog-iceberg.md)
- [iceberg-v3](iceberg-v3.md)
- [delta-operations pattern](../patterns/delta-operations.md)

## Sources

- https://github.com/delta-io/delta/releases
- https://docs.delta.io/latest/versioning.html
- https://docs.delta.io/delta-catalog-managed-tables/
- https://github.com/delta-io/delta/releases/tag/v4.0.0
- https://docs.databricks.com/aws/en/tables/features/generated-columns
- https://docs.databricks.com/aws/en/tables/features/type-widening
- https://docs.databricks.com/aws/en/tables/features/deletion-vectors
- https://docs.databricks.com/aws/en/tables/features/row-tracking
- https://docs.databricks.com/aws/en/tables/features/change-data-feed
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://docs.databricks.com/aws/en/tables/features/drop-feature
- https://docs.databricks.com/aws/en/tables/operations/vacuum
