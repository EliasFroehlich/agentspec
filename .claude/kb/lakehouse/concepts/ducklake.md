# DuckLake

> **Purpose**: DuckLake 1.0 — lakehouse format with a SQL-database catalog, Parquet data, data inlining, Iceberg interop; reference implementation is the DuckDB `ducklake` extension
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

DuckLake is an open lakehouse format in which a SQL database stores all metadata (snapshots, schemas, file lists, statistics) while data lives as Parquet on local disk or object storage. **DuckLake v1.0 (2026-04-13)** is declared production-ready with guaranteed backward compatibility; its reference implementation is the `ducklake` extension shipped with **DuckDB 1.5.2**. v1.0 brought full inlining of small inserts, updates and deletes into the catalog database, sorted tables, Iceberg-compatible bucket partitioning and Iceberg-compatible deletion vectors (Puffin). Besides DuckDB there are clients for Apache Spark, Trino, Apache DataFusion and pandas, and a hosted service from MotherDuck. Planned next: spec v1.1 (variant inlining, multi-deletion-vector Puffin files); v2.0 ideas include permission-based roles.

## The Concept

```sql
INSTALL ducklake;
INSTALL postgres;

-- Multi-user lakehouse: PostgreSQL catalog (12+), data in object storage
ATTACH 'ducklake:postgres:dbname=ducklake_catalog host=localhost' AS lake
    (DATA_PATH 's3://my-bucket/lake/');
USE lake;

CREATE SCHEMA IF NOT EXISTS analytics;
CREATE TABLE analytics.events (
    event_id   VARCHAR,
    user_id    VARCHAR,
    event_type VARCHAR,
    event_ts   TIMESTAMP,
    payload    JSON
);

INSERT INTO analytics.events
SELECT * FROM read_parquet('s3://raw-data/events/*.parquet');

-- Layout options added in v1.0
ALTER TABLE analytics.events SET SORTED BY (event_ts ASC);
ALTER TABLE analytics.events SET PARTITIONED BY (bucket(8, user_id));

-- Time travel
SELECT * FROM analytics.events AT (VERSION => 3);
SELECT * FROM analytics.events AT (TIMESTAMP => now() - INTERVAL '1 week');
```

## Quick Reference

| Catalog database | When (DuckLake docs) |
|---|---|
| DuckDB file (`ATTACH 'ducklake:metadata.ducklake' AS lake (DATA_PATH '...')`) | Single client only |
| SQLite (`ducklake:sqlite:metadata.sqlite`) | Several local clients |
| PostgreSQL (`ducklake:postgres:...`) | Multi-user lakehouse with remote clients |
| MySQL 8+ | Documented but **not recommended** (known connector issues) |

| Feature | DuckLake 1.0 | Iceberg v3 | Delta 4.x |
|---|---|---|---|
| Metadata store | SQL database | Metadata files + catalog | `_delta_log` + catalog |
| Multi-table transactions | Yes (catalog database ACID) | Catalog-dependent | Catalog-managed tables / catalog commits |
| Small writes | Data inlining in the catalog | Small files + compaction | Small files + compaction |
| Partitioning | `SET PARTITIONED BY` (incl. `bucket()` since 1.0); sorted tables | Hidden partitioning + evolution | Liquid clustering |
| Deletes | Delete files; Iceberg-compatible deletion vectors | Deletion vectors (v3) | Deletion vectors |
| Engines | DuckDB; Spark, Trino, DataFusion clients | Broad | Broad |

| Version | Date | Highlights |
|---|---|---|
| 0.1 | May 2025 | Initial specification and DuckDB extension |
| 0.2 | 2025-07-04 | Spec update (see release calendar) |
| 0.3 | 2025-09-17 | Iceberg interop, geometry, `MERGE INTO`, `CHECKPOINT` |
| 1.0 | 2026-04-13 | Production-ready spec; inlining of updates/deletes, sorted tables, bucket partitioning, deletion vectors |

## Iceberg Interop (since 0.3)

```sql
INSTALL iceberg;
LOAD iceberg;
-- Deep copy (data + metadata of the latest snapshot), either direction
COPY FROM DATABASE iceberg_datalake TO my_ducklake;
COPY FROM DATABASE my_ducklake TO iceberg_datalake;   -- target schemas must exist
-- Metadata-only copy, Iceberg -> DuckLake only, keeps snapshot history
CALL iceberg_to_ducklake('iceberg_datalake', 'ducklake');
```

## Common Mistakes

### Wrong

```sql
-- DuckDB-file catalog shared by several writers or services
ATTACH 'ducklake:metadata.ducklake' AS lake (DATA_PATH 's3://shared/lake/');
-- "COPY FROM DATABASE" assumed to be metadata-only: it copies the data
```

### Correct

```sql
-- Several local clients: SQLite catalog; remote/multi-user: PostgreSQL catalog
ATTACH 'ducklake:postgres:dbname=ducklake_catalog host=db.internal' AS lake
    (DATA_PATH 's3://shared/lake/');
-- Metadata-only Iceberg import: CALL iceberg_to_ducklake(...)
```

## Related

- [iceberg-v3](iceberg-v3.md)
- [delta-lake](delta-lake.md)
- [duckdb concept](../../modern-stack/concepts/duckdb.md)

## Sources

- https://ducklake.select/2026/04/13/ducklake-10/
- https://ducklake.select/2025/09/17/ducklake-03/
- https://ducklake.select/release_calendar
- https://ducklake.select/docs/stable/duckdb/usage/choosing_a_catalog_database
- https://ducklake.select/docs/stable/duckdb/usage/connecting
- https://ducklake.select/docs/stable/duckdb/usage/time_travel
