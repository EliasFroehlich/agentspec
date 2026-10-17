# Pipeline Configuration

> **Purpose**: Pipeline settings (JSON/YAML spec), editions, compute, run-as identity, modes, channels, source code, dependencies, notifications and properties
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Defaults for a new pipeline

Created from New > ETL pipeline (or a workspace/Git folder, or as a bundle project): Unity Catalog,
`current` channel, serverless compute, default publishing mode, source code in a `transformations`
folder. Pipelines created before 2025-02-05 may use the legacy publishing mode and `LIVE` schema.

## 2. Settings reference

| Key | Type | Notes |
|-----|------|-------|
| `name` | string | Display name |
| `catalog` | string | Default catalog; setting it enables Unity Catalog |
| `schema` | string | Default schema (replaces legacy `target`) |
| `target` | string | Legacy; use `schema` |
| `storage` | string | Legacy Hive metastore storage location |
| `serverless` | bool | Serverless compute (needs `ADVANCED`) |
| `clusters` | array | Classic compute only; mutually exclusive with serverless |
| `photon` | bool | Classic only, default `false`; ignored with serverless |
| `edition` | `CORE` / `PRO` / `ADVANCED` | Default `ADVANCED` |
| `channel` | `current` / `preview` | Default `current` |
| `continuous` | bool | Default `false` (triggered) |
| `development` | bool | Default `false`; bundles set it through `mode` |
| `libraries` | array | `glob` (`include` path or folder `/**`), `file`, `notebook`; `whl` deprecated; JARs unsupported in UC |
| `root_path` | string | Root folder; added to `sys.path` for Python imports |
| `environment` | object | `dependencies` (pip lines), `environment_version` (Public Preview) |
| `configuration` | map | Spark conf + values for `spark.conf.get()` / `${key}` |
| `parameters` | map | Beta; SQL named parameters (`:key`) |
| `notifications` | array | `email_recipients` + `alerts`: `on-update-success`, `on-update-failure`, `on-update-fatal-failure`, `on-flow-failure` |
| `event_log` | object | `{catalog, schema, name}` to publish the event log |
| `run_as` | object | `user_name`, `service_principal_name` or `group_name` |
| `tags` | map | Up to 25; not billing tags |
| `budget_policy_id`, `usage_policy_id` | string | Serverless cost attribution |
| `restart_window` | object | Window in which the pipeline may be restarted |

## 3. Editions

| Edition | Includes |
|---------|----------|
| `CORE` | Streaming ingest |
| `PRO` | Core + CDC (`AUTO CDC`) |
| `ADVANCED` | Pro + expectations; required for serverless |

Using a feature outside the edition (for example expectations on `CORE`) fails with an explanatory
error.

## 4. Compute

- Serverless (recommended): no compute options to tune. See [serverless-pipelines.md](serverless-pipelines.md).
- Classic: Databricks recommends Enhanced autoscaling; options include cluster policy, fixed size or
  legacy autoscaling, min/max workers, Photon toggle, instance profile, cluster tags,
  worker/driver types. Cluster attributes set by the system cannot be overridden: `cluster_name`,
  `data_security_mode`/`access_mode`, `spark_version`, `autotermination_minutes`, `runtime_engine`.
- UC pipelines run on standard access mode compute.

## 5. Run-as identity

Run-as changes who the pipeline runs as and who owns its published tables. Service principals are
the documented best practice; an account group assigned to the workspace is also allowed (Sep 2026),
but not workspace-local groups, `admins`, `users` or `account users`.

| Who | Needs |
|-----|-------|
| User making the change | `CAN_MANAGE` on the pipeline; `CAN_USE` on the service principal, or membership/Assume on the group |
| Run-as identity | Workspace access, cluster policy use and compute creation (classic), read on source files, `USE CATALOG`, `USE SCHEMA` + `CREATE TABLE` (and `CREATE MATERIALIZED VIEW`), `MODIFY` on existing tables, `CREATE SCHEMA` if it creates schemas |

Bundles: `run_as: { service_principal_name: ... }` or `run_as: { group_name: ... }` on the pipeline.

## 6. Pipeline mode

| Mode | Behavior | Use |
|------|----------|-----|
| Triggered (default) | Refreshes with data available at start, then stops | Scheduled batch; most workloads |
| Continuous | Keeps processing as data arrives; compute always on | Only with a proven low-latency requirement |

For production continuous processing Databricks recommends a continuous **job** wrapping a
triggered pipeline; the job's schedule decides the execution mode. Changing settings applies to the
next update; continuous pipelines restart automatically (can interrupt an active update).

## 7. Source code

- Workspace files (`.py`, `.sql`) in the pipeline root folder, by default under `transformations/`.
  Notebooks from the legacy notebook editing experience are still supported (`libraries.notebook`).
- Python modules shared across files: put them under the root folder (for example `utilities/`) or
  a Git folder and import them.
- Python dependencies: `environment.dependencies` instead of `%pip install`.

## 8. Parameters and configuration

```json
{
  "configuration": { "landing_path": "/Volumes/main/landing/files", "pipelines.trigger.interval": "1 hour" },
  "parameters": { "start_date": "2026-01-01" }
}
```

Python reads `spark.conf.get("landing_path")`; SQL reads `${landing_path}` or `:start_date`. Details:
[parameters.md](parameters.md).

## 9. Notable properties

| Property | Default | Meaning |
|----------|---------|---------|
| `pipelines.trigger.interval` | 5 s streaming; 1 min complete (Delta sources); 10 min complete (non-Delta) | Set per table where possible |
| `pipelines.maxFlowRetryAttempts` | 2 retries | Retries of a single flow on retryable errors |
| `pipelines.numUpdateRetryAttempts` | 5 triggered / unlimited continuous | Retries of the whole update |
| `pipelines.reset.allowed` (table) | `true` | Allow full refresh of the table |
| `pipelines.autoOptimize.managed` (table) | `true` | Scheduled optimization (not used under predictive optimization) |
| `pipelines.autoOptimize.zOrderCols` (table) | none | Prefer liquid clustering |
| `pipelines.externalMetadata.enabled` | - | External data access for MVs/STs (DBR 17.3+) |
| `pipelines.dropInactiveTables` | - | `true` removes inactive tables from UC on the next update |

## 10. Example (classic compute, when serverless is not an option)

```json
{
  "name": "sales_pipeline_classic",
  "catalog": "main",
  "schema": "sales",
  "edition": "ADVANCED",
  "channel": "CURRENT",
  "photon": true,
  "clusters": [
    { "label": "default",
      "autoscale": { "min_workers": 1, "max_workers": 5, "mode": "ENHANCED" },
      "node_type_id": "<node-type-id>" }
  ],
  "libraries": [{ "glob": { "include": "/Workspace/Users/<user>/sales_pipeline/transformations/**" } }],
  "notifications": [{ "email_recipients": ["data-team@example.com"], "alerts": ["on-update-failure"] }]
}
```

## Sources

- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/serverless
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/multi-file-editor
- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/external-access/external-for-pipelines
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
