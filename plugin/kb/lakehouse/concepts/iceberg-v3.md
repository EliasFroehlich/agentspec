# Iceberg v3

> **Purpose**: Apache Iceberg format spec v1–v3 (v4 in development), library 1.12.0, hidden partitioning, partition evolution, deletion vectors, REST catalog, Iceberg on Databricks
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Apache Iceberg is an open table format with snapshot-based metadata (metadata file → manifest list → manifests → data files). Spec **v2** added row-level deletes (position and equality delete files); **v3** — complete and adopted — adds new types (nanosecond timestamps, `unknown`, `variant`, `geometry`, `geography`), column default values, multi-argument transforms, row lineage, **binary deletion vectors** and table encryption keys. **v4 is under active development and not formally adopted.** The Java library is at **1.12.0 (2026-09-29)**, which drops Spark 3.4 and Flink 2.0 support and adds a v4 manifest reader. Hidden partitioning decouples layout from queries: users filter on `event_ts`, the engine prunes `day(event_ts)` partitions.

## The Concept

```sql
-- Spark SQL with the Iceberg runtime: v3 table with hidden partitioning
CREATE TABLE catalog.analytics.events (
    event_id        STRING,
    user_id         STRING,
    event_type      STRING,
    event_timestamp TIMESTAMP,
    payload         STRING
)
USING iceberg
PARTITIONED BY (day(event_timestamp), bucket(16, user_id))   -- days()/hours() still accepted
TBLPROPERTIES (
    'format-version' = '3',
    'write.delete.mode' = 'merge-on-read',
    'write.update.mode' = 'merge-on-read',
    'write.merge.mode'  = 'merge-on-read'
);

-- Partition evolution (needs the Iceberg Spark SQL extensions): metadata-only, old files
-- keep their spec. Daily -> hourly does not require dropping the daily field.
ALTER TABLE catalog.analytics.events ADD PARTITION FIELD hour(event_timestamp);

-- Row-level delete: in v3 merge-on-read writes a deletion vector (Puffin blob)
DELETE FROM catalog.analytics.events
WHERE user_id = 'user-to-forget' AND event_timestamp < TIMESTAMP '2025-01-01 00:00:00';
```

## Quick Reference

| Feature | v2 | v3 |
|---|---|---|
| Row-level deletes | Position + equality delete files | **Binary deletion vectors** (Puffin `deletion-vector-v1`) |
| Types | Primitive, nested | + `timestamp_ns`, `timestamptz_ns`, `unknown`, `variant`, `geometry`, `geography` |
| Default values | No | `initial-default` / `write-default` |
| Transforms | Single-argument | + multi-argument transforms |
| Row lineage | No | Yes |
| Encryption | — | Table encryption keys |

| Iceberg v4 (spec, not adopted) | Purpose |
|---|---|
| Adaptive metadata tree | Efficient small commits, column updates, columnar statistics |
| Data file + deletion vector in one entry | Removes two-phase planning |
| Relative locations in metadata | Relocate tables without rewriting metadata |
| No new equality deletes | Writing equality deletes no longer allowed |

| Catalog layer | Notes |
|---|---|
| REST Catalog | Defined by the Iceberg project as an OpenAPI spec, independent of format version; implemented by Apache Polaris, Apache Gravitino, Unity Catalog (Databricks and OSS 0.7+) and others |
| Hive / JDBC / Glue | Supported catalog implementations; REST is the interoperable default |

### Iceberg on Databricks (Unity Catalog)

| Topic | Behavior |
|---|---|
| Spec versions | Parquet only; spec v1, v2 and v3 |
| Managed Iceberg (GA 2026-05-21) | `USING iceberg`; layout via liquid clustering; `PARTITIONED BY` columns become clustering keys; `day()`/`bucket()` transforms unsupported |
| Iceberg v2 row-level deletes | Position and equality deletes **not supported**; use v3 deletion vectors |
| Iceberg v3 features | DBR 18 LTS+ (not on DBR 17.3 LTS) |
| Maintenance | Predictive optimization (required for managed Iceberg) instead of `CALL system.*` procedures |
| Not supported | Branching/tagging, UUID, `Fixed(L)`, TIME types |

Details: [unity-catalog-iceberg](unity-catalog-iceberg.md).

## Common Mistakes

### Wrong

```sql
-- Derived partition column: users must filter on it, and it can drift from event_timestamp
CREATE TABLE events (event_timestamp TIMESTAMP, event_date DATE, ...) USING iceberg
PARTITIONED BY (event_date);
```

### Correct

```sql
-- Hidden partitioning: filter on event_timestamp, pruning is automatic
CREATE TABLE events (event_timestamp TIMESTAMP, ...) USING iceberg
PARTITIONED BY (day(event_timestamp));
-- On Databricks managed Iceberg, use CLUSTER BY (event_timestamp) instead.
```

## Related

- [delta-lake](delta-lake.md)
- [unity-catalog-iceberg](unity-catalog-iceberg.md)
- [catalog-wars](catalog-wars.md)
- [iceberg-operations pattern](../patterns/iceberg-operations.md)

## Sources

- https://iceberg.apache.org/spec/
- https://iceberg.apache.org/releases/
- https://iceberg.apache.org/docs/latest/spark-ddl/
- https://iceberg.apache.org/rest-catalog-spec/
- https://docs.databricks.com/aws/en/iceberg/
- https://docs.databricks.com/aws/en/iceberg/iceberg-v3
- https://docs.databricks.com/aws/en/release-notes/product/2026/may
