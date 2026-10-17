# Deployment Model

> **Purpose**: How pipelines and jobs are deployed with Declarative Automation Bundles (DABs, formerly Databricks Asset Bundles): targets, modes, identity, CLI engine
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

Declarative Automation Bundles (formerly known as Databricks Asset Bundles; the DABs acronym is kept)
describe jobs, pipelines and other resources as YAML (or Python) next to the source code. The
Databricks CLI validates, plans, deploys and runs a bundle per target. Since CLI v1.20.0 (2026-10-07)
bundles deploy only with the direct engine; the Terraform engine was removed.

## The Concept

```yaml
# databricks.yml
bundle:
  name: sales_etl
  databricks_cli_version: ">= 1.20.0"   # fail validate on CLIs that still allow Terraform

include:
  - resources/*.yml

variables:
  catalog:
    default: dev_catalog

targets:
  dev:
    mode: development
    default: true
  prod:
    mode: production
    variables:
      catalog: prod_catalog
    run_as:
      service_principal_name: "<service-principal-application-id>"
```

```yaml
# resources/sales_pipeline.yml
resources:
  pipelines:
    sales_pipeline:
      name: sales_pipeline
      catalog: ${var.catalog}
      schema: sales                # `target` is the deprecated legacy field
      serverless: true             # requires edition ADVANCED (the default)
      channel: CURRENT
      root_path: ../src/sales_pipeline
      libraries:
        - glob:
            include: ../src/sales_pipeline/transformations/**
```

## Deployment Modes

| Behavior | `mode: development` | `mode: production` |
|----------|---------------------|--------------------|
| Resource names | Prefixed `[dev <short_name>]`, `dev` tag | As written |
| Pipelines | Deployed with `development: true` | Validates `development: false` |
| Schedules and triggers | Paused | As written |
| Job concurrency | Concurrent runs enabled | As written |
| Deployment lock | Disabled | Enabled |
| `--cluster-id` override | Allowed | Not allowed |
| Git branch | Not checked | Must match `git.branch` if set (`--force` overrides) |
| Identity | Deploying user | Recommended `run_as` service principal |

Presets (`name_prefix`, `pipelines_development`, `trigger_pause_status`, `jobs_max_concurrent_runs`,
`tags`) customize these behaviors; resource settings override presets.

## Quick Reference

| Command | Purpose |
|---------|---------|
| `databricks bundle validate -t dev` | Syntax, references, `databricks_cli_version` |
| `databricks bundle plan -t dev` | Show create/update/delete actions without deploying |
| `databricks bundle deploy -t dev` | Create or update resources |
| `databricks bundle run -t dev sales_pipeline` | Run a pipeline or job by resource key |
| `databricks bundle run --validate-only sales_pipeline` | Pipeline graph validation update |
| `databricks bundle destroy -t dev` | Delete deployed resources |
| `databricks bundle deployment bind <key> <id>` | Adopt an existing resource |

| CLI version | Bundle engine behavior |
|-------------|------------------------|
| 0.279.0+ | Two engines available: `terraform` and `direct` |
| 1.3.0+ | New bundles use the direct engine by default |
| 1.14 - 1.19 | Terraform-state bundles auto-migrate to direct on deploy; `engine: terraform` still honored |
| >= 1.20.0 | Terraform engine removed; `engine: terraform` errors; failed auto-migration aborts the deploy |

Bundle variables are resolved at deploy time; pass run-time values as job parameters instead.

## Common Mistakes

### Wrong

```yaml
resources:
  pipelines:
    etl:
      target: "prod_catalog.etl"   # legacy field, hardcoded environment
      development: true            # let `mode` set this
      libraries:
        - notebook:
            path: ../src/etl.sql   # files are not notebooks
```

### Correct

```yaml
resources:
  pipelines:
    etl:
      catalog: ${var.catalog}
      schema: etl
      libraries:
        - file:
            path: ../src/etl.sql
```

## Related

- [DABs deployment pattern](../patterns/dabs-deployment.md)
- [Bundles and CLI reference](../reference/bundles-cli.md)
- [Jobs bundle YAML](../patterns/jobs-bundle-yaml.md)

## Sources

- https://docs.databricks.com/aws/en/dev-tools/bundles/
- https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes
- https://docs.databricks.com/aws/en/dev-tools/bundles/direct
- https://docs.databricks.com/aws/en/dev-tools/bundles/reference
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/dev-tools/bundles/variables
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://github.com/databricks/cli/releases/tag/v1.20.0
