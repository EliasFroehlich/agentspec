# DAG Design

> **Purpose**: Idempotency, atomicity, retries, timetables and naming for Airflow 3.x Dags; `@dag` and `@asset` authoring
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

A well-designed Dag is idempotent (re-runs produce the same result), atomic (each task does one thing) and
observable (failure notifiers, deadline alerts). Author with the Task SDK: `from airflow.sdk import dag, task`
(the Airflow 2 import paths still work but emit deprecation warnings). Airflow 3 also changed two defaults:
`catchup` is `False`, and a bare cron string now uses `CronTriggerTimetable`, so `logical_date` is the trigger
time rather than the start of the previous interval.

## The Concept

```python
from datetime import datetime, timedelta

from airflow.providers.smtp.notifications.smtp import SmtpNotifier
from airflow.sdk import dag, task

default_args = {
    "owner": "data-engineering",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": 2.0,          # 3.2+ numeric; True on 3.0/3.1
    "execution_timeout": timedelta(hours=1),
    "on_failure_callback": SmtpNotifier(       # instead of email_on_failure (legacy helper)
        to="data-alerts@example.com",
        subject="[Airflow] {{ dag.dag_id }}.{{ ti.task_id }} failed",
        html_content="Run {{ run_id }} failed.",
    ),
}


@dag(
    dag_id="daily_orders_pipeline",
    schedule="0 6 * * *",              # CronTriggerTimetable by default in 3.x
    start_date=datetime(2026, 1, 1),
    catchup=False,                     # already the default in 3.x; explicit is clearer
    max_active_runs=1,                 # prevent overlapping runs
    tags=["finance", "daily"],
    default_args=default_args,
    doc_md="Extracts orders, loads to the warehouse, runs dbt models.",
)
def daily_orders_pipeline():
    @task
    def extract_orders(ds=None) -> dict:
        """ds = logical date (YYYY-MM-DD): partition the work by it, never by datetime.now()."""
        return {"date": ds, "count": 1500}

    @task
    def load_to_warehouse(extract_result: dict) -> str:
        return f"Loaded {extract_result['count']} rows for {extract_result['date']}"

    @task
    def run_dbt() -> None: ...
    load_to_warehouse(extract_orders()) >> run_dbt()


daily_orders_pipeline()
```

## Quick Reference

| Principle | Rule | Why |
|-----------|------|-----|
| Idempotency | Key writes by `ds` / `data_interval_*` / `run_id`; overwrite partitions (MERGE/replace) | Re-runs produce the same result |
| Atomicity | One task = one logical operation | Retry granularity, clear debugging |
| No heavy top-level code | Logic and imports inside tasks | Files are re-parsed every `[dag_processor] min_file_process_interval` (30 s) |
| Retries | `retries` + backoff; per-exception `retry_policy` | 3.3+ for `retry_policy` |
| Time limits | `execution_timeout` (task), `dagrun_timeout` (run), `deadline=` alert | Deadline Alerts 3.1+ |
| Timetable | Need interval semantics? Use `CronDataIntervalTimetable(...)` explicitly | Bare cron = `CronTriggerTimetable` in 3.x |
| Overlap | `max_active_runs=1` for pipelines that cannot overlap | Prevents contention |

## Common Mistakes

### Wrong

```python
import pandas as pd
df = pd.read_csv("big_file.csv")   # runs on every parse of the file, not once per run

@dag(schedule="@daily", start_date=datetime(2026, 1, 1))
def bad_dag(): ...
```

### Correct

```python
from datetime import datetime

from airflow.sdk import dag, task


@dag(schedule="@daily", start_date=datetime(2026, 1, 1))
def good_dag():
    @task
    def process():
        import pandas as pd        # imported only when the task runs
        return len(pd.read_csv("big_file.csv"))

    process()

good_dag()
```

## Asset-Driven Example (3.0+)

```python
from datetime import datetime

from airflow.sdk import Asset, asset, dag, task


@asset(uri="s3://my-bucket/silver/orders/", schedule="@daily")   # schedule is required
def orders():
    """Creates an Asset, a Dag and a task, all named "orders"."""


@dag(schedule=[Asset(name="orders", uri="s3://my-bucket/silver/orders/")], start_date=datetime(2026, 1, 1))
def orders_consumer():
    @task
    def process_orders() -> None:
        """Runs after each update of the orders asset."""

    process_orders()


orders_consumer()
```

`@asset` takes `schedule` as a required keyword; `name` defaults to the function name and `uri` is optional.

## Related

- [operators](operators.md) · [deadline-alerts](deadline-alerts.md) · [error-handling](../patterns/error-handling.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
- https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html (`catchup_by_default`, `create_cron_data_intervals`, `[dag_processor]`)
- https://airflow.apache.org/docs/task-sdk/stable/api.html
- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/assets.html
- https://airflow.apache.org/docs/apache-airflow-providers-smtp/stable/notifications/smtp_notifier_howto_guide.html
- https://airflow.apache.org/docs/apache-airflow-providers-smtp/stable/connections/smtp.html
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html
