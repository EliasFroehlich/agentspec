# DAG Factory

> **Purpose**: Parameterized Dag generation from YAML config with TaskGroups; Airflow 3.x imports, auto-registration and parse-time optimization
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Many similar pipelines that differ only by source, target or schedule
- Standardizing Dag structure, retries and alerting across teams
- Reducing boilerplate in projects with dozens of Dags

## Implementation

```python
from __future__ import annotations

from datetime import datetime, timedelta
from pathlib import Path

import yaml

from airflow.providers.slack.notifications.slack_webhook import SlackWebhookNotifier
from airflow.sdk import BaseHook, TaskGroup, dag, get_parsing_context, task

# --- YAML CONFIG (configs/pipeline_orders.yaml, next to this file) ---
# pipeline_name: orders
# schedule: "0 6 * * *"
# owner: data-team
# retries: 3
# source: {type: postgres, conn_id: source_postgres, tables: [raw_orders, raw_order_items]}
# target: {type: snowflake, conn_id: target_snowflake, schema: staging}

CONFIG_DIR = Path(__file__).parent / "configs"   # relative to the Dag file, not the CWD


def load_pipeline_configs(config_dir: Path = CONFIG_DIR) -> list[dict]:
    """Load all YAML pipeline configs (cheap: runs on every parse)."""
    return [yaml.safe_load(p.read_text()) for p in sorted(config_dir.glob("pipeline_*.yaml"))]


def create_dag(config: dict):
    """Factory: build one Dag from a config dict."""
    default_args = {
        "owner": config.get("owner", "data-team"),
        "retries": config.get("retries", 2),
        "retry_delay": timedelta(minutes=5),
        "execution_timeout": timedelta(hours=2),
        "on_failure_callback": SlackWebhookNotifier(
            slack_webhook_conn_id="slack_data_alerts",
            text="{{ dag.dag_id }}.{{ ti.task_id }} failed (run {{ run_id }})",
        ),
    }

    @dag(
        dag_id=f"pipeline_{config['pipeline_name']}",
        default_args=default_args,
        schedule=config.get("schedule"),
        start_date=datetime(2026, 1, 1),
        catchup=False,
        tags=["factory", config["pipeline_name"]],
    )
    def pipeline():
        @task
        def extract(table: str, conn_id: str) -> dict:
            hook = BaseHook.get_hook(conn_id)        # DbApiHook subclass for SQL sources
            count = hook.get_records(f"SELECT COUNT(*) FROM {table}")[0][0]
            return {"table": table, "row_count": count}

        @task
        def load(extract_result: dict, target_conn: str, target_schema: str) -> str:
            return f"Loaded {extract_result['table']} ({extract_result['row_count']} rows) to {target_schema}"

        @task
        def quality_check(load_result: str) -> None:
            if "Loaded" not in load_result:
                raise ValueError(f"Load failed: {load_result}")

        with TaskGroup("ingest"):
            for table in config["source"]["tables"]:
                with TaskGroup(f"process_{table}"):
                    loaded = load(
                        extract_result=extract(table=table, conn_id=config["source"]["conn_id"]),
                        target_conn=config["target"]["conn_id"],
                        target_schema=config["target"]["schema"],
                    )
                    quality_check(load_result=loaded)

    return pipeline()


# --- Generate Dags. @dag-created Dags auto-register (2.4+): no globals() needed. ---
# When a worker runs one task, build only that Dag (get_parsing_context, 2.4+).
_current = get_parsing_context().dag_id
for cfg in load_pipeline_configs():
    if _current is not None and _current != f"pipeline_{cfg['pipeline_name']}":
        continue
    create_dag(cfg)
```

## Configuration

| Parameter | Location | Purpose |
|-----------|----------|---------|
| `pipeline_name` | YAML | Unique Dag identifier suffix |
| `schedule` | YAML | Cron string (3.x default timetable: `CronTriggerTimetable`) or `null` |
| `source.tables` | YAML | Tables to ingest (one TaskGroup each) |
| `retries` | YAML / `default_args` | Per-pipeline override |
| `auto_register` | `@dag(...)` | Default `True`; set `False` to stop automatic registration |
| `[dag_processor] min_file_process_interval` | airflow.cfg | Default 30 s: how often each file (and its YAML loop) is re-parsed |
| `[dag_processor] refresh_interval` | airflow.cfg | Default 300 s: how often new files in the bundle are discovered |

## Example Usage

```yaml
# configs/pipeline_payments.yaml
pipeline_name: payments
schedule: "30 7 * * *"
owner: finance-team
retries: 5
source:
  type: postgres
  conn_id: payments_db
  tables: [transactions, refunds, chargebacks]
target:
  type: snowflake
  conn_id: warehouse
  schema: staging_finance
```

Keep the factory cheap at parse time: no network or database calls outside tasks, and keep the config
in files that ship with the Dag bundle. When a table list must come from a database at runtime, map over it
with `expand()` inside a single Dag instead (see [dynamic-task-mapping](dynamic-task-mapping.md)).

## See Also

- [dag-design](../concepts/dag-design.md)
- [dynamic-task-mapping](dynamic-task-mapping.md)
- [error-handling](error-handling.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/howto/dynamic-dag-generation.html
- https://airflow.apache.org/docs/task-sdk/stable/api.html (`BaseHook.get_hook`, `TaskGroup`, `get_parsing_context`)
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
- https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html (`[dag_processor]`)
- https://airflow.apache.org/docs/apache-airflow-providers-slack/stable/_api/airflow/providers/slack/notifications/slack_webhook/index.html
