# Lakeflow Connect

> **Purpose**: Managed ingestion connectors and how they relate to Lakeflow pipelines, Auto Loader and DIY ingestion
> **Confidence**: Medium (connector release states vary per connector; check each connector page)
> **MCP Validated**: 2026-10-17

## Overview

Lakeflow Connect is the ingestion layer of Lakeflow. Fully-managed connectors create ingestion
pipelines that are governed by Unity Catalog, run on serverless compute, and are built on Lakeflow
pipelines. Databricks recommends starting at the most managed layer and dropping down (Lakeflow
pipelines with Auto Loader or Structured Streaming, then DIY code) only when a source is not covered.
Connectors are in various release states; the overview pages do not list GA/Preview per connector.

## The Concept

### Connector types

| Type | Examples named in the docs | Components |
|------|----------------------------|------------|
| Database (CDC) | MySQL, PostgreSQL, SQL Server, Oracle | Connection, ingestion gateway (own continuous job), staging storage, ingestion pipeline |
| SaaS | Salesforce, HubSpot, Jira, Workday | Connection, ingestion pipeline, destination tables |
| File | Google Drive, SharePoint | Connection, ingestion pipeline, destination tables |
| Streaming | Apache Kafka, RabbitMQ | Connection, ingestion pipeline (serverless) |
| Query-based | Databases without CDC | Connection or foreign catalog, ingestion pipeline |
| Direct write | Zerobus Ingest API | Application writes records to streaming tables |
| Community / custom | Sources without a managed connector | Open source or your own code |

### Ingestion layers

| Layer | Use when |
|-------|----------|
| Fully-managed connector | The source is supported; you want auth, CDC, retries and schema evolution handled |
| Lakeflow pipelines + Auto Loader / `read_files` / Kafka | Files in cloud storage or message buses, custom transforms |
| Structured Streaming / DIY | Features not yet in pipelines, or unsupported sources |

### Operating model

- Each schedule added to an ingestion pipeline creates a Lakeflow Job with the pipeline as a task;
  you can add more tasks to that job.
- Incremental: the first run loads all selected data, later runs use the source's change tracking
  (CDC, change tracking, or a cursor column) when available.
- Deployable with Declarative Automation Bundles: the `pipeline` resource takes an
  `ingestion_definition` (`connection_name`, `objects` with schema/table/report specs,
  `table_configuration` such as `primary_keys`, `sequence_by`). `ingestion_definition` cannot be
  combined with `libraries`, `schema`, `target` or `catalog`.
- Connections for API-auth connectors can be created programmatically
  (`databricks connections create --json ...`); browser-OAuth-only connectors cannot.

## Quick Reference

| Question | Answer |
|----------|--------|
| Files landing in a UC volume or external location | Auto Loader in a streaming table, often with a file arrival job trigger |
| Relational DB with CDC enabled | Database connector (gateway + ingestion pipeline) |
| Relational DB without CDC | Query-based connector |
| SaaS API | SaaS connector, if listed for that application |
| Cost tracking | `system.billing.usage` |

## Common Mistakes

### Wrong

```text
Build a custom JDBC polling notebook for a source that has a managed connector,
then hand-roll retries, schema drift handling and incremental cursors.
```

### Correct

```text
Check the connector list first. Use the managed connector when available; fall back to
Lakeflow pipelines (Auto Loader / Kafka) and only then to custom code.
```

## Related

- [Bronze ingestion](../patterns/bronze-ingestion.md)
- [Lakeflow Jobs reference](../reference/lakeflow-jobs.md)
- [Core concepts](core-concepts.md)

## Sources

- https://docs.databricks.com/aws/en/ingestion/overview
- https://docs.databricks.com/aws/en/ingestion/lakeflow-connect/
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/
