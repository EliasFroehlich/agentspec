# Serverless Lakeflow Pipelines

> **Purpose**: Requirements, configuration, performance modes, cost attribution and limits of serverless pipelines
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Position

Serverless is the default compute for new pipelines and the Databricks recommendation for new
development. Some workloads still need classic compute or the legacy Hive metastore. Cluster
creation permission is not required; by default all workspace users can use serverless pipelines.

Marketing note: a Databricks blog post (July 2025) claims large TCO reductions for serverless
pipelines; the product documentation states only that standard performance mode consumes fewer DBUs
than performance-optimized mode.

## 2. Requirements

| Requirement | Detail |
|-------------|--------|
| Unity Catalog | Always; serverless pipelines publish to UC |
| Product edition | `ADVANCED` (the default edition) |
| Terms | Serverless terms of use accepted |
| Region | Serverless-enabled region |
| Networking | AWS PrivateLink: contact your Databricks account team |

## 3. Configuration

```json
{
  "name": "sales_pipeline",
  "catalog": "main",
  "schema": "sales",
  "serverless": true,
  "channel": "CURRENT",
  "continuous": false,
  "root_path": "/Workspace/Users/<user>/sales_pipeline",
  "libraries": [{ "glob": { "include": "/Workspace/Users/<user>/sales_pipeline/transformations/**" } }],
  "configuration": { "landing_path": "/Volumes/main/landing/files" },
  "environment": { "dependencies": ["requests==2.32.3"] },
  "notifications": [
    { "email_recipients": ["data-team@example.com"],
      "alerts": ["on-update-failure", "on-update-fatal-failure", "on-flow-failure"] }
  ]
}
```

- Do not add a `clusters` block (error). `configuration` keeps working for Spark conf and
  parameters; `parameters` (Beta) holds SQL named parameters.
- Converting a UC pipeline to serverless (Settings > Compute > Serverless) removes its compute
  settings; switching back requires reconfiguring them.
- Python dependencies go in `environment.dependencies`; `dbutils.library.restartPython()` is not
  supported and serverless pipelines do not support init scripts.

## 4. Performance modes

| Mode | How to select | Startup | Cost |
|------|---------------|---------|------|
| Standard (default) | Clear "Performance optimized" in the pipeline scheduler | Typically 4-6 minutes | Fewer DBUs, same SKU |
| Performance optimized | Check "Performance optimized" in the scheduler | Faster | More DBUs, same SKU |

The setting applies to triggered pipelines. For a continuous pipeline in standard mode, run it from
a continuous job and clear "Performance optimized"; in a bundle, set `performance_target: STANDARD`
and `continuous.pause_status: UNPAUSED` on that job (the pipeline's built-in continuous mode does not
support standard mode).

## 5. Built-in behaviors

- Incremental refresh of materialized views (only on serverless).
- Stream pipelining: micro-batches run concurrently instead of sequentially (on by default).
- Vertical autoscaling on top of enhanced autoscaling (CPU-based since Mar 2026); state store is
  managed automatically for stateful streaming.
- Modes supported: triggered, continuous and real-time.

## 6. Cost attribution and monitoring

- Serverless usage policies (Public Preview) tag usage for billing attribution; select the policy
  in the pipeline settings (API/bundle fields `usage_policy_id`; `budget_policy_id` also exists in
  the pipeline spec).
- DBU usage: query the `system.billing.usage` system table.
- Update and flow events: event log (`event_log(<pipeline-id>)` or a published `event_log` table) or
  the Beta system table `system.lakeflow_pipeline_events_preview.pipeline_events`.

## 7. Runtime channels

| Channel | Runtime (2026-10-17) |
|---------|----------------------|
| `CURRENT` (default) | Databricks Runtime 18 |
| `PREVIEW` | Databricks Runtime 19 |

Upgrades roll out region by region; check the pipeline's runtime information when it matters.

## 8. When to choose classic compute

- A specific instance type, init script or dedicated access pattern is required.
- The region does not offer serverless.
- Hive metastore pipelines (serverless always uses Unity Catalog).

## 9. Optimization checklist

| Do | Avoid |
|----|-------|
| Triggered mode on a schedule unless low latency is proven | Continuous mode by default |
| Streaming tables for append-only ingestion | Full refreshes when incremental works |
| `CLUSTER BY AUTO` / `cluster_by_auto=True` | Z-order and manual partitioning |
| `refresh_policy="incremental_strict"` where full recompute is unaffordable | Float sums in MVs (cast to DECIMAL) |
| Watch DBUs in `system.billing.usage` | Performance-optimized mode without a latency need |

## Sources

- https://docs.databricks.com/aws/en/ldp/serverless
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/stateful-processing
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/monitor-event-logs
- https://docs.databricks.com/aws/en/jobs/tasks/pipeline
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/release-notes/dlt/
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://www.databricks.com/blog/whats-new-lakeflow-declarative-pipelines-july-2025
