# Error Handling

> **Purpose**: Retries and backoff, exception-aware retry policies (3.3+), notifier-based alerting, deadline alerts, failure routing, and resumable tasks on Airflow 3.x
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Transient failures (rate limits, timeouts) need backoff, while permanent ones (bad credentials) should fail fast
- Failures must reach Slack, email or PagerDuty with run context
- A run must finish by a deadline (the SLA feature was removed in 3.0, so use Deadline Alerts, 3.1+)
- Long external jobs must survive a worker crash without being submitted twice

## Implementation

```python
from __future__ import annotations

from datetime import datetime, timedelta

from airflow.providers.pagerduty.notifications.pagerduty import PagerdutyNotifier
from airflow.providers.slack.notifications.slack_webhook import SlackWebhookNotifier
from airflow.providers.smtp.notifications.smtp import SmtpNotifier
from airflow.providers.standard.operators.trigger_dagrun import TriggerDagRunOperator
from airflow.sdk import (
    AsyncCallback, DeadlineAlert, DeadlineReference, ExceptionRetryPolicy,
    RetryAction, RetryRule, TriggerRule, dag, task,
)

# --- 1. Notifiers: templated, reusable callbacks (no hand-built operators) ---
slack_on_failure = SlackWebhookNotifier(
    slack_webhook_conn_id="slack_data_alerts",
    text=":red_circle: {{ dag.dag_id }}.{{ ti.task_id }} failed (try {{ ti.try_number }}), run {{ run_id }}",
)
email_on_failure = SmtpNotifier(
    to="data-oncall@example.com",
    subject="[Airflow] {{ dag.dag_id }} failed",
    html_content="Task {{ ti.task_id }} failed in run {{ run_id }}.",
    smtp_conn_id="smtp_default",
)
page_on_failure = PagerdutyNotifier(
    summary="{{ dag.dag_id }}.{{ ti.task_id }} failed",
    severity="critical",
    pagerduty_events_conn_id="pagerduty_events_default",
)

# --- 2. Baseline retries for every task ---
DEFAULT_ARGS = {
    "retries": 4,
    "retry_delay": timedelta(minutes=2),
    "retry_exponential_backoff": 2.0,   # 3.2+ numeric multiplier; use True on 3.0/3.1
    "max_retry_delay": timedelta(minutes=30),
    "execution_timeout": timedelta(hours=1),
    "on_failure_callback": [slack_on_failure, email_on_failure],
}

# --- 3. Retry policy (3.3+): decide per exception type ---
API_RETRY_POLICY = ExceptionRetryPolicy(
    rules=[
        RetryRule(exception="requests.exceptions.HTTPError", action=RetryAction.RETRY,
                  retry_delay=timedelta(minutes=5), reason="Rate limit, back off"),
        RetryRule(exception=PermissionError, action=RetryAction.FAIL,
                  reason="Credential problem, retrying will not help"),
    ],
)


@dag(
    dag_id="production_pipeline",
    schedule="0 6 * * *",
    start_date=datetime(2026, 1, 1),
    default_args=DEFAULT_ARGS,
    dagrun_timeout=timedelta(hours=3),
    # --- 4. Deadline Alert (3.1+, experimental): not done 1h after the scheduled time ---
    deadline=DeadlineAlert(
        reference=DeadlineReference.DAGRUN_LOGICAL_DATE,
        interval=timedelta(hours=1),
        callback=AsyncCallback(SlackWebhookNotifier, kwargs={
            "text": "{{ dag_run.dag_id }} missed its deadline ({{ deadline.deadline_time }})",
        }),
    ),
    tags=["production"],
)
def production_pipeline():
    @task(retry_policy=API_RETRY_POLICY)
    def extract() -> str:
        import requests
        resp = requests.get("https://api.example.com/orders", timeout=30)
        resp.raise_for_status()
        return resp.text

    @task(on_failure_callback=[slack_on_failure, page_on_failure])  # critical step: also page
    def transform(raw: str) -> int:
        return len(raw)

    # --- 5. Failure routing: runs only if an upstream task failed ---
    notify_central = TriggerDagRunOperator(
        task_id="notify_failure_dag",
        trigger_dag_id="failure_notifier",
        conf={"source_dag": "{{ dag.dag_id }}", "run_id": "{{ run_id }}", "severity": "P2"},
        trigger_rule=TriggerRule.ONE_FAILED,
    )

    transform(extract()) >> notify_central


production_pipeline()


# --- 6. Resumable external job (3.3+): task state store survives retries/crashes ---
# submit_job / wait_for_job are placeholders for your own client calls
@task(retries=3)
def run_external_job(**context) -> str:
    from airflow.sdk import NEVER_EXPIRE
    store = context["task_state_store"]
    job_id = store.get("job_id")
    if job_id is None:                       # first attempt: submit once
        job_id = submit_job()                # your client call
        store.set("job_id", job_id, retention=NEVER_EXPIRE)
    return wait_for_job(job_id)              # retries re-attach instead of resubmitting
```

## Configuration

| Setting | Where | Default / Notes | Since |
|---------|-------|-----------------|-------|
| `retries`, `retry_delay` | task / `default_args` | Base retry count and delay | all |
| `retry_exponential_backoff` | task / `default_args` | Multiplier (`2.0`); `0` disables; booleans still accepted (`True` = 2.0) | numeric: 3.2 |
| `max_retry_delay` | task / `default_args` | Caps the backoff | all |
| `retry_policy` | task / operator | `ExceptionRetryPolicy(rules=[RetryRule(...)])`; first matching rule wins; `RetryAction.RETRY` / `FAIL`; evaluated in the worker | 3.3 |
| `execution_timeout` | task | Fails the attempt when exceeded | all |
| `dagrun_timeout` | Dag | Fails the whole run when exceeded | all |
| `on_failure_callback` / `on_retry_callback` / `on_success_callback` | task or Dag | Callable, Notifier, or list of them | all |
| `deadline` | Dag | `DeadlineAlert` or list; see [deadline-alerts](../concepts/deadline-alerts.md) | 3.1 (list: 3.2) |
| `context["task_state_store"]` | task | `get/set/delete/clear`; `retention=timedelta(...)` or `NEVER_EXPIRE` | 3.3 |
| `trigger_rule=TriggerRule.ONE_FAILED` | task | Failure-only branch; does not wait for all upstreams | all |

## Example Usage

```python
# Central notifier Dag that receives conf from TriggerDagRunOperator
from airflow.providers.pagerduty.hooks.pagerduty_events import PagerdutyEventsHook
from airflow.providers.smtp.hooks.smtp import SmtpHook
from airflow.sdk import dag, task


@dag(dag_id="failure_notifier", schedule=None, tags=["alerting"])
def failure_notifier():
    @task
    def route(**context) -> str:
        conf = context["dag_run"].conf or {}
        summary = f"[{conf.get('severity', 'P3')}] {conf.get('source_dag')} run {conf.get('run_id')} failed"
        if conf.get("severity") in ("P1", "P2"):
            PagerdutyEventsHook(pagerduty_events_conn_id="pagerduty_events_default").send_event(
                summary=summary, severity="error", source="airflow",
            )
        with SmtpHook(smtp_conn_id="smtp_default") as hook:
            hook.send_email_smtp(to="data-oncall@example.com", subject=summary, html_content=summary)
        return summary

    route()


failure_notifier()
```

### Wrong (migration only: removed or deprecated in Airflow 3.x)

```python
from airflow.utils.email import send_email          # legacy helper, slated for deprecation
@dag(sla_miss_callback=notify)                      # SLA removed in 3.0
def old():
    @task(sla=timedelta(hours=1))                   # removed in 3.0
    def t(**context):
        context["execution_date"]                   # key removed in 3.0 -> context["logical_date"]
```

`logical_date` can be `None` for runs triggered through the REST API without one, so prefer `run_id`
or `data_interval_*` when you only need a unique run identifier.

## See Also

- [deadline-alerts](../concepts/deadline-alerts.md)
- [dag-design](../concepts/dag-design.md)
- [task-dependencies](../concepts/task-dependencies.md)
- [sensors-triggers](sensors-triggers.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html (Retry Policies)
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (3.2.0 backoff multiplier; 3.3.0 retry policies, state store)
- https://airflow.apache.org/docs/apache-airflow/stable/howto/deadline-alerts.html
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/task-state-store.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
- https://airflow.apache.org/docs/apache-airflow-providers-smtp/stable/notifications/smtp_notifier_howto_guide.html
- https://airflow.apache.org/docs/apache-airflow-providers-smtp/stable/connections/smtp.html
- https://airflow.apache.org/docs/apache-airflow-providers-slack/stable/_api/airflow/providers/slack/notifications/slack_webhook/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-pagerduty/stable/_api/airflow/providers/pagerduty/notifications/pagerduty/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/trigger_dagrun/index.html
- https://airflow.apache.org/docs/task-sdk/stable/api.html
