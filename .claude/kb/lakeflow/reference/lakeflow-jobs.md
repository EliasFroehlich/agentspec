# Lakeflow Jobs Reference

> **Purpose**: Lakeflow Jobs as defined in Declarative Automation Bundles YAML (same shape as the Jobs API 2.1+): task types, parameters, dynamic value references, dependencies, triggers, compute, retries, notifications, repair
> **Confidence**: High
> **MCP Validated**: 2026-10-17

Lakeflow Jobs is the workflow automation service of Databricks (the docs no longer use the name
"Workflows"). A job has one or more tasks forming a DAG. This page lists the YAML keys used under
`resources.jobs.<key>` in a bundle; the same fields appear in the Jobs API and `databricks jobs` CLI.
Pattern with a full example: [jobs-bundle-yaml.md](../patterns/jobs-bundle-yaml.md).

---

## 1. Task types

| UI name | YAML key | Key fields | Serverless? |
|---------|----------|-----------|-------------|
| Notebook | `notebook_task` | `notebook_path`, `base_parameters`, `warehouse_id` (SQL-only notebooks) | Yes (recommended) |
| Python script | `spark_python_task` | `python_file`, `parameters` (list) | Yes, needs `environment_key` |
| Python wheel | `python_wheel_task` | `package_name`, `entry_point`, `parameters` or `named_parameters` | Yes, needs `environment_key` |
| SQL | `sql_task` | `warehouse_id` + one of `file.path`, `query.query_id`, `alert`, `dashboard`; `parameters` | Runs on a SQL warehouse |
| Pipeline | `pipeline_task` | `pipeline_id`, `full_refresh` | Runs on the pipeline's compute |
| dbt | `dbt_task` | `commands` (max 10), `project_directory`, `warehouse_id` | Yes, needs `environment_key` |
| JAR | `spark_jar_task` | `main_class_name`, `parameters`; JAR via task `libraries` | Yes (classic recommended) |
| Run Job | `run_job_task` | `job_id`, `job_parameters`, `pipeline_params` | - |
| If/else | `condition_task` | `left`, `op`, `right` | No compute; no retries or notifications |
| For each | `for_each_task` | `inputs`, `task`, `concurrency` | Nested task decides |
| Dashboard | `dashboard_task` | `dashboard_id`, `subscription`, `warehouse_id` | - |
| SQL Alert (Public Preview) | `alert_task` | `alert_id` or `workspace_path`, `warehouse_id`, `subscribers` | - |
| Power BI (Public Preview) | `power_bi_task` | `connection_resource_name`, `power_bi_model`, `tables` | - |
| Clean Room notebook | `clean_rooms_notebook_task` | `clean_room_name`, `notebook_name` | - |
| Spark Submit | `spark_submit_task` | Marked legacy in the Jobs API; classic only | No |

Other UI task types (Visual data prep, Lakeflow Integrations Beta, Database Table Sync pipeline,
Ingestion pipeline, dbt platform, Genie Code Beta) exist; check the configure-task page before use.

Bundle paths (`notebook_path`, `python_file`, `sql_task.file.path`) are relative to the YAML file and
are uploaded with the bundle. `git_source` / `source: GIT` is not recommended inside bundles.

## 2. Settings common to every task

| Key | Meaning |
|-----|---------|
| `task_key` | Unique name (required) |
| `depends_on` | `[{task_key, outcome?}]`; `outcome` only after a `condition_task` |
| `run_if` | `ALL_SUCCESS` (default), `AT_LEAST_ONE_SUCCESS`, `NONE_FAILED`, `ALL_DONE`, `AT_LEAST_ONE_FAILED`, `ALL_FAILED` |
| `job_cluster_key` / `existing_cluster_id` / `new_cluster` | Classic compute; omit all for serverless |
| `environment_key` | Serverless environment (from job `environments`) |
| `libraries` | `whl`, `jar`, `pypi`, `maven`, `cran`, `requirements` |
| `max_retries`, `min_retry_interval_millis`, `retry_on_timeout` | Retry policy (default: no retries in most configurations) |
| `timeout_seconds` | Per run of the task; applies to each retry; `0` = none |
| `disable_auto_optimization` | Serverless: stop auto-optimization retries (non-idempotent tasks) |
| `email_notifications`, `webhook_notifications`, `notification_settings` | Task-level alerts |
| `health.rules` | `RUN_DURATION_SECONDS` or `STREAMING_BACKLOG_*` with `op: GREATER_THAN` |
| `description`, `compute.hardware_accelerator` | Optional |

`run_if` details: unmet `ALL_SUCCESS`/`AT_LEAST_ONE_SUCCESS`/`NONE_FAILED` marks the task
`Upstream failed`; unmet `AT_LEAST_ONE_FAILED`/`ALL_FAILED` marks it `Excluded` (skipped). Excluded
upstream tasks count as successful; if all dependencies are excluded the task is excluded too.

## 3. Job-level keys

| Key | Meaning |
|-----|---------|
| `name`, `description`, `tags` (max 25) | Metadata; tags propagate to job clusters |
| `parameters` | `[{name, default}]` job parameters |
| `tasks` | Task list |
| `job_clusters` | `[{job_cluster_key, new_cluster}]` shared classic clusters |
| `environments` | `[{environment_key, spec: {environment_version, dependencies}}]` for serverless |
| `schedule` | `quartz_cron_expression`, `timezone_id` (both required), `pause_status` |
| `trigger` | `file_arrival`, `table_update`, `periodic`, `pause_status` |
| `continuous` | `pause_status`, `task_retry_mode` (`NEVER` / `ON_FAILURE`) |
| `triggers` | Beta list of triggers (CLI 1.13.0+) |
| `max_concurrent_runs` | Default 1; extra runs are skipped unless queued |
| `queue.enabled` | Queue runs (up to 48 hours) instead of skipping |
| `timeout_seconds` | Whole-run timeout |
| `performance_target` | Serverless: `STANDARD` or `PERFORMANCE_OPTIMIZED` |
| `email_notifications` | `on_start`, `on_success`, `on_failure`, `on_duration_warning_threshold_exceeded`, `on_streaming_backlog_exceeded` (+ maintenance events, CLI 1.20.0+) |
| `webhook_notifications` | Same events; items are `{id: <notification destination id>}`, max 3 per event |
| `notification_settings` | `no_alert_for_skipped_runs`, `no_alert_for_canceled_runs` |
| `health` | Job-level health rules |
| `run_as` | Exactly one of `user_name`, `service_principal_name`, `group_name` |
| `permissions` | Jobs: `CAN_VIEW`, `CAN_MANAGE_RUN`, `CAN_MANAGE`, `IS_OWNER` |
| `usage_policy_id`, `budget_policy_id` | Serverless cost attribution |
| `environment_variables` | Beta (CLI 1.20.0+) |

## 4. Parameters and dynamic value references

| Kind | Where | Notes |
|------|-------|-------|
| Job parameters | `parameters:` on the job | Keys: alphanumeric, `_`, `-`, `.`; values strings or JSON; overridable per run and on repair |
| Task parameters | `base_parameters` (notebook), `parameters` (script/wheel/SQL/JAR), `named_parameters` | Key-value tasks receive job parameters automatically (job wins on key clash); JSON-array tasks must reference `{{job.parameters.<name>}}` |
| Task values | `dbutils.jobs.taskValues.set(key, value)` in Python notebooks | Read with `{{tasks.<task>.values.<key>}}` |
| Bundle variables | `${var.name}` | Resolved at deploy time, not per run |

| Reference | Value |
|-----------|-------|
| `{{job.id}}`, `{{job.name}}`, `{{job.run_id}}`, `{{job.repair_count}}` | Job/run identity |
| `{{job.start_time.<arg>}}`, `{{job.trigger.time.<arg>}}` | UTC time; args `iso_date`, `iso_datetime`, `year`, `month`, `day`, `hour`, `minute`, `second`, `iso_weekday`, `is_weekday` |
| `{{job.parameters.<name>}}` | Job parameter |
| `{{job.trigger.type}}` | `periodic`, `one_time`, `run_job_task`, `file_arrival`, `continuous`, `table`, `model` |
| `{{job.trigger.file_arrival.location}}` | Monitored path |
| `{{job.trigger.table_update.updated_tables}}` | JSON list of updated tables |
| `{{task.name}}`, `{{task.run_id}}`, `{{task.execution_count}}` | Current task |
| `{{tasks.<t>.result_state}}`, `{{tasks.<t>.error_code}}`, `{{tasks.<t>.run_id}}` | Upstream task state |
| `{{tasks.<t>.values.<k>}}` | Task value (JSON array of per-iteration values if `<t>` ran inside For each) |
| `{{tasks.<t>.output.rows}}`, `{{tasks.<t>.output.first_row.<col>}}` | Upstream SQL task output (max 1,000 rows / 48 KB) |
| `{{workspace.id}}`, `{{workspace.url}}` | Workspace |
| `{{input}}`, `{{input.<key>}}` | Current For each element |

Braces are not expressions (no functions inside). References outside known namespaces are left as
literal text; an invalid reference in a known namespace (for example `{{job.notebook_url}}`) errors.

## 5. Control flow

- **If/else (`condition_task`)**: `op` in `EQUAL_TO`, `NOT_EQUAL`, `GREATER_THAN`,
  `GREATER_THAN_OR_EQUAL`, `LESS_THAN`, `LESS_THAN_OR_EQUAL`. `==`/`!=` compare as strings
  (`12.0 == 12` is false); `>`, `>=`, `<`, `<=` compare numerically. Downstream tasks use
  `depends_on: [{task_key: <cond>, outcome: "true"}]`.
- **For each (`for_each_task`)**: `inputs` is a JSON array string, a task value reference, a SQL task
  `output.rows`, or a job parameter. Limits: 5,000 characters in the Inputs box, 48 KB through task
  value references, 10,000 characters for job parameter values. No nested For each. The UI default
  concurrency is 1, while the bundle reference describes an unset value as "may run in parallel
  subject to limits": always set `concurrency` explicitly. Downstream tasks depend on the For each
  task, not the nested task.
- **Run Job (`run_job_task`)**: trigger another job by ID, e.g. `${resources.jobs.other.id}`.
- **Continuous jobs**: one active run, restarts within ~60 s, exponential backoff on failure, no
  task dependencies and no retry policies (use `task_retry_mode`).

## 6. Triggers

| Trigger | YAML | Notes |
|---------|------|-------|
| Scheduled | `schedule: {quartz_cron_expression, timezone_id, pause_status}` | Quartz cron, e.g. `0 0 5 * * ?` |
| File arrival | `trigger.file_arrival: {url, min_time_between_triggers_seconds, wait_after_last_change_seconds}` | UC volume or external location; checked about every minute; no wildcards; without file events max 50 jobs per workspace and 10,000 files per path |
| Table update | `trigger.table_update: {table_names, condition, min_time_between_triggers_seconds, wait_after_last_change_seconds}` | `condition`: `ANY_UPDATED` or `ALL_UPDATED`; run-as needs `SELECT` on each table; timing options minimum 60 s |
| Periodic | `trigger.periodic: {interval, unit}` | `unit`: `HOURS`, `DAYS`, `WEEKS` |
| Continuous | `continuous: {pause_status, task_retry_mode}` | Serverless only with bounded triggers such as `Trigger.AvailableNow` |
| Model update | UI / API | Beta |
| Multiple | `triggers: [...]` | Beta, max 5 per job, each item one type; not with legacy `schedule`/`trigger`/`continuous`; continuous cannot be combined |

Workspace limit: 12,000 active triggers. In bundle `mode: development` every schedule/trigger is
deployed paused.

## 7. Compute

| Task | Recommended | Supported |
|------|-------------|-----------|
| Notebook, Python script, Python wheel, dbt CLI | Serverless jobs | Serverless, classic jobs, classic all-purpose |
| SQL, dbt (warehouse) | Serverless SQL warehouse | Serverless or pro SQL warehouse |
| Pipeline | Serverless pipeline | Serverless or classic pipeline |
| JAR | Classic jobs | Classic jobs, all-purpose |
| Spark Submit | Classic jobs | Classic jobs |

- Serverless jobs need Unity Catalog and standard access mode; Databricks recommends serverless for
  all supported tasks. Standard performance mode starts in about 4-6 minutes and uses fewer DBUs;
  `PERFORMANCE_OPTIMIZED` starts faster (same SKU). Only allow-listed Spark confs can be set, at
  session level. Serverless auto-optimization can add retries.
- Job clusters (`job_clusters`) start with the first task that uses them and stop after the last;
  tasks on a shared cluster share the driver JVM. A repair creates a new job cluster
  (`<key>_v1`). Libraries are declared on tasks, not on the shared cluster. Classic access modes:
  `DATA_SECURITY_MODE_STANDARD` (alias `USER_ISOLATION`) or `DATA_SECURITY_MODE_DEDICATED`
  (alias `SINGLE_USER`).
- Pipeline tasks: a pipeline runs one update at a time, so concurrency is capped to 1 when the job
  has `max_concurrent_runs > 1` or the pipeline task is inside a For each. In a continuous job the
  pipeline runs continuously regardless of its own pipeline mode.

## 8. Repair and re-run

- Repair re-runs failed/canceled tasks and their dependents in the same run, using the current job
  and task settings; parameters can be overridden. Only for jobs with 2+ tasks.
- Repair does not make tasks idempotent: a task that wrote part of its output can duplicate data.
- CLI: `databricks jobs repair-run RUN_ID --rerun-all-failed-tasks [--rerun-dependent-tasks]`
  (`--latest-repair-id`, `--performance-target`, or `--json` with `rerun_tasks`, `job_parameters`).
- Bundle: `databricks bundle run <job> --only <task_key>[,+upstream,downstream+]` runs a subset in a
  new run; `--restart` cancels and restarts an active run.

## 9. Bundle commands for jobs

```bash
databricks bundle validate -t <target>
databricks bundle plan -t <target>
databricks bundle deploy -t <target>
databricks bundle run -t <target> <job_key> --params key1=v1,key2=v2
databricks bundle run -t <target> <job_key> --only +task_b+
databricks bundle generate job --existing-job-id <id> --bind # YAML from an existing job (notebook tasks only)
databricks bundle deployment bind <job_key> <job_id> -t <target>
```

`--params` is the recommended way to pass job parameters; task-parameter flags (`--notebook-params`,
`--python-params`, ...) error if the job defines job parameters.

## Sources

- https://docs.databricks.com/aws/en/jobs/
- https://docs.databricks.com/aws/en/jobs/configure-task
- https://docs.databricks.com/aws/en/jobs/configure-job
- https://docs.databricks.com/aws/en/jobs/compute
- https://docs.databricks.com/aws/en/jobs/run-serverless-jobs
- https://docs.databricks.com/aws/en/jobs/parameters
- https://docs.databricks.com/aws/en/jobs/job-parameters
- https://docs.databricks.com/aws/en/jobs/task-parameters
- https://docs.databricks.com/aws/en/jobs/task-values
- https://docs.databricks.com/aws/en/jobs/dynamic-value-references
- https://docs.databricks.com/aws/en/jobs/run-if
- https://docs.databricks.com/aws/en/jobs/tasks/if-else
- https://docs.databricks.com/aws/en/jobs/tasks/for-each
- https://docs.databricks.com/aws/en/jobs/tasks/pipeline
- https://docs.databricks.com/aws/en/jobs/triggers
- https://docs.databricks.com/aws/en/jobs/file-arrival-triggers
- https://docs.databricks.com/aws/en/jobs/trigger-table-update
- https://docs.databricks.com/aws/en/jobs/continuous
- https://docs.databricks.com/aws/en/jobs/repair-job-failures
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/dev-tools/bundles/job-task-types
- https://docs.databricks.com/aws/en/dev-tools/bundles/permissions
- https://docs.databricks.com/aws/en/dev-tools/bundles/deployment-modes
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- https://github.com/databricks/databricks-sdk-py/blob/main/databricks/sdk/service/jobs.py
- https://github.com/databricks/databricks-sdk-py/blob/main/databricks/sdk/service/compute.py
