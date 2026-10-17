# Deadline Alerts

> **Purpose**: Time-based alerting for Dag runs (Airflow 3.1+), the replacement for the SLA feature removed in 3.0
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

The SLA feature was removed in Airflow 3.0 (see the migration block below). Airflow 3.1 added **Deadline
Alerts**: when a Dag run is created, Airflow stores `reference + interval`, and the scheduler checks
periodically (every `scheduler_heartbeat_sec`, default 5 s). If the run has not finished when the deadline
passes, the callback runs right away. It does not wait for the run to finish, as SLAs did.
Deadline Alerts are still marked **experimental** in the 3.3.2 docs.

## The Concept

```python
# Airflow 3.2+ import path (on 3.1: from airflow.sdk.definitions.deadline import ...)
from datetime import datetime, timedelta

from airflow.providers.slack.notifications.slack_webhook import SlackWebhookNotifier
from airflow.sdk import AsyncCallback, DeadlineAlert, DeadlineReference, dag, task


@dag(
    schedule="0 6 * * *",
    start_date=datetime(2026, 1, 1),
    deadline=DeadlineAlert(
        reference=DeadlineReference.DAGRUN_LOGICAL_DATE,  # scheduled start of the run
        interval=timedelta(hours=1),                      # must finish by 07:00
        callback=AsyncCallback(                           # runs in the triggerer
            SlackWebhookNotifier,
            kwargs={"text": "Dag {{ dag_run.dag_id }} missed its deadline at {{ deadline.deadline_time }}"},
        ),
    ),
)
def daily_orders():
    @task
    def load() -> None: ...

    load()


daily_orders()
```

## Quick Reference

| Piece | Options | Since |
|-------|---------|-------|
| `reference` | `DeadlineReference.DAGRUN_QUEUED_AT`, `DAGRUN_LOGICAL_DATE`, `FIXED_DATETIME(dt)` | 3.1 |
| `reference` | `DeadlineReference.AVERAGE_RUNTIME(max_runs=10, min_runs=...)` (no deadline until enough history) | 3.2 |
| `reference` | Custom: subclass `BaseDeadlineReference` + `@deadline_reference`, implement `_evaluate_with()` | 3.2 (in `airflow.sdk` since 3.3) |
| `interval` | `timedelta` (negative = alert *before* a `FIXED_DATETIME`); dynamic `VariableInterval` | 3.1 / 3.3 |
| `callback` | `AsyncCallback(callable_or_notifier, kwargs={...})`: runs in the **triggerer** | 3.1 |
| `callback` | `SyncCallback(callable, kwargs={...}, executor=...)`: runs on an **executor** like a top-priority task | 3.2 |
| `deadline=` | One `DeadlineAlert` or a **list** of them (tiered warn/escalate) | list: 3.2 |
| Import path | `from airflow.sdk import DeadlineAlert, DeadlineReference, AsyncCallback, SyncCallback` | 3.2 |

| Rule | Detail |
|------|--------|
| Callback context | Airflow injects a `context` kwarg; accept `**kwargs` or a named `context` param. `context` is reserved in `kwargs` (`ValueError` at parse time) |
| Templating | Kwargs are **not** Jinja-rendered; Notifiers render their own templates (`dag_run`, `deadline` available) |
| Custom async callable | Top-level function in a module importable by the triggerer (e.g. plugins folder); restart the triggerer after changes |
| Custom sync callable | Must be importable on the worker that runs it |
| UI | 3.3 adds a **Deadlines** page under Browse |
| Local testing | Observed 2026-10-17 on 3.3.2: `dag.test()` raises `AttributeError: 'dict' object has no attribute 'hex'` for any Dag with `deadline=` (the official example too); test deadlines in a deployment |

## Common Mistakes

### Wrong

```python
# Migration only: removed in Airflow 3.0, fails on 3.x
@dag(schedule="@daily", sla_miss_callback=notify)
def pipeline():
    @task(sla=timedelta(hours=1))
    def extract(): ...
```

### Correct

```python
from datetime import datetime, timedelta

from airflow.providers.slack.notifications.slack_webhook import SlackWebhookNotifier
from airflow.sdk import AsyncCallback, DeadlineAlert, DeadlineReference, SyncCallback, dag  # 3.2+

tiered = [
    DeadlineAlert(  # warn 30 min after queueing
        reference=DeadlineReference.DAGRUN_QUEUED_AT,
        interval=timedelta(minutes=30),
        callback=AsyncCallback(SlackWebhookNotifier, kwargs={"text": "{{ dag_run.dag_id }} is late"}),
    ),
    DeadlineAlert(  # escalate after 60 min (3.2+: list + SyncCallback)
        reference=DeadlineReference.DAGRUN_QUEUED_AT,
        interval=timedelta(minutes=60),
        callback=SyncCallback("my_plugins.escalation.page_oncall", kwargs={"severity": "high"}),
    ),
]

@dag(schedule="@daily", start_date=datetime(2026, 1, 1), deadline=tiered)
def pipeline(): ...


pipeline()
```

## Choosing a Reference

| Goal | Reference |
|------|-----------|
| Closest to old SLA ("done N after scheduled time") | `DAGRUN_LOGICAL_DATE` |
| Alert on runs stuck in the queue (resource limits) | `DAGRUN_QUEUED_AT` |
| Hard business cut-off ("report by 09:00") | `FIXED_DATETIME(...)` with a negative interval for an early warning |
| Detect "slower than usual" | `AVERAGE_RUNTIME(...)` (3.2+) |

## Related

- [error-handling](../patterns/error-handling.md)
- [dag-design](dag-design.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/howto/deadline-alerts.html
- https://airflow.apache.org/docs/apache-airflow/stable/howto/sla-to-deadlines.html
- https://airflow.apache.org/docs/apache-airflow/3.1.8/howto/deadline-alerts.html (3.1 import path)
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (3.1.0, 3.2.0, 3.3.0)
- https://airflow.apache.org/docs/task-sdk/stable/api.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
