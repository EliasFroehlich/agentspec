# Sensors & Triggers

> **Purpose**: Cross-Dag and external-event orchestration on Airflow 3.x: asset-driven schedules, event-driven `AssetWatcher`s, partitioned assets (3.2+), deferrable sensors, ExternalTaskSensor and TriggerDagRunOperator
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- A Dag must run when upstream data is updated, not on a clock: use assets
- An external system emits events (queue message, file event): use `AssetWatcher` + an event trigger
- Downstream work is per partition (hour, day, region): use partitioned assets (3.2+)
- You must wait for a file, a job or another Dag's task: use a **deferrable** sensor

## Implementation

```python
from __future__ import annotations

from datetime import datetime, timedelta

from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.common.messaging.triggers.msg_queue import MessageQueueTrigger
from airflow.providers.standard.operators.trigger_dagrun import TriggerDagRunOperator
from airflow.providers.standard.sensors.external_task import ExternalTaskSensor
from airflow.sdk import Asset, AssetWatcher, dag, task

# --- 1. Asset-driven scheduling (3.0+; "Dataset" before 3.0) ---
orders = Asset(name="orders", uri="s3://my-bucket/staging/orders/")
payments = Asset(name="payments", uri="s3://my-bucket/staging/payments/")


@dag(schedule="0 6 * * *", start_date=datetime(2026, 1, 1))
def producer_orders():
    @task(outlets=[orders])          # success emits an asset event
    def ingest_orders() -> None: ...

    ingest_orders()


@dag(schedule=(orders & payments), start_date=datetime(2026, 1, 1))  # AND; use | for OR
def consumer_analytics():
    @task
    def build_analytics() -> None: ...

    build_analytics()


# --- 2. Event-driven scheduling with an AssetWatcher (3.0+) ---
# Needs apache-airflow-providers-common-messaging[apache.kafka] (or [amazon] for SQS)
orders_topic = Asset(
    "orders_topic",
    watchers=[AssetWatcher(name="orders_kafka", trigger=MessageQueueTrigger(scheme="kafka", topics=["orders"]))],
)


@dag(schedule=[orders_topic])
def on_order_message():
    @task
    def handle(triggering_asset_events) -> None:
        for event in triggering_asset_events[orders_topic]:
            print(event.extra["payload"])    # message payload from the trigger

    handle()


# --- 3. Deferrable sensors (worker slot freed while waiting; needs a triggerer) ---
@dag(schedule="0 7 * * *", start_date=datetime(2026, 1, 1))
def file_watcher():
    wait_for_file = S3KeySensor(
        task_id="wait_for_landing_file",
        bucket_name="my-bucket",
        bucket_key="landing/orders/{{ ds }}/data.parquet",
        aws_conn_id="aws_default",
        deferrable=True,
        timeout=3600,
    )
    wait_for_upstream = ExternalTaskSensor(
        task_id="wait_for_staging_dag",
        external_dag_id="staging_pipeline",
        external_task_id="final_quality_check",
        allowed_states=["success"],
        execution_delta=timedelta(hours=1),   # upstream runs 1h earlier
        deferrable=True,
        timeout=7200,
    )
    trigger_downstream = TriggerDagRunOperator(
        task_id="trigger_transform",
        trigger_dag_id="transform_pipeline",
        conf={"source": "file_watcher", "date": "{{ ds }}"},
        wait_for_completion=False,
    )

    @task
    def process_file() -> None: ...

    wait_for_file >> wait_for_upstream >> process_file() >> trigger_downstream


producer_orders()
consumer_analytics()
on_order_message()
file_watcher()
```

## Partitioned Assets (3.2+, rollups 3.3+)

```python
from airflow.sdk import (
    DAG, Asset, CronPartitionTimetable, DayWindow, PartitionedAssetTimetable,
    RollupMapper, StartOfHourMapper, task,
)

hourly_clicks = Asset(name="hourly_clicks", uri="s3://my-bucket/raw/clicks/")

with DAG(dag_id="ingest_hourly_clicks", schedule=CronPartitionTimetable("0 * * * *", timezone="UTC")):
    @task(outlets=[hourly_clicks])        # event carries partition_key, e.g. 2026-03-10T09:00:00
    def ingest() -> None: ...
    ingest()

with DAG(
    dag_id="daily_click_rollup",
    schedule=PartitionedAssetTimetable(
        assets=hourly_clicks,
        default_partition_mapper=RollupMapper(upstream_mapper=StartOfHourMapper(), window=DayWindow()),
    ),
    catchup=False,
):
    @task
    def summarize(dag_run=None) -> None:
        print(dag_run.partition_key)      # the day, e.g. "2026-03-10"
    summarize()
```

- 3.2: `CronPartitionTimetable` (producer), `PartitionedAssetTimetable` (consumer) and one-to-one mappers
  (`IdentityMapper` default, `StartOfHourMapper`/`StartOfDayMapper`/..., `ProductMapper`, `AllowedKeyMapper`).
- 3.3: `RollupMapper` + windows (`DayWindow`, `WeekWindow`, ...; wait policies `WaitForAll` / `MinimumCount(n)`),
  `FanOutMapper`, `FixedKeyMapper` + `SegmentWindow`, and `PartitionedAtRuntime` (key set by the task via
  `outlet_events[...].add_partitions(...)`).
- Partitioned events trigger only partition-aware consumers; events without a `partition_key` never trigger them.
  Use a UTC upstream mapper for rollups that cross a DST change.

## Configuration

| Pattern | Worker usage | Best for | Since |
|---------|-------------|----------|-------|
| Sensor `mode="poke"` | Holds a worker slot | Very short waits | all |
| Sensor `mode="reschedule"` | Releases slot between pokes | Medium waits | all |
| `deferrable=True` | Runs in the triggerer (async) | Long waits; set `[operators] default_deferrable = True` to make it the default (default `False`) | 2.2+ |
| `schedule=[a1, a2]` / `(a1 & a2)` / `(a1 \| a2)` | No sensor at all | Cross-Dag data dependencies | 3.0 |
| `AssetOrTimeSchedule` | No sensor | Asset events **or** a timetable | 3.0 |
| `AssetWatcher` + `BaseEventTrigger` trigger | Triggerer | External events (Kafka, SQS, ...) | 3.0 |
| REST API asset events | None | External systems push events | 2.9+ |
| `CronPartitionTimetable` / `PartitionedAssetTimetable` | No sensor | Partition-aware scheduling | 3.2 |

## Example Usage

```python
# Only BaseEventTrigger subclasses can back an AssetWatcher. "Wait until X exists"
# triggers (e.g. S3KeyTrigger) would re-fire forever once the condition is true.
from airflow.triggers.base import BaseEventTrigger  # base class for custom event triggers
```

### Wrong (migration only: pre-3.0 paths)

```python
from airflow.datasets import Dataset                          # not importable in 3.3.2 -> airflow.sdk.Asset
from airflow.sensors.external_task import ExternalTaskSensor  # -> airflow.providers.standard.sensors.external_task
from airflow.operators.trigger_dagrun import TriggerDagRunOperator  # -> providers.standard.operators.trigger_dagrun
```

## See Also

- [task-dependencies](../concepts/task-dependencies.md)
- [error-handling](error-handling.md)
- [dynamic-task-mapping](dynamic-task-mapping.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/assets.html
- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html
- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/event-scheduling.html
- https://airflow.apache.org/docs/apache-airflow-providers-common-messaging/stable/triggers.html
- https://airflow.apache.org/docs/apache-airflow-providers-common-messaging/stable/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/sensors/external_task/index.html
- https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/trigger_dagrun/index.html
- https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html (`[operators] default_deferrable`)
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (3.2.0, 3.3.0 asset partitioning)
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
