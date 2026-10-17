# Airflow Quick Reference

> **Purpose**: Fast lookup for Airflow 3.x (stable 3.3.2, 2026-09-17; Python 3.10–3.14). For code, see linked files.
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Versions (as of 2026-10-17)

| Release | Date | Headline changes |
|---------|------|------------------|
| 3.0.0 | 2025-04-22 | Task SDK (`airflow.sdk`), assets, DAG versioning, React UI, Edge Executor, `airflowctl`; SLA/SubDAG removed |
| 3.1.0 | 2025-09-25 | Deadline Alerts, HITL operators, `ALL_DONE_MIN_ONE_SUCCESS`; Python 3.9 dropped, 3.13 added |
| 3.2.0 | 2026-04-07 | Asset partitioning, multi-team (experimental), `SyncCallback`, numeric backoff, Python 3.14 |
| 3.3.0 | 2026-07-06 | Task/Asset State Store, pluggable retry policies, Java/Go Task SDK (experimental), partition rollups |

## Imports (Airflow 3.x)

| Need | Import |
|------|--------|
| Authoring | `from airflow.sdk import dag, task, DAG, task_group, TaskGroup, TriggerRule, chain, Label` |
| Assets | `from airflow.sdk import Asset, AssetAlias, AssetWatcher, asset, Metadata` |
| Hooks/conn/vars | `from airflow.sdk import BaseHook, Connection, Variable, get_current_context` |
| Deadlines (3.2+ path) | `from airflow.sdk import DeadlineAlert, DeadlineReference, AsyncCallback, SyncCallback` |
| Retry policy (3.3+) | `from airflow.sdk import ExceptionRetryPolicy, RetryRule, RetryAction` |
| Core operators/sensors | `airflow.providers.standard.operators.{python,bash,empty,trigger_dagrun,hitl}`, `...sensors.external_task` |

## Operator Selection Matrix

| Task type | Recommended | Notes |
|-----------|-------------|-------|
| Python function | `@task` (TaskFlow) | Default choice |
| Shell command | `@task.bash` / `BashOperator` (standard provider) | |
| SQL on any DB | `SQLExecuteQueryOperator` (common.sql) | Snowflake, Postgres, ... |
| dbt Cloud | `DbtCloudRunJobOperator(deferrable=True)` | dbt Core: `@task.bash` |
| Databricks | `DatabricksSubmitRunOperator` / `DatabricksRunNowOperator` (deferrable) | |
| BigQuery | `BigQueryInsertJobOperator(deferrable=True)` | |
| HTTP call | `HttpOperator` (http provider) | Deferrable |
| Wait for file | `S3KeySensor(deferrable=True)` | |
| Wait for data from another Dag | Asset schedule (`schedule=[asset]`) | `ExternalTaskSensor` only for task-level coupling |
| Human approval | `ApprovalOperator` / `HITLOperator` | 3.1+ |

## Schedules

| Expression | Meaning | Since |
|------------|---------|-------|
| `"0 6 * * *"` (bare cron) | `CronTriggerTimetable` by default in 3.x: `logical_date` = trigger time | 3.0 default |
| `schedule=[a1, a2]` / `(a1 & a2)` / `(a1 \| a2)` | Asset-driven (all / and / or) | 3.0 |
| `Asset(..., watchers=[AssetWatcher(...)])` | External event (Kafka, SQS) | 3.0 |
| `AssetOrTimeSchedule(...)` | Asset events or a timetable | 3.0 |
| `CronPartitionTimetable` / `PartitionedAssetTimetable` | Partitioned producer / consumer | 3.2 |
| `RollupMapper`, `FanOutMapper`, `PartitionedAtRuntime` | Partition rollups / fan-out / runtime keys | 3.3 |

## Removed / renamed (migration)

| Removed | Use instead |
|---------|-------------|
| `Dataset` (2.x) | `Asset` (3.0) |
| `SimpleHttpOperator` (http provider 5.0.0) / `SnowflakeOperator` (snowflake 6.0.0) | `HttpOperator` / `SQLExecuteQueryOperator` |
| 3.0: SLA (`sla=`, `sla_miss_callback`) | Deadline Alerts (3.1+), see `concepts/deadline-alerts.md` |
| 3.0: SubDAGs | TaskGroups, assets |
| 3.0: `execution_date` (and `next_ds`, `prev_ds`, ...) in context | `logical_date`, `data_interval_start/end` |
| 3.0: SequentialExecutor; CeleryKubernetes/LocalKubernetes executors | LocalExecutor; multi-executor `executor = CeleryExecutor,KubernetesExecutor` |
| 3.0: direct metadata DB access from task code | Task SDK APIs / REST API |
| 3.0: `[scheduler] dag_dir_list_interval` | `[dag_processor] refresh_interval` |

## Config: `[dag_processor]` (3.x)

| Key | Default | Meaning |
|-----|---------|---------|
| `refresh_interval` | 300 | Seconds between scans of a Dag bundle for new files |
| `min_file_process_interval` | 30 | Seconds between re-parses of a Dag file |
| `parsing_processes` | 2 | Parallel parsing processes |
| `dag_file_processor_timeout` | 50 | Seconds before a file-processor is killed |

## Executors (3.x)

Local: `LocalExecutor` (default). Queued: `CeleryExecutor`, `BatchExecutor`, `EdgeExecutor` (provider `edge3`).
Containerized: `KubernetesExecutor`, `EcsExecutor`. Combine with a comma list; the first is the default and
tasks pick another with `executor="..."`.

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Heavy top-level code in Dag files | Imports and I/O inside tasks (files re-parse every 30 s by default) |
| Large values in XCom | Pass references; or `XComObjectStorageBackend` with `xcom_objectstorage_threshold` |
| `ti.xcom_pull(key=...)` without `task_ids` | Pass `task_ids` (3.0 pulls only from the current task by default) |
| Blocking sensors for long waits | `deferrable=True` (requires a running triggerer) |
| Hand-built Slack/email operators in callbacks | Notifiers: `SlackWebhookNotifier`, `SmtpNotifier`, `PagerdutyNotifier` |

## Related Documentation

Getting started: `concepts/dag-design.md` · Full index: `index.md`

## Sources

https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html · https://airflow.apache.org/docs/apache-airflow/stable/installation/prerequisites.html · https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
https://airflow.apache.org/docs/task-sdk/stable/api.html · https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html · https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html
https://airflow.apache.org/docs/apache-airflow-providers-http/stable/changelog.html · https://airflow.apache.org/docs/apache-airflow-providers-snowflake/stable/changelog.html · https://airflow.apache.org/docs/apache-airflow-providers-common-io/stable/xcom_backend.html
