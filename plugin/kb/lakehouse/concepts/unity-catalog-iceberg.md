# Unity Catalog Managed Tables, Iceberg and Catalog Commits

> **Purpose**: Databricks Unity Catalog as an open catalog — managed Delta/Iceberg tables, foreign Iceberg, Iceberg v3, Iceberg REST access, Iceberg reads (UniForm) and catalog commits
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

UC managed tables are the default and recommended table type on Databricks, for **Delta Lake or Apache Iceberg** (`USING iceberg`). Managed Iceberg, foreign Iceberg (read through Lakehouse Federation from AWS Glue, Hive metastore or Snowflake Horizon) and Iceberg v3 features became **GA on 2026-05-21**. External engines reach UC tables through the Iceberg REST Catalog (IRC) endpoint `/api/2.1/unity-catalog/iceberg-rest`, with credential vending. Catalog commits move commit coordination from storage into UC (`delta.feature.catalogManaged`).

| Feature | Requirement |
|---|---|
| Managed or foreign Iceberg tables | UC workspace, DBR 16.4 LTS+; managed Iceberg also needs serverless compute and predictive optimization |
| Iceberg v3 (deletion vectors, VARIANT, row lineage, GEOMETRY/GEOGRAPHY, write defaults) | DBR 18 LTS+ (**not available on DBR 17.3 LTS**) |
| Iceberg reads on Delta (UniForm, IcebergCompatV2) | DBR 14.3 LTS+ writers; column mapping; reader 2 / writer 7; **no deletion vectors** |
| IRC access from external engines | Metastore "External data access" + `EXTERNAL USE SCHEMA` grant; Iceberg client 1.9.2+ recommended |
| Catalog commits | DBR 16.4+ to read/write/create; DBR 18.0+ to enable or disable on an existing table |

## The Concept

```sql
-- Managed Iceberg table (DBR 16.4 LTS+); no LOCATION, no expression partitions
CREATE TABLE main.analytics.events (event_id STRING, event_ts TIMESTAMP, payload STRING)
USING iceberg;

-- Managed Iceberg v3 with liquid clustering (DBR 18 LTS+). On v2 tables, liquid
-- clustering requires turning off deletion vectors and row tracking; v3 supports both.
CREATE TABLE main.analytics.events_v3 (event_id STRING, event_ts TIMESTAMP, payload VARIANT)
USING iceberg
CLUSTER BY (event_ts)
TBLPROPERTIES ('format-version' = 3);

-- Delta table readable by Iceberg clients (Iceberg reads, IcebergCompatV2)
CREATE TABLE main.analytics.orders (order_id STRING, amount DECIMAL(12,2))
TBLPROPERTIES (
  'delta.columnMapping.mode' = 'id',
  'delta.enableIcebergCompatV2' = 'true',
  'delta.universalFormat.enabledFormats' = 'iceberg');

-- Catalog commits on a new managed table (multi-table transactions, governed writes)
CREATE TABLE main.analytics.sales (sale_id BIGINT, amount DECIMAL(10,2))
TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');

-- Let an external engine use the IRC endpoint
GRANT EXTERNAL USE SCHEMA ON SCHEMA main.analytics TO `etl-service-principal`;
```

External Spark/Trino/Flink clients point their Iceberg REST catalog `uri` at `https://<workspace-url>/api/2.1/unity-catalog/iceberg-rest`.

## Quick Reference

| Table type | Writable from Databricks | Writable externally (IRC) | Notes |
|---|---|---|---|
| Managed Delta | Yes | With catalog commits (Beta) | Predictive optimization, `CLUSTER BY AUTO`, metadata caching |
| Managed Delta + Iceberg reads | Yes | No (read-only as Iceberg) | Zstd Parquet; async Iceberg metadata generation |
| Managed Iceberg | Yes | Yes | `PARTITIONED BY` columns act as clustering keys; partition evolution only from external engines; `days()`/`bucket()` transforms unsupported |
| Foreign Iceberg | No (read-only) | Owned by the foreign catalog | Run `REFRESH FOREIGN TABLE` for the latest snapshot |

| Catalog commits limitation | Detail |
|---|---|
| Enable/disable | Not through `CREATE OR REPLACE`; use `CREATE TABLE ... TBLPROPERTIES` or `ALTER TABLE` |
| Streaming tables / materialized views | Add the property to the definition and refresh; `ALTER TABLE ... SET TBLPROPERTIES` not supported; DBR 17.3+ |
| External access on ST/MV | Not compatible with catalog commits |

Delta Lake OSS ships the same idea as catalog-managed tables (`catalogManaged`, Delta 4.0.1+; preview in 4.1; UC Delta REST API by default for managed tables in 4.3).

## Common Mistakes

### Wrong

```sql
-- Deletion vectors cannot be enabled on a table with Iceberg reads (IcebergCompatV2)
CREATE TABLE main.analytics.orders (...) TBLPROPERTIES (
  'delta.enableDeletionVectors' = 'true',
  'delta.enableIcebergCompatV2' = 'true',
  'delta.universalFormat.enabledFormats' = 'iceberg');

-- Property that does not exist
TBLPROPERTIES ('delta.catalogManaged' = 'true');
```

### Correct

```sql
-- Iceberg v3 compatibility supports deletion vectors (DBR 18 LTS+)
CREATE TABLE main.analytics.orders (...) TBLPROPERTIES (
  'delta.enableDeletionVectors' = 'true',
  'delta.enableIcebergCompatV3' = 'true',
  'delta.universalFormat.enabledFormats' = 'iceberg');

TBLPROPERTIES ('delta.feature.catalogManaged' = 'supported');
```

## Related

- [liquid-clustering](liquid-clustering.md)
- [delta-lake](delta-lake.md)
- [iceberg-v3](iceberg-v3.md)
- [catalog-setup pattern](../patterns/catalog-setup.md)

## Sources

- https://docs.databricks.com/aws/en/tables/managed
- https://docs.databricks.com/aws/en/iceberg/
- https://docs.databricks.com/aws/en/iceberg/iceberg-v3
- https://docs.databricks.com/aws/en/external-access/iceberg
- https://docs.databricks.com/aws/en/delta/iceberg-reads
- https://docs.databricks.com/aws/en/tables/features/catalog-commits
- https://docs.databricks.com/aws/en/release-notes/product/2026/may
- https://docs.delta.io/delta-catalog-managed-tables/
