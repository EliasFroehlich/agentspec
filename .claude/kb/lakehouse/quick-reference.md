# Lakehouse Quick Reference

> **Purpose**: Fast lookup tables for table formats, catalogs and Databricks table defaults. For code, see linked files.
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Table Format Comparison

| Feature | Iceberg (spec v3, lib 1.12.0) | Delta Lake (OSS 4.4.1) | Hudi (1.2.1) | DuckLake (1.0) |
|---|---|---|---|---|
| Status | v1–v3 adopted; v4 in development | 4.x line; catalog-managed tables | Not covered in this KB | Production-ready since 2026-04-13 |
| Schema evolution | Full + partition evolution | Full + type widening (GA 4.0) | — | Full (SQL) |
| Layout | Hidden partitioning, sort orders | Liquid clustering (3.1+); Hive partitions legacy | — | Partitions incl. `bucket()`, sorted tables |
| Row-level deletes | v2 delete files; v3 deletion vectors | Deletion vectors (2.3+) | — | Delete files; Iceberg-compatible DVs |
| Semi-structured | `variant` (v3) | Variant (4.0+) | — | JSON, variant |
| Geospatial | `geometry`/`geography` (v3) | Kernel 4.2+ reads/writes geo types; not documented for Spark | — | Geometry |
| Interop | IRC standard | UniForm / Iceberg reads (IcebergCompatV2/V3) | — | Iceberg deep copy + metadata import |
| Next | v4: adaptive metadata tree, relative paths, no new equality deletes | Delta 5.0 announced together with Iceberg v4 convergence | — | Spec 1.1 |

## Catalog Comparison

| Catalog | Version | Formats | Governance |
|---|---|---|---|
| Unity Catalog (Databricks) | Managed | Delta, Iceberg (managed + foreign) | Privileges + ABAC; IRC endpoint; catalog commits |
| Unity Catalog OSS | 0.7.0 | Delta (UC Delta API), Iceberg (IRC) | Privileges |
| Apache Polaris (TLP 2026-02-18) | 1.8.0 | Iceberg; generic tables (beta) | RBAC; OPA (preview) |
| Apache Gravitino | 1.3.1 | Iceberg, Hive, JDBC, Hudi, Paimon, Lance | Role-based privileges |
| Project Nessie | 0.109.0 | Iceberg | Git-like branches |

## Databricks Defaults (UC managed tables)

| Topic | Default / recommendation | Minimum |
|---|---|---|
| Table type | UC managed (Delta, or `USING iceberg`) | — |
| Layout | Liquid clustering; `CLUSTER BY AUTO` | DBR 15.4 LTS+ |
| Maintenance | Predictive optimization: OPTIMIZE, VACUUM, ANALYZE (no ZORDER) | Accounts created ≥ 2024-11-11 |
| Partitioning | Only tables ≥ 100 TB, and only if it beats liquid clustering | — |
| Convert partitioned table | `REPLACE PARTITIONED BY WITH CLUSTER BY [AUTO]` | DBR 18.1+ (not 17.3 LTS) |
| Iceberg v3 features | Deletion vectors, VARIANT, row lineage, geo types | DBR 18 LTS+ |
| Catalog commits | `delta.feature.catalogManaged = supported` | DBR 16.4+ (enable existing: 18.0+) |
| External engines | IRC `/api/2.1/unity-catalog/iceberg-rest` + `EXTERNAL USE SCHEMA` | Iceberg client 1.9.2+ |

## When to Use What

| Scenario | Recommended |
|---|---|
| Databricks platform | UC managed Delta tables, `CLUSTER BY AUTO`, predictive optimization |
| Databricks + Iceberg engines must read | Delta with Iceberg reads, or managed Iceberg (engines can also write) |
| Multi-engine, no single vendor | Iceberg v3 + REST catalog (Polaris, Gravitino, UC OSS) |
| Git-like data versioning | Iceberg + Nessie |
| Local / small-team lakehouse, SQL catalog | DuckLake (DuckDB, SQLite or PostgreSQL catalog) |
| Low-latency OLTP for apps next to the lakehouse (Databricks) | Lakebase (managed Postgres; sync UC tables into it; Postgres changes to Delta in Public Preview) |
| Snowflake + open tables | Iceberg + Snowflake Open Catalog or Polaris |
| Read external Glue/HMS/Snowflake Horizon Iceberg from Databricks | Foreign Iceberg tables (read-only) via Lakehouse Federation |

## Common Pitfalls

| Don't | Do |
|---|---|
| Schedule nightly OPTIMIZE/ZORDER on UC managed tables | Enable predictive optimization; schedule only external/legacy tables |
| `PARTITIONED BY` date on small/medium tables | `CLUSTER BY AUTO` or `CLUSTER BY (cols)` |
| Deletion vectors + IcebergCompatV2 | IcebergCompatV3 (DBR 18 LTS+) or REORG to purge DVs |
| `'delta.catalogManaged' = 'true'` | `'delta.feature.catalogManaged' = 'supported'` |
| Identity columns on tables with concurrent writers | Hash keys or a single-writer load step |
| `VACUUM ... RETAIN 0 HOURS` | Keep ≥ 7 days (`delta.deletedFileRetentionDuration`) |
| HMS as long-term catalog | HMS federation into UC (Databricks) or a REST catalog |
| DuckDB-file DuckLake catalog for many writers | PostgreSQL catalog |

## Related Documentation

| Topic | Path |
|---|---|
| Liquid clustering | `concepts/liquid-clustering.md` |
| UC managed / Iceberg / catalog commits | `concepts/unity-catalog-iceberg.md` |
| Delta features by version | `concepts/delta-lake.md` |
| Full Index | `index.md` |

## Sources

- https://iceberg.apache.org/spec/ · https://iceberg.apache.org/releases/
- https://github.com/delta-io/delta/releases · https://docs.delta.io/latest/versioning.html
- https://github.com/apache/hudi/releases · https://ducklake.select/2026/04/13/ducklake-10/
- https://github.com/unitycatalog/unitycatalog/releases · https://polaris.apache.org/downloads/
- https://github.com/apache/gravitino/releases · https://github.com/projectnessie/nessie/releases
- https://docs.databricks.com/aws/en/tables/clustering · https://docs.databricks.com/aws/en/tables/partitions
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization · https://docs.databricks.com/aws/en/tables/managed
- https://docs.databricks.com/aws/en/iceberg/ · https://docs.databricks.com/aws/en/external-access/iceberg
- https://docs.databricks.com/aws/en/tables/features/catalog-commits · https://docs.databricks.com/aws/en/oltp/
- https://www.databricks.com/blog/unity-catalog-and-next-era-apache-icebergtm
