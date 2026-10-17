# Lakehouse Knowledge Base

> **Purpose**: Open table formats and catalogs — Iceberg (spec v3 adopted, v4 in development; library 1.12.0), Delta Lake 4.4.1, DuckLake 1.0, Unity Catalog (Databricks; OSS 0.7.0), Gravitino 1.3.1, Polaris 1.8.0 (Apache TLP), Nessie 0.109.0 — plus Databricks table layout (liquid clustering, `CLUSTER BY AUTO`, predictive optimization) and UC managed Iceberg
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/delta-lake.md](concepts/delta-lake.md) | Delta 4.x release line, feature × version table (OSS vs DBR), identity-column caveats, catalog-managed tables |
| [concepts/liquid-clustering.md](concepts/liquid-clustering.md) | `CLUSTER BY` / `CLUSTER BY AUTO`, predictive optimization, when partitioning or ZORDER still apply |
| [concepts/unity-catalog-iceberg.md](concepts/unity-catalog-iceberg.md) | UC managed/foreign Iceberg, Iceberg v3 on Databricks, Iceberg REST access, Iceberg reads (UniForm), catalog commits |
| [concepts/iceberg-v3.md](concepts/iceberg-v3.md) | Iceberg spec v2/v3/v4, hidden partitioning, deletion vectors, REST catalog |
| [concepts/catalog-wars.md](concepts/catalog-wars.md) | Unity Catalog vs Polaris vs Gravitino vs Nessie, 2025–2026 milestones |
| [concepts/ducklake.md](concepts/ducklake.md) | DuckLake 1.0: SQL-database catalog, inlining, Iceberg interop |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/delta-operations.md](patterns/delta-operations.md) | MERGE (with schema evolution), INSERT REPLACE, CDF, OPTIMIZE/VACUUM vs predictive optimization |
| [patterns/iceberg-operations.md](patterns/iceberg-operations.md) | DDL, MERGE, time travel, maintenance procedures; Databricks managed-Iceberg differences |
| [patterns/catalog-setup.md](patterns/catalog-setup.md) | UC namespaces, grants, ABAC policies, IRC client config; Gravitino, Nessie, Polaris |
| [patterns/migration-to-open-formats.md](patterns/migration-to-open-formats.md) | Hive→Iceberg, Parquet→Delta, external→UC managed, partitioned→liquid clustering, Delta→Iceberg readers |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Catalog as source of truth** | IRC is the common protocol; catalog-managed tables / catalog commits move commit coordination into the catalog |
| **Liquid clustering by default** | Databricks recommends it for all new tables; partitioning only for very large tables after measuring |
| **Predictive optimization** | OPTIMIZE, VACUUM and ANALYZE run automatically on UC managed tables; ZORDER is never run |
| **Version gates** | Many Databricks features need DBR 18 LTS (e.g., Iceberg v3, `REPLACE PARTITIONED BY`) and are absent on DBR 17.3 LTS — see [version matrix](../spark/reference/version-matrix.md) |

## Related Domains

| Domain | Why |
|---|---|
| [../medallion/index.md](../medallion/index.md) | Layer design that uses these tables |
| [../data-modeling/index.md](../data-modeling/index.md) | Logical models (star schema, SCD) on top of the formats |
| [../metric-views/index.md](../metric-views/index.md) | Unity Catalog metric views (semantic layer over Gold tables) |

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| lakehouse-architect | All files | Format selection, Databricks table layout, catalog governance, migrations |
| data-platform-engineer | concepts/unity-catalog-iceberg.md, patterns/catalog-setup.md | Platform setup and external-engine access |
| medallion-architect | concepts/liquid-clustering.md | Layout per layer |

## Sources

- https://iceberg.apache.org/releases/ · https://github.com/delta-io/delta/releases · https://ducklake.select/2026/04/13/ducklake-10/
- https://github.com/unitycatalog/unitycatalog/releases · https://github.com/apache/gravitino/releases · https://polaris.apache.org/downloads/ · https://github.com/projectnessie/nessie/releases
- https://docs.databricks.com/aws/en/tables/clustering · https://docs.databricks.com/aws/en/optimizations/predictive-optimization · https://docs.databricks.com/aws/en/iceberg/
