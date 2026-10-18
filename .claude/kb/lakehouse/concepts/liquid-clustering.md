# Liquid Clustering and Predictive Optimization

> **Purpose**: Data layout on Databricks — `CLUSTER BY`, `CLUSTER BY AUTO`, predictive optimization, and the few cases where partitioning or ZORDER still apply
> **Confidence**: 0.93
> **MCP Validated**: 2026-10-17

## Overview

Liquid clustering "replaces table partitioning and ZORDER": rows are organized by clustering keys that can be changed without rewriting existing data. Databricks recommends it for **all new tables**, including streaming tables and materialized views. On Unity Catalog (UC) managed tables, `CLUSTER BY AUTO` lets predictive optimization pick and change keys from the query workload, and predictive optimization runs `OPTIMIZE`, `VACUUM` and `ANALYZE` on serverless jobs compute, so you do not schedule them yourself.

| Capability | Minimum version |
|---|---|
| Liquid clustering GA for Delta | DBR 15.4 LTS+ (Delta Lake OSS: `CLUSTER BY` 3.1+, `ALTER ... CLUSTER BY` 3.3+) |
| Liquid clustering on Iceberg tables | DBR 16.4 LTS+ (Public Preview); managed Iceberg v3 with auto clustering DBR 18.0+ |
| `CLUSTER BY AUTO` | DBR 15.4 LTS+, UC managed tables with predictive optimization only |
| `OPTIMIZE t FULL` (recluster everything) | DBR 16.4 LTS+; `OPTIMIZE t FULL WHERE ...` DBR 18.1+ |
| Hierarchical clustering (`delta.liquid.hierarchicalClusteringColumns`) | DBR 17.1+ |
| `ALTER TABLE ... REPLACE PARTITIONED BY WITH CLUSTER BY` | DBR 18.1+ (in DBR 18 LTS; **not in DBR 17.3 LTS**) |
| Predictive optimization | Premium plan, DBR 12.2 LTS+ or SQL warehouses; default for accounts created on or after 2024-11-11 |

## The Concept

```sql
-- New UC managed table: let the platform choose keys and run maintenance
CREATE TABLE main.sales.orders (
  order_id    STRING,
  customer_id STRING,
  order_ts    TIMESTAMP,
  amount      DECIMAL(12,2)
) CLUSTER BY AUTO;

-- Known filter columns (max 4 keys); optional hints before switching to AUTO
ALTER TABLE main.sales.orders CLUSTER BY (customer_id, order_ts);
ALTER TABLE main.sales.orders CLUSTER BY AUTO;

-- Changing keys does not rewrite old data; force it when needed
OPTIMIZE main.sales.orders FULL;

-- Predictive optimization is inherited account -> catalog -> schema -> table
ALTER SCHEMA main.sales ENABLE PREDICTIVE OPTIMIZATION;
DESCRIBE TABLE EXTENDED main.sales.orders;  -- shows clusterByAuto / clusteringColumns

-- Legacy partitioned table -> liquid clustering (DBR 18.1+)
ALTER TABLE main.sales.events REPLACE PARTITIONED BY WITH CLUSTER BY AUTO;
```

DataFrame writers: `.clusterBy("col")` and `.option("clusterByAuto", "true")` (Python, DBR 16.4+) only apply when the table is created or replaced (overwrite mode); on an existing table change keys with `ALTER TABLE`.

## When partitioning or ZORDER still make sense

| Situation | Guidance (Databricks docs) |
|---|---|
| Table under 1 TB | Do not partition |
| 1 TB – 100 TB | Use liquid clustering instead of partitioning |
| 100 TB or more | Partitioning *might* help, but try liquid clustering first and measure |
| You keep partitioning | Low-cardinality key (date, region), at least 1 GB per partition; ZORDER only inside partitions and never on the partition column |
| Existing Hive-style partitioned or Z-ordered table | Keep it working; convert with `REPLACE PARTITIONED BY WITH CLUSTER BY` (DBR 18.1+) when you can |
| Tables read by external Delta clients | Liquid clustering needs reader 3 / writer 7; old clients cannot read it |
| External (non-managed) tables | `CLUSTER BY` works, `CLUSTER BY AUTO` and predictive optimization do not — schedule `OPTIMIZE`/`VACUUM` yourself |

Predictive optimization never runs ZORDER and ignores Z-ordered files. Unpartitioned Delta tables already get ingestion-time clustering; for tables with heavy `UPDATE`/`MERGE`, cluster on a column that follows ingestion order (event timestamp).

## Quick Reference

| Input | Output | Notes |
|---|---|---|
| `CLUSTER BY (a, b)` | Manual keys | Up to 4; keys need column statistics (first 32 columns by default) |
| `CLUSTER BY AUTO` | Platform-selected keys | Keys change only when predicted savings beat clustering cost |
| `CLUSTER BY NONE` | Clustering off | Existing layout kept |
| `CREATE OR REPLACE` without `CLUSTER BY AUTO` | Auto clustering turned off | Repeat `CLUSTER BY AUTO` in the replace |
| `ALTER TABLE t SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = '30 days')` | Longer time travel | Set before enabling predictive optimization; floor is 7 days |

## Common Mistakes

### Wrong

```sql
CREATE TABLE main.sales.orders (...) PARTITIONED BY (order_date);   -- small/medium table
OPTIMIZE main.sales.orders ZORDER BY (customer_id);                 -- nightly job
-- Combining CLUSTER BY with PARTITIONED BY or ZORDER is an error
```

### Correct

```sql
CREATE TABLE main.sales.orders (...) CLUSTER BY AUTO;   -- UC managed + predictive optimization
-- External table or no predictive optimization: CLUSTER BY (cols) + scheduled OPTIMIZE
```

## Related

- [delta-lake](delta-lake.md)
- [unity-catalog-iceberg](unity-catalog-iceberg.md)
- [delta-operations pattern](../patterns/delta-operations.md)
- [migration-to-open-formats pattern](../patterns/migration-to-open-formats.md)
- [DBR version matrix](../../spark/reference/version-matrix.md)

## Sources

- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/tables/partitions
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- https://docs.databricks.com/aws/en/tables/managed
- https://docs.delta.io/latest/delta-clustering.html
