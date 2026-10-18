# Iceberg Operations

> **Purpose**: CREATE TABLE, MERGE INTO, time travel, partition evolution, compaction, snapshot management
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Core Iceberg table operations for production use with Spark and the Iceberg runtime (library 1.12.0): table creation with hidden partitioning, upsert via MERGE, time travel queries, and maintenance procedures (compaction, snapshot expiration, orphan removal). `ALTER TABLE ... PARTITION FIELD` and `CALL ... system.*` need the Iceberg Spark SQL extensions. On Databricks managed Iceberg tables the rules differ — see the last section.

## The Pattern

```sql
-- ============================================================
-- Iceberg Table Lifecycle: Create → Write → Query → Maintain
-- ============================================================

-- 1. CREATE with hidden partitioning
CREATE TABLE catalog.bronze.raw_events (
    event_id        STRING NOT NULL,
    user_id         STRING NOT NULL,
    event_type      STRING NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    properties      MAP<STRING, STRING>,
    _ingested_at    TIMESTAMP NOT NULL
)
USING iceberg
PARTITIONED BY (day(event_timestamp), bucket(8, user_id))   -- days() is the older, still accepted form
TBLPROPERTIES (
    'format-version' = '3',
    'write.parquet.compression-codec' = 'zstd',
    'commit.retry.num-retries' = '4'
);

-- 2. MERGE INTO (upsert pattern)
MERGE INTO catalog.silver.dim_users AS target
USING catalog.bronze.staging_users AS source
ON target.user_id = source.user_id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
    UPDATE SET
        target.user_name = source.user_name,
        target.email = source.email,
        target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (user_id, user_name, email, created_at, updated_at)
    VALUES (source.user_id, source.user_name, source.email, source.created_at, source.updated_at);

-- 3. INSERT OVERWRITE (partition-level replace)
INSERT OVERWRITE catalog.silver.daily_aggregates
SELECT
    DATE(event_timestamp) AS event_date,
    event_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users
FROM catalog.bronze.raw_events
WHERE DATE(event_timestamp) = '2026-03-26'
GROUP BY 1, 2;

-- 4. Time travel queries
SELECT * FROM catalog.bronze.raw_events VERSION AS OF 12345;           -- by snapshot ID
SELECT * FROM catalog.bronze.raw_events TIMESTAMP AS OF '2026-03-25'; -- by timestamp

-- 5. Partition evolution (metadata-only; daily -> hourly does not require dropping day())
ALTER TABLE catalog.bronze.raw_events ADD PARTITION FIELD hour(event_timestamp);

-- 6. Maintenance: compaction (rewrite small files)
CALL catalog.system.rewrite_data_files(
    table => 'catalog.bronze.raw_events',
    strategy => 'sort',
    sort_order => 'event_timestamp ASC'
);

-- 7. Expire old snapshots (free metadata storage)
CALL catalog.system.expire_snapshots(
    table => 'catalog.bronze.raw_events',
    older_than => TIMESTAMP '2026-03-19',  -- keep 7 days
    retain_last => 10
);

-- 8. Remove orphan files (unreferenced data files)
CALL catalog.system.remove_orphan_files(
    table => 'catalog.bronze.raw_events',
    older_than => TIMESTAMP '2026-03-19'
);
```

## Quick Reference

| Operation | Command | Frequency |
|-----------|---------|-----------|
| Compaction | `rewrite_data_files` | Daily or when small file ratio > 50% |
| Snapshot expiration | `expire_snapshots` | Daily, retain 7-30 days |
| Orphan removal | `remove_orphan_files` | Weekly |
| Metadata rewrite | `rewrite_manifests` | Monthly or after large schema changes |

Frequencies above are starting points, not documented defaults — tune to file counts and commit rate.

## Iceberg on Databricks (Unity Catalog managed Iceberg)

| Topic | Databricks behavior |
|---|---|
| Layout | Liquid clustering (`CLUSTER BY`); `PARTITIONED BY` columns are treated as clustering keys; expression transforms (`day()`, `bucket()`, ...) are not supported |
| Partition evolution | Only from external engines through the Iceberg REST Catalog, not from Databricks SQL |
| Maintenance | Predictive optimization (required for managed Iceberg) handles compaction and snapshot expiration — do not schedule `CALL system.*` |
| Row-level deletes | v2 position/equality deletes unsupported; v3 deletion vectors (DBR 18 LTS+) |
| Unsupported | Branching/tagging, UUID, `Fixed(L)`, TIME, compression codec changes (Zstd only) |

## Related

- [iceberg-v3 concept](../concepts/iceberg-v3.md)
- [unity-catalog-iceberg concept](../concepts/unity-catalog-iceberg.md)
- [delta-operations](delta-operations.md)
- [migration-to-open-formats](migration-to-open-formats.md)

## Sources

- https://iceberg.apache.org/docs/latest/spark-ddl/
- https://iceberg.apache.org/docs/latest/spark-procedures/
- https://iceberg.apache.org/docs/latest/spark-queries/
- https://iceberg.apache.org/releases/
- https://docs.databricks.com/aws/en/iceberg/
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-using
