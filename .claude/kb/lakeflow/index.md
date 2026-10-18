# Databricks Lakeflow Knowledge Base

> **Purpose**: Lakeflow Spark Declarative Pipelines (Lakeflow pipelines, formerly Delta Live Tables), Lakeflow Jobs, Lakeflow Connect, and their deployment with Declarative Automation Bundles (DABs, formerly Databricks Asset Bundles)
> **Confidence**: High
> **MCP Validated**: 2026-10-17
> **Coverage**: Pipelines release notes through September 2026 (channels CURRENT = DBR 18, PREVIEW = DBR 19); Databricks CLI v1.21.0; Apache Spark 4.1/4.2 Spark Declarative Pipelines

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/core-concepts.md](concepts/core-concepts.md) | Datasets, flows, sinks, pipelines, channels, recent changes |
| [concepts/python-module-migration.md](concepts/python-module-migration.md) | `dlt` to `pyspark.pipelines` mapping; open-source SDP |
| [concepts/getting-started.md](concepts/getting-started.md) | Pipelines Editor, folder layout, first CDC pipeline |
| [concepts/cdc-fundamentals.md](concepts/cdc-fundamentals.md) | AUTO CDC / AUTO CDC FROM SNAPSHOT, SCD types |
| [concepts/expectations-model.md](concepts/expectations-model.md) | Expectation actions, scope, limits |
| [concepts/deployment-model.md](concepts/deployment-model.md) | Bundles, targets, deployment modes, CLI engine |
| [concepts/lakeflow-connect.md](concepts/lakeflow-connect.md) | Managed connectors and ingestion layers |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/python-decorators.md](patterns/python-decorators.md) | `@dp.table`, `@dp.materialized_view`, `@dp.temporary_view`, parameters, loops |
| [patterns/python-streaming.md](patterns/python-streaming.md) | Auto Loader, Kafka, batch vs streaming, medallion in Python |
| [patterns/sql-tables.md](patterns/sql-tables.md) | `CREATE OR REFRESH`, `read_files`, `CLUSTER BY`, `PRIVATE`, targets, parameters |
| [patterns/sql-streaming.md](patterns/sql-streaming.md) | `CREATE FLOW` (AUTO CDC, INSERT ONCE), joins, windows, watermarks |
| [patterns/cdc-apply-changes.md](patterns/cdc-apply-changes.md) | AUTO CDC (formerly APPLY CHANGES): SCD1/2, snapshots, partial updates |
| [patterns/cdc-example.md](patterns/cdc-example.md) | End-to-end CDC pipeline in Python and SQL |
| [patterns/expectations.md](patterns/expectations.md) | Warn / drop / fail expectations |
| [patterns/expectations-advanced.md](patterns/expectations-advanced.md) | Grouped rules, referential checks, metrics from the event log |
| [patterns/flows-and-sinks.md](patterns/flows-and-sinks.md) | Append flows, backfills, sinks, ForEachBatch |
| [patterns/bronze-ingestion.md](patterns/bronze-ingestion.md) | Raw ingestion, `_metadata` columns, volumes |
| [patterns/silver-cleansing.md](patterns/silver-cleansing.md) | Typing, PII, watermark dedup, keep-latest |
| [patterns/gold-aggregation.md](patterns/gold-aggregation.md) | Aggregate MVs, 360 views, time series, incremental tips |
| [patterns/dabs-deployment.md](patterns/dabs-deployment.md) | Pipeline resources in bundles, commands, CLI 1.20 impact |
| [patterns/jobs-bundle-yaml.md](patterns/jobs-bundle-yaml.md) | Lakeflow Job in bundle YAML: tasks, branching, loops, triggers, repair |

### Reference (no line limit)

| File | Purpose |
|------|---------|
| [reference/lakeflow-jobs.md](reference/lakeflow-jobs.md) | Jobs task types, parameters, dynamic value references, triggers, compute, repair |
| [reference/bundles-cli.md](reference/bundles-cli.md) | Bundle mappings, variables, modes, CLI versions, direct engine, commands, CI auth |
| [reference/pipeline-configuration.md](reference/pipeline-configuration.md) | Pipeline settings, editions, run-as, modes, properties |
| [reference/serverless-pipelines.md](reference/serverless-pipelines.md) | Serverless requirements, performance modes, cost attribution |
| [reference/materialized-views.md](reference/materialized-views.md) | Incremental refresh support, refresh policy, access |
| [reference/stateful-processing.md](reference/stateful-processing.md) | Watermarks, joins, windows, dedup, state store |
| [reference/parameters.md](reference/parameters.md) | Pipeline parameters vs configuration, job pushdown |
| [reference/unity-catalog.md](reference/unity-catalog.md) | Publishing mode, permissions, governance, inactive tables |
| [reference/limitations.md](reference/limitations.md) | Quotas and restrictions with workarounds |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Streaming table** | Processes each record of an append-only source once |
| **Materialized view** | Cached query result, refreshed incrementally on serverless when possible |
| **Temporary view** | Pipeline-scoped, not published |
| **Flow** | Append (default, many per target), AUTO CDC, Update (Preview) |
| **Sink** | Delta / Kafka / Event Hubs / custom Python target, Python only |
| **Expectation** | Warn, drop or fail on a SQL boolean rule |
| **AUTO CDC** | SCD Type 1/2 (bitemporal Beta) from change feeds or snapshots; replaces APPLY CHANGES |
| **Default publishing mode** | Fully qualified names across catalogs/schemas; `LIVE` ignored |
| **Lakeflow Job** | DAG of tasks with triggers, parameters, retries and notifications |
| **DABs** | Bundles of jobs/pipelines in YAML; direct deployment engine only since CLI 1.20.0 |

## Recent Release Highlights

| Release | Key feature |
|---------|-------------|
| 2025.04 (Jan 2025) | Default publishing mode: multi-catalog/schema; `LIVE` no longer required |
| 2025.29 (Jul 2025) | Move tables between pipelines GA; ALTER on pipeline MVs/STs |
| 2025.30 (Jul 2025) | AUTO CDC `name` (multiple flows per target), `once`, `CREATE FLOW` SQL |
| Jan 2026 | Queued updates; option to drop inactive tables; UC-stored expectations |
| Feb 2026 | Type widening; SCD Type 1 materialization with AUTO CDC |
| Mar 2026 | Retain UC tables on pipeline delete; SQL in `foreachBatch` |
| May 2026 | Bitemporal AUTO CDC; pipeline-declared Python dependencies; `SET OWNER TO` service principal |
| Jun 2026 | Liquid clustering on AUTO CDC targets; expectations on standalone MVs |
| Aug 2026 | `CREATE TABLE ... FLOW` (Beta); event log system table (Beta); metadata-only column ALTER (Beta) |
| Sep 2026 | Run-as group; maintenance windows; Rewind (Beta); Python unit tests (Beta); native expression evaluation for AUTO CDC |
| CLI v1.20.0 (Oct 2026) | Terraform deployment engine removed from bundles |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/core-concepts.md -> concepts/python-module-migration.md -> concepts/getting-started.md |
| **Intermediate** | patterns/python-decorators.md -> patterns/expectations.md -> patterns/cdc-apply-changes.md -> patterns/dabs-deployment.md |
| **Advanced** | reference/materialized-views.md -> reference/stateful-processing.md -> reference/lakeflow-jobs.md -> reference/bundles-cli.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| lakeflow-architect | concepts/, patterns/, reference/lakeflow-jobs.md | Pipeline and orchestration design |
| lakeflow-expert | All files, reference/limitations.md | Troubleshooting, CDC, operations |
| lakeflow-pipeline-builder | patterns/bronze-*, silver-*, gold-*, dabs-*, jobs-bundle-yaml.md | Implementation |
| lakeflow-specialist | quick-reference.md, patterns/python-decorators.md, patterns/expectations.md | Focused pipeline tasks |
| ci-cd-specialist | patterns/dabs-deployment.md, reference/bundles-cli.md | Bundle CI/CD |

Complementary official skills: `databricks:databricks-pipelines`, `databricks:databricks-jobs`,
`databricks:databricks-dabs` (and `databricks:databricks-docs` for anything else).

## Sources

- https://docs.databricks.com/aws/en/ldp/
- https://docs.databricks.com/aws/en/ldp/concepts/where-is-dlt
- https://docs.databricks.com/aws/en/release-notes/dlt/
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/04/
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/29/
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/30/
- https://docs.databricks.com/aws/en/jobs/
- https://docs.databricks.com/aws/en/ingestion/overview
- https://docs.databricks.com/aws/en/dev-tools/bundles/
- https://github.com/databricks/cli/releases/tag/v1.20.0
- https://github.com/databricks/cli/releases/tag/v1.21.0
