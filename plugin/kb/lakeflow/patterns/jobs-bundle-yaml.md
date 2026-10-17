# Lakeflow Jobs in Bundle YAML

> **Purpose**: A production-shaped Lakeflow Job defined in Declarative Automation Bundles YAML: mixed task types, parameters, branching, loops, triggers, compute, retries, notifications, and run/repair commands
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Orchestrating notebooks, Python scripts/wheels, SQL files and pipelines as one DAG
- Running the same task per item (table, country, partition) with `for_each_task`
- Event-driven runs (file arrival, table update) instead of a cron schedule

## Implementation

```yaml
# resources/daily_sales.job.yml
resources:
  jobs:
    daily_sales:
      name: daily_sales
      max_concurrent_runs: 1
      queue: { enabled: true }
      timeout_seconds: 14400
      performance_target: STANDARD          # serverless: STANDARD | PERFORMANCE_OPTIMIZED
      parameters:                            # job parameters (pushed to key-value tasks)
        - name: run_date
          default: "{{job.trigger.time.iso_date}}"
        - name: catalog
          default: ${var.catalog}            # bundle variable, resolved at deploy time
      trigger:                               # or `schedule:` (cron) / `continuous:`
        pause_status: UNPAUSED
        table_update:
          table_names: ["${var.catalog}.raw.orders"]
          condition: ANY_UPDATED             # ANY_UPDATED | ALL_UPDATED
          wait_after_last_change_seconds: 300
      email_notifications:
        on_failure: ["data-team@example.com"]
      webhook_notifications:
        on_failure: [{ id: "<notification-destination-id>" }]
      notification_settings: { no_alert_for_skipped_runs: true }
      health:
        rules:
          - { metric: RUN_DURATION_SECONDS, op: GREATER_THAN, value: 7200 }
      environments:                          # serverless Python environments
        - environment_key: default
          spec:
            environment_version: "5"
            dependencies: ["pyyaml==6.0.2"]
      job_clusters:                          # classic compute shared by tasks
        - job_cluster_key: heavy
          new_cluster:
            spark_version: 17.3.x-scala2.13  # DBR 17.3 LTS
            node_type_id: i3.xlarge          # AWS example; use your cloud's type
            autoscale: { min_workers: 2, max_workers: 8 }
            data_security_mode: DATA_SECURITY_MODE_STANDARD
      tasks:
        - task_key: ingest                   # serverless notebook (no compute keys)
          notebook_task:
            notebook_path: ../src/ingest.ipynb
          max_retries: 2
          min_retry_interval_millis: 60000
          retry_on_timeout: true
          timeout_seconds: 3600

        - task_key: refresh_pipeline
          depends_on: [{ task_key: ingest }]
          pipeline_task:
            pipeline_id: ${resources.pipelines.sales_pipeline.id}
            full_refresh: false

        - task_key: has_rows                 # If/else: no compute, no retries
          depends_on: [{ task_key: refresh_pipeline }]
          condition_task:
            op: GREATER_THAN
            left: "{{tasks.ingest.values.row_count}}"
            right: "0"

        - task_key: validate_inputs           # serverless Python script
          depends_on: [{ task_key: has_rows, outcome: "true" }]
          spark_python_task:
            python_file: ../src/validate_inputs.py
            parameters: ["--date", "{{job.parameters.run_date}}"]
          environment_key: default           # required for serverless Python script/wheel

        - task_key: list_regions             # Python notebook sets task value "regions"
          depends_on: [{ task_key: validate_inputs }]
          notebook_task:
            notebook_path: ../src/list_regions.ipynb

        - task_key: per_region
          depends_on: [{ task_key: list_regions }]
          for_each_task:
            inputs: "{{tasks.list_regions.values.regions}}"
            concurrency: 4
            task:
              task_key: per_region_iteration
              job_cluster_key: heavy
              notebook_task:
                notebook_path: ../src/aggregate_region.ipynb
                base_parameters: { region: "{{input}}" }

        - task_key: publish_sql
          depends_on: [{ task_key: per_region }]
          sql_task:
            warehouse_id: ${var.warehouse_id}   # declared under `variables`
            file: { path: ../src/publish.sql }
            parameters: { run_date: "{{job.parameters.run_date}}" }

        - task_key: cleanup                  # runs whatever happened upstream
          depends_on: [{ task_key: publish_sql }, { task_key: has_rows }]
          run_if: ALL_DONE
          notebook_task:
            notebook_path: ../src/cleanup.ipynb
      permissions:
        - { level: CAN_MANAGE_RUN, group_name: data-engineers }
```

## Configuration

| Topic | Rule |
|-------|------|
| Serverless vs classic | A task with no `job_cluster_key` / `existing_cluster_id` / `new_cluster` runs serverless where supported (notebook, Python script, Python wheel, dbt, JAR) |
| `environment_key` | Required for serverless Python script, Python wheel and dbt tasks |
| Task values | `dbutils.jobs.taskValues.set()` works only in Python notebooks; read with `{{tasks.<key>.values.<name>}}` |
| Job clusters | Shared by tasks of one run; libraries go on tasks, not on the shared cluster |
| `depends_on.outcome` | Only for edges from a `condition_task` (`"true"` / `"false"`) |
| `run_if` | `ALL_SUCCESS` (default), `AT_LEAST_ONE_SUCCESS`, `NONE_FAILED`, `ALL_DONE`, `AT_LEAST_ONE_FAILED`, `ALL_FAILED` |
| `for_each_task` | No nested For each; `{{input}}` / `{{input.<key>}}`; set `concurrency` explicitly (UI default 1) |
| Pipeline in a loop or with `max_concurrent_runs > 1` | Concurrency is capped to 1 (a pipeline runs one update at a time) |
| Triggers | One of `schedule`, `trigger` (file_arrival / table_update / periodic), `continuous`; or the Beta `triggers` list (max 5), never mixed with the legacy fields |
| `development` mode | Schedules and triggers deploy paused; set `pause_status: UNPAUSED` to override |
| Serverless retries | Auto-optimization may retry; set `disable_auto_optimization: true` on non-idempotent tasks |

## Example Usage

```bash
databricks bundle validate -t prod
databricks bundle deploy -t prod
databricks bundle run -t prod daily_sales --params run_date=2026-10-16
databricks bundle run -t prod daily_sales --only +per_region     # task and its upstream
databricks jobs repair-run <run-id> --rerun-all-failed-tasks       # re-run failed tasks only
```

Repair re-runs failed tasks and their dependents with the current job settings; it does not make
tasks idempotent, so write with overwrite/MERGE where duplicates matter. Repair applies to jobs with
two or more tasks; re-run single-task jobs with Run now.

## See Also

- [Lakeflow Jobs reference](../reference/lakeflow-jobs.md)
- [DABs deployment](dabs-deployment.md)
- [Bundles and CLI reference](../reference/bundles-cli.md)

## Sources

- https://docs.databricks.com/aws/en/dev-tools/bundles/job-task-types
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/dev-tools/bundles/examples
- https://docs.databricks.com/aws/en/dev-tools/bundles/permissions
- https://docs.databricks.com/aws/en/jobs/dynamic-value-references
- https://docs.databricks.com/aws/en/jobs/task-values
- https://docs.databricks.com/aws/en/jobs/tasks/for-each
- https://docs.databricks.com/aws/en/jobs/tasks/if-else
- https://docs.databricks.com/aws/en/jobs/run-if
- https://docs.databricks.com/aws/en/jobs/triggers
- https://docs.databricks.com/aws/en/jobs/trigger-table-update
- https://docs.databricks.com/aws/en/jobs/run-serverless-jobs
- https://docs.databricks.com/aws/en/jobs/compute
- https://docs.databricks.com/aws/en/jobs/repair-job-failures
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- https://github.com/databricks/databricks-sdk-py/blob/main/databricks/sdk/service/jobs.py
- https://github.com/databricks/databricks-sdk-py/blob/main/databricks/sdk/service/compute.py
- https://github.com/databricks/cli/releases/tag/v1.15.0
