# Catalog Wars

> **Purpose**: Lakehouse catalog comparison — Unity Catalog (Databricks and OSS 0.7.0), Apache Polaris 1.8.0, Apache Gravitino 1.3.1, Nessie 0.109.0, foreign catalogs, decision matrix
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Catalogs hold table metadata pointers, access control and discovery for every engine. The market converged on the **Iceberg REST Catalog (IRC) spec** as the common protocol: Unity Catalog OSS 0.7.0 (2026-10-13) implements full IRC create/write/read/rename/drop plus a UC Delta REST API (since 0.5.0), Databricks Unity Catalog serves managed Delta and Iceberg tables over IRC, Polaris became an Apache top-level project on 2026-02-18, and Gravitino 1.3 added a Glue catalog, unified views and a federated IRC. Delta Lake moved commit coordination into the catalog (catalog-managed tables, Delta 4.0.1+), so the catalog is increasingly the source of truth for table state, not just a name registry.

## The Concept

```properties
# Gravitino Iceberg REST service as an auxiliary service of the main server (gravitino.conf).
# Standalone mode uses conf/gravitino-iceberg-rest-server.conf with the same keys.
gravitino.auxService.names = iceberg-rest
gravitino.iceberg-rest.classpath = iceberg-rest-server/libs, iceberg-rest-server/conf
gravitino.iceberg-rest.host = 0.0.0.0
gravitino.iceberg-rest.httpPort = 9001
# Backend: memory (default, dev only) | hive | jdbc | rest; each needs its own uri/warehouse keys
gravitino.iceberg-rest.catalog-backend = hive
gravitino.iceberg-rest.uri = thrift://hive-metastore:9083
gravitino.iceberg-rest.warehouse = s3://my-bucket/warehouse/
# Clients then use http://<host>:9001/iceberg/ as their REST catalog URI
```

```sql
-- Nessie Spark SQL extensions (NessieSparkSessionExtensions): git-like branches
CREATE BRANCH IF NOT EXISTS dev_feature IN nessie FROM main;
-- Writes go to the branch configured in spark.sql.catalog.nessie.ref
MERGE BRANCH dev_feature INTO main IN nessie;
DROP BRANCH IF EXISTS dev_feature IN nessie;
```

## Quick Reference

| Catalog | Version (2026-10-17) | Formats | Governance | Notes |
|---|---|---|---|---|
| Unity Catalog (Databricks) | Managed service | Delta, Iceberg (managed and foreign) | Privileges + ABAC (governed tags, row filter/column mask policies, GRANT policies; metastore-level and DENY policies in Beta) | IRC at `/api/2.1/unity-catalog/iceberg-rest`, credential vending, catalog commits, HMS/Glue federation |
| Unity Catalog OSS (LF AI & Data) | 0.7.0 | Delta (UC Delta API), Iceberg (IRC) | Privileges; no ABAC | Storage credentials/external locations on AWS since 0.4.0 |
| Apache Polaris (TLP) | 1.8.0 | Iceberg; generic tables (Delta, CSV, …) in beta | RBAC; OPA authorization (preview) | Docker images `apache/polaris`; Helm chart |
| Apache Gravitino (TLP) | 1.3.1 | Iceberg, Hive, JDBC, Hudi, Paimon; Lance REST (1.1+) | Role-based privileges | Iceberg REST service, Table Maintenance Service (1.2+), MCP server |
| Project Nessie | 0.109.0 | Iceberg | Basic | Git-like branches/tags/merges across tables |
| Foreign catalogs (AWS Glue, Hive metastore, Snowflake Horizon) | — | Iceberg | Owned by the source catalog | Readable from Databricks as foreign Iceberg tables (Lakehouse Federation) |

| Decision factor | Best option |
|---|---|
| Databricks workloads | Unity Catalog (managed tables), exposing tables to other engines over IRC |
| Engine-neutral Iceberg catalog | Polaris or Gravitino (REST spec) |
| Git-like data branching | Nessie |
| Federating many heterogeneous sources | Gravitino |
| Snowflake + open Iceberg | Snowflake Open Catalog (managed Polaris) or self-hosted Polaris |
| Existing Hive metastore on Databricks | HMS federation into Unity Catalog, then convert to managed tables |

## 2025–2026 Milestones

| Date | Event |
|---|---|
| 2025-07-09 | Polaris 1.0.0 (still incubating) |
| 2025-09-30 | Gravitino 1.0.0 |
| 2025-12-12 | Unity Catalog OSS 0.3.1 (Java client, credential renewal) |
| 2025-12-16 | Gravitino 1.1.0 (Lance REST service, generic lakehouse catalog) |
| 2026-01-16 | Polaris 1.3.0 |
| 2026-02-18 | Polaris graduates to Apache top-level project |
| 2026-02-26 | Delta Lake 4.1.0 (catalog-managed tables in preview) |
| 2026-05-21 | Databricks: managed Iceberg, foreign Iceberg and Iceberg v3 GA in Unity Catalog |
| 2026-06-18 | Delta Lake 4.3.0 (UC Delta REST API default) and Unity Catalog OSS 0.5.0 (UC Delta API) |
| 2026-06-29 | Gravitino 1.3.0 (Glue catalog, unified views, federated IRC) |
| 2026-09-28 / 09-29 | Polaris 1.8.0 / Iceberg 1.12.0 |
| 2026-10-13 | Unity Catalog OSS 0.7.0 (full Iceberg REST catalog) |

## Common Mistakes

### Wrong

```text
Keeping Hive Metastore as the long-term catalog: no REST protocol, no fine-grained policies,
no catalog-coordinated commits.
```

### Correct

```text
Databricks: federate the HMS into Unity Catalog, then convert tables to UC managed tables.
Elsewhere: move Iceberg tables to a REST catalog (Polaris, Gravitino, UC OSS); keep HMS read-only
during the migration window.
```

## Related

- [unity-catalog-iceberg](unity-catalog-iceberg.md)
- [iceberg-v3](iceberg-v3.md)
- [delta-lake](delta-lake.md)
- [catalog-setup pattern](../patterns/catalog-setup.md)

## Sources

- https://github.com/unitycatalog/unitycatalog/releases
- https://polaris.apache.org/downloads/
- https://polaris.apache.org/blog/2026/02/19/apache-polaris-graduates-to-top-level-project/
- https://polaris.apache.org/releases/1.7.0/managing-security/external-pdp/opa/
- https://polaris.apache.org/in-dev/unreleased/generic-table/
- https://github.com/apache/gravitino/releases
- https://gravitino.apache.org/docs/1.3.1/iceberg-rest-service
- https://github.com/projectnessie/nessie/releases
- https://projectnessie.org/nessie-latest/spark-sql/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/
- https://docs.databricks.com/aws/en/query-federation/hms-federation-concepts
- https://docs.databricks.com/aws/en/iceberg/
- https://docs.snowflake.com/en/user-guide/opencatalog/release-notes
