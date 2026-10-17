# Task Dependencies

> **Purpose**: Dependencies, trigger rules (incl. `all_done_min_one_success`, 3.1+), XCom and its limits, asset conditions
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Tasks form a Dag through explicit dependencies (`>>`, `chain()`) or TaskFlow data flow (passing one task's
return value to another). Trigger rules decide whether a task runs given its upstream states. XCom passes
**small** values between tasks; anything large should go to object storage, either by hand or through the
object-storage XCom backend. Dag-to-Dag dependencies are best expressed with assets: `schedule=[asset]`
with `&` (AND) and `|` (OR).

## The Concept

```python
from datetime import datetime

from airflow.sdk import Asset, TriggerRule, dag, task

orders_asset = Asset(name="orders", uri="s3://my-bucket/silver/orders/")


@dag(schedule=[orders_asset], start_date=datetime(2026, 1, 1))
def downstream_analytics():
    @task
    def build_metrics() -> dict:
        return {"revenue": 150000, "orders": 1200}       # small value -> XCom

    @task
    def send_report(metrics: dict) -> None:              # pulled automatically
        print(f"Revenue: {metrics['revenue']}")

    @task(trigger_rule=TriggerRule.ALL_DONE)
    def cleanup() -> None:
        """Runs whatever the upstream outcome."""

    @task(trigger_rule=TriggerRule.ALL_DONE_MIN_ONE_SUCCESS)   # 3.1+
    def publish_partial() -> None:
        """Runs when all non-skipped upstreams are done and at least one succeeded."""

    report = send_report(build_metrics())
    report >> [cleanup(), publish_partial()]


downstream_analytics()
```

## Quick Reference

| Trigger rule | Runs when |
|--------------|-----------|
| `all_success` (default) | All upstreams succeeded |
| `all_failed` | All upstreams failed or upstream_failed |
| `all_done` | All upstreams finished (any state) |
| `all_done_min_one_success` (3.1+) | All non-skipped upstreams done and at least one succeeded |
| `all_done_setup_success` | Like `all_done`, but at least one upstream setup task succeeded (teardown default) |
| `all_skipped` | All upstreams skipped |
| `one_failed` / `one_success` | At least one failed / succeeded (does not wait for the rest) |
| `one_done` | At least one succeeded or failed |
| `none_failed` | No upstream failed (success or skipped) |
| `none_failed_min_one_success` | No upstream failed and at least one succeeded (typical join after branching) |
| `none_skipped` | No upstream skipped |
| `always` | No dependency at all |

| Feature | Mechanism | Guidance |
|---------|-----------|----------|
| XCom | Return value of `@task` | Designed for small values; not for dataframes |
| Large XCom values | `[core] xcom_backend = airflow.providers.common.io.xcom.backend.XComObjectStorageBackend` | Set `[common.io] xcom_objectstorage_path` and `xcom_objectstorage_threshold` (bytes; above it goes to object storage, only a reference stays in the DB) |
| `ti.xcom_pull(...)` | Classic operators | 3.0+: without `task_ids` it reads only the current task; pass `task_ids` |
| State across retries | `context["task_state_store"]` (3.3+) | XComs are cleared on retry, so do not use them to persist state |
| Asset schedule | `schedule=[a1, a2]` | Runs after all listed assets updated since the last run |
| AND / OR | `schedule=(a1 & a2)` / `schedule=(a1 \| a2)` | Compose freely: `a1 \| (a2 & a3)` |

## Common Mistakes

### Wrong

```python
@task
def extract() -> "pd.DataFrame":
    return huge_dataframe          # serialized into the metadata DB by the default backend
```

### Correct

```python
from airflow.sdk import task


@task
def extract(ds=None) -> str:
    path = f"s3://my-bucket/staging/extract_{ds}.parquet"
    load_source().to_parquet(path)     # load_source(): your extraction
    return path                        # lightweight reference via XCom

@task
def transform(path: str) -> None:
    import pandas as pd
    df = pd.read_parquet(path)
```

## Related

- [dynamic-task-mapping](../patterns/dynamic-task-mapping.md)
- [sensors-triggers](../patterns/sensors-triggers.md)
- [error-handling](../patterns/error-handling.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html (Trigger Rules)
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (3.1.0 `ALL_DONE_MIN_ONE_SUCCESS`)
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html
- https://airflow.apache.org/docs/apache-airflow-providers-common-io/stable/xcom_backend.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html (xcom_pull default)
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/task-and-asset-state-store.html
- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html
- https://airflow.apache.org/docs/task-sdk/stable/api.html
