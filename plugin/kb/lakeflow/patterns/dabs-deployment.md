# DABs Deployment Patterns

> **Purpose**: Deploy Lakeflow pipelines (and the jobs that schedule them) with Declarative Automation Bundles (DABs, formerly Databricks Asset Bundles)
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Promoting the same pipeline code to dev, staging and prod workspaces or catalogs
- Putting pipeline settings, schedules and permissions under source control and CI/CD
- Adopting an existing pipeline into a bundle (`bundle deployment bind`)

## Implementation

```text
sales_etl/
├── databricks.yml
├── resources/
│   ├── sales_pipeline.pipeline.yml
│   └── sales_job.job.yml
└── src/sales_pipeline/
    ├── transformations/   # .py / .sql pipeline sources
    └── utilities/         # importable Python modules
```

```yaml
# databricks.yml
bundle:
  name: sales_etl
  databricks_cli_version: ">= 1.20.0"

include:
  - resources/*.yml

variables:
  catalog:
    description: Target Unity Catalog catalog
  landing_path:
    description: Volume path with raw files

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<dev-workspace-host>
    variables:
      catalog: dev_sales
      landing_path: /Volumes/dev_sales/landing/files
  prod:
    mode: production
    workspace:
      host: https://<prod-workspace-host>
    git:
      branch: main
    variables:
      catalog: prod_sales
      landing_path: /Volumes/prod_sales/landing/files
    run_as:
      service_principal_name: "<service-principal-application-id>"
```

```yaml
# resources/sales_pipeline.pipeline.yml
resources:
  pipelines:
    sales_pipeline:
      name: sales_pipeline
      catalog: ${var.catalog}
      schema: sales
      serverless: true
      channel: CURRENT
      root_path: ../src/sales_pipeline
      libraries:
        - glob:
            include: ../src/sales_pipeline/transformations/**
      configuration:            # read with spark.conf.get / ${key}
        landing_path: ${var.landing_path}
      environment:
        dependencies:
          - "requests==2.32.3"
      notifications:
        - email_recipients: ["data-team@example.com"]
          alerts: [on-update-failure, on-update-fatal-failure, on-flow-failure]
      permissions:
        - level: CAN_VIEW
          group_name: data-consumers
        - level: CAN_RUN
          group_name: data-engineers
```

```yaml
# resources/sales_job.job.yml - schedule the pipeline (details: jobs-bundle-yaml.md)
resources:
  jobs:
    sales_job:
      name: sales_job
      schedule:
        quartz_cron_expression: "0 0 5 * * ?"
        timezone_id: UTC
      tasks:
        - task_key: refresh_sales
          pipeline_task:
            pipeline_id: ${resources.pipelines.sales_pipeline.id}
```

## Configuration

| Pipeline key | Use | Notes |
|--------------|-----|-------|
| `catalog` + `schema` | Default publish location | `target` is the deprecated legacy field |
| `serverless: true` | Serverless compute | Needs edition `ADVANCED` (default); no `clusters` block |
| `libraries` | `glob.include` (folder `/**`), `file.path`, `notebook.path` | `glob` cannot be mixed with `file`/`notebook`; `whl` deprecated; JARs unsupported in UC |
| `configuration` | Spark conf / Python parameters | `spark.conf.get("key")`, SQL `${key}` |
| `parameters` | SQL named parameters (Beta) | Referenced as `:key`; overridable per run/job |
| `environment.dependencies` | pip requirements | Instead of `%pip install` in source files |
| `notifications.alerts` | `on-update-success`, `on-update-failure`, `on-update-fatal-failure`, `on-flow-failure` | Hyphenated values |
| `run_as` | `user_name`, `service_principal_name` or `group_name` | Exactly one; account groups only |
| `event_log` | `{catalog, schema, name}` | Publish the event log as a UC table |
| `cascade_on_destroy` | `false` keeps datasets on `bundle destroy` | Direct engine |
| `development` | Do not set | `mode: development` sets it to `true` |

## Example Usage

```bash
databricks bundle validate -t dev
databricks bundle plan -t dev                       # preview actions
databricks bundle deploy -t dev
databricks bundle run -t dev sales_pipeline          # incremental update
databricks bundle run -t dev sales_pipeline --full-refresh-all
databricks bundle run -t dev sales_pipeline --refresh orders_silver,daily_sales
databricks bundle run --validate-only sales_pipeline # graph validation update
databricks bundle deploy -t prod                     # from CI, as the service principal
databricks bundle deployment bind sales_pipeline <existing-pipeline-id> -t prod
```

CLI version matters: CLI v1.20.0 (2026-10-07) removed the Terraform deployment engine. Bundles that
still hold Terraform state are migrated to the direct engine on deploy; `bundle.engine: terraform`
now errors and a failed migration aborts the deploy (pin v1.19.x to keep Terraform). With the direct
engine, removing a field from YAML resets it to the resource default on the next deploy, so keep
every setting you rely on explicit. Pin the CLI in CI and set `databricks_cli_version`.

```python
# Read deployment-time configuration in Python sources
landing = spark.conf.get("landing_path")
```

## See Also

- [Deployment model](../concepts/deployment-model.md)
- [Bundles and CLI reference](../reference/bundles-cli.md)
- [Jobs bundle YAML](jobs-bundle-yaml.md)
- [Pipeline configuration](../reference/pipeline-configuration.md)

## Sources

- https://docs.databricks.com/aws/en/dev-tools/bundles/
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes
- https://docs.databricks.com/aws/en/dev-tools/bundles/direct
- https://docs.databricks.com/aws/en/dev-tools/bundles/reference
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://docs.databricks.com/aws/en/dev-tools/bundles/library-dependencies
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/limitations
- https://github.com/databricks/cli/releases/tag/v1.20.0
