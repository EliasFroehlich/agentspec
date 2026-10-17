# Operators

> **Purpose**: TaskFlow `@task`, provider operators (SQL, dbt Cloud, Spark, BigQuery, Databricks, HTTP), deferrable execution, HITL operators (3.1+), and executor choice
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Operators are the building blocks of tasks. `@task` (TaskFlow) covers most Python work; the classic
`PythonOperator`, `BashOperator`, `EmptyOperator`, `ExternalTaskSensor` and friends moved out of core into
`apache-airflow-providers-standard` in Airflow 3. Provider packages supply integrations. **Deferrable**
operators hand long waits to the triggerer and free the worker slot. Where a task runs is decided by the
**executor**, and 3.x can run several executors side by side.

## The Concept

```python
from datetime import datetime

from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.providers.standard.operators.hitl import ApprovalOperator
from airflow.sdk import dag, task


@dag(schedule="@daily", start_date=datetime(2026, 1, 1))
def multi_operator_dag():
    @task
    def extract(ds=None) -> dict:
        return {"date": ds, "rows": 5000}

    run_bq = BigQueryInsertJobOperator(
        task_id="run_bq_query",
        configuration={"query": {"query": "SELECT 1", "useLegacySql": False}},
        deferrable=True,                       # worker freed while BigQuery runs
    )
    load_sf = SQLExecuteQueryOperator(         # generic SQL (Snowflake, Postgres, ...)
        task_id="clear_partition",
        conn_id="snowflake_default",
        sql="DELETE FROM analytics.orders WHERE order_date = '{{ ds }}'",  # idempotent reload
    )
    run_dbt = DbtCloudRunJobOperator(task_id="run_dbt_models", job_id=12345, deferrable=True)
    approve = ApprovalOperator(                # 3.1+: waits for a human in the UI
        task_id="approve_publish",
        subject="Publish today's revenue numbers?",
    )
    run_spark = SparkSubmitOperator(
        task_id="run_spark_job",
        application="/opt/spark/jobs/transform.py",
        conn_id="spark_default",
    )

    extract() >> run_bq >> load_sf >> run_dbt >> approve >> run_spark


multi_operator_dag()
```

## Quick Reference

| Operator | Package | Deferrable? | Notes |
|----------|---------|-------------|-------|
| `@task` / `PythonOperator` | core SDK / standard provider | No | Async callables supported in `PythonOperator` (3.2+) |
| `@task.bash` / `BashOperator` | standard provider | No | |
| `@asset` | `airflow.sdk` | N/A | Dag + task + asset in one (3.0+) |
| `SQLExecuteQueryOperator` | common.sql | No | Generic SQL for any DbApiHook connection (Snowflake, Postgres, ...) |
| `CopyFromExternalStageToSnowflakeOperator` | snowflake | No | `COPY INTO` from an external stage |
| `BigQueryInsertJobOperator` | google | Yes | |
| `DbtCloudRunJobOperator` | dbt.cloud | Yes | |
| `DatabricksSubmitRunOperator` / `DatabricksRunNowOperator` | databricks | Yes | |
| `SparkSubmitOperator` | apache.spark | No | |
| `HttpOperator` | http | Yes | Pagination, `response_check`, `response_filter` |
| `HITLOperator`, `ApprovalOperator`, `HITLBranchOperator`, `HITLEntryOperator` | standard | Waits in `awaiting_input` (3.3) | 3.1+; `response_timeout`, `defaults`, `assigned_users`, `notifiers` |

## Executors (3.x)

| Executor | Type | Use when |
|----------|------|----------|
| `LocalExecutor` | Local (in the scheduler) | Default; small deployments, dev (also replaces SequentialExecutor) |
| `CeleryExecutor` | Queued workers | Steady, high-volume workloads |
| `BatchExecutor` (AWS) | Queued | Batch jobs on AWS Batch |
| `EdgeExecutor` (`apache-airflow-providers-edge3`) | Queued, remote over HTTPS | Workers behind firewalls / remote sites |
| `KubernetesExecutor` | Container per task | Isolation and per-task resources |
| `EcsExecutor` (AWS) | Container per task | Isolation on ECS/Fargate |

Multiple executors (2.10+): `[core] executor = CeleryExecutor,KubernetesExecutor` (aliases allowed, e.g.
`my_k8s:KubernetesExecutor`). The first one is the default; pick another per task with
`@task(executor="KubernetesExecutor")` or for the whole Dag via `default_args={"executor": ...}`.
The hard-coded hybrid executors were dropped in 3.0 in favour of this.

## Common Mistakes

### Wrong

```python
run_bq = BigQueryInsertJobOperator(task_id="bq_query", configuration={...})
# no deferrable=True: a worker slot sits idle for the whole query

# Migration only: removed provider operators
# SimpleHttpOperator        -> HttpOperator            (http provider 5.0.0, 2024-12-26)
# SnowflakeOperator         -> SQLExecuteQueryOperator (snowflake provider 6.0.0, 2024-12-26)
# S3ToSnowflakeOperator     -> CopyFromExternalStageToSnowflakeOperator (snowflake provider 5.0.0)
```

### Correct

```python
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator

run_bq = BigQueryInsertJobOperator(task_id="bq_query", configuration={...}, deferrable=True)
# or make it the deployment default: [operators] default_deferrable = True  (default: False)
```

## Related

- [dag-design](dag-design.md)
- [task-dependencies](task-dependencies.md)
- [sensors-triggers](../patterns/sensors-triggers.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html (standard provider)
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-edge3/stable/index.html
- https://airflow.apache.org/docs/apache-airflow/stable/tutorial/hitl.html
- https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/hitl/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-http/stable/changelog.html
- https://airflow.apache.org/docs/apache-airflow-providers-snowflake/stable/changelog.html
- https://airflow.apache.org/docs/apache-airflow-providers-google/stable/_api/airflow/providers/google/cloud/operators/bigquery/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-dbt-cloud/stable/_api/airflow/providers/dbt/cloud/operators/dbt/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-databricks/stable/_api/airflow/providers/databricks/operators/databricks/index.html
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (3.2.0 async PythonOperator)
- https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html (`default_deferrable`)
