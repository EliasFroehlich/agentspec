# Medallion Architecture Quick Reference

> **Purpose**: Fast lookup tables for Bronze/Silver/Gold design. For code, see linked files.
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

Medallion is a recommended practice, not a requirement; layers are defined by purpose and consumer.

## Layer Comparison

| Property | Bronze | Silver | Gold |
|---|---|---|---|
| Purpose | Raw ingestion | Cleaning, validation, dedup | Dimensional modeling, aggregation |
| Types | String / VARIANT / binary + metadata | Enforced, cast | Business-typed, documented |
| Writes | Append (incremental) | MERGE / `AUTO CDC` | MV, MERGE, `INSERT ... REPLACE WHERE`, plain overwrite |
| Reads from previous layer | — | Streaming read of Bronze | Batch or MV over Silver |
| Consumers | Data engineers, audit | Engineers, analysts, data scientists | BI, ML, apps, executives |
| Layout | `CLUSTER BY AUTO` / `(_ingested_at)` | `CLUSTER BY AUTO` / business keys | `CLUSTER BY AUTO` |

## Naming Convention (one common option)

| Layer | Schema | Table |
|---|---|---|
| Bronze | `bronze_{domain}` (or catalog `<env>`, schema `bronze`) | `raw_{source}_{entity}` |
| Silver | `silver_{domain}` | `{entity}` or `cleansed_{entity}` |
| Gold | `gold_{domain}` | `dim_{entity}`, `fact_{entity}`, `agg_{metric}` |

## Decision Matrix

| Use Case | Choose |
|---|---|
| Files landing in a volume | Auto Loader or `read_files` streaming table → Bronze |
| Kafka events | Kafka source → Bronze (VARIANT payload) |
| Dedup + cleansing | Streaming read of Bronze + MERGE per micro-batch |
| CDC feed with history | `AUTO CDC ... STORED AS SCD TYPE 2` (Lakeflow pipelines) |
| Business KPIs | Gold materialized views; governed metrics in UC metric views |
| Quality enforcement | Expectations, NOT NULL/CHECK constraints, quarantine — per layer |
| Source schema changes | Evolve automatically in Bronze; explicit DDL in Silver/Gold |
| ML features | Feature Engineering in UC tables from Silver/Gold |

## Common Pitfalls

| Don't | Do |
|---|---|
| `input_file_name()` | `_metadata.file_path` (DBR 10.5+); the function is unsupported on DBR 17.3 LTS+ and UC |
| Write Silver straight from ingestion | Bronze first, then streaming reads |
| `overwriteSchema=true` on Silver/Gold | DDL (`ALTER TABLE ADD COLUMN`, type widening), then write |
| `ADD COLUMN ... DEFAULT` on an existing Delta table | `ADD COLUMN`, then `ALTER COLUMN ... SET DEFAULT` |
| `PARTITIONED BY` date + nightly ZORDER | `CLUSTER BY AUTO` + predictive optimization |
| `/mnt/...` paths | `/Volumes/...` or external-location URIs |
| Identity columns on concurrently written tables | They disable concurrent transactions; use hash keys |
| Gold without refresh metadata | `_computed_at`, `DESCRIBE HISTORY`, or MV refresh history |

## Essential Commands (Delta on Databricks / Iceberg)

| Operation | Delta | Iceberg |
|---|---|---|
| Upsert | `MERGE [WITH SCHEMA EVOLUTION] INTO` (15.2+) | `MERGE INTO` |
| Add column | `ALTER TABLE t ADD COLUMN c STRING` | `ALTER TABLE t ADD COLUMN c string` |
| Rename / drop | Column mapping (`id` new, `name` existing) + `RENAME/DROP COLUMN` | Native |
| Widen type | `delta.enableTypeWidening` + `ALTER COLUMN c TYPE BIGINT` (DBR 15.4 LTS+) | `ALTER COLUMN c TYPE bigint` |
| Maintenance | Predictive optimization; else `OPTIMIZE`, `VACUUM` | `rewrite_data_files`, `expire_snapshots` |
| Time travel | `VERSION AS OF 5` | `VERSION AS OF <snapshot-id>` |
| Semi-structured | `VARIANT` (DBR 15.4+), `col:path::STRING` | `variant` (spec v3) |

## AI-Era Extensions

| Extension | Source | Databricks-native | Alternatives |
|---|---|---|---|
| Feature layer | Silver/Gold | Feature Engineering in Unity Catalog | Feast, Tecton |
| Vector layer | Gold | AI Search (formerly Vector Search) | Qdrant, Pinecone, Weaviate |
| Semantic layer | Gold | UC metric views ([../metric-views/index.md](../metric-views/index.md)) | dbt Semantic Layer, Cube, AtScale |

## Related Documentation

| Topic | Path |
|---|---|
| Layer Transitions | `patterns/layer-transitions.md` |
| Quality Gates | `patterns/data-quality-gates.md` |
| Full Index | `index.md` |

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion
- https://docs.databricks.com/aws/en/sql/language-manual/functions/input_file_name
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table-manage-column
- https://docs.databricks.com/aws/en/tables/clustering · https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/tables/features/column-mapping · https://docs.databricks.com/aws/en/tables/features/type-widening
- https://docs.databricks.com/aws/en/uc-semantics/metric-views · https://docs.databricks.com/aws/en/release-notes/product/
