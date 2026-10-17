# Orchestrator Comparison

> **Purpose**: Airflow 3.3 vs Dagster 1.13 vs Prefect 3.8: decision matrix, trade-offs and the 2026 market change
> **Confidence**: MEDIUM (product facts verified; "best for" rows are guidance, not benchmarks)
> **MCP Validated**: 2026-10-17

## Overview

The three main Python orchestrators start from different models. **Airflow 3.x** combines time-based Dags
with assets (asset-driven schedules, event-driven `AssetWatcher`s, partitioned assets since 3.2).
**Dagster** is asset-centric: you declare the data that should exist and the system works out how to produce
it. **Prefect** is execution-centric: you decorate Python functions and get dynamic, retryable flows.
**Market change:** on 2026-07-13 Prefect announced it is acquiring Dagster Labs. Dagster keeps its name and
open-source license, and Dagster+ remains a supported commercial offering. The companies say there are no
plans to merge Dagster into Prefect and no required migration.

## The Concept

```python
# Same pipeline in three orchestrators

# --- AIRFLOW 3.x ---
from datetime import datetime
from airflow.sdk import dag, task

@dag(schedule="@daily", start_date=datetime(2026, 1, 1))
def orders_pipeline():
    @task
    def extract() -> list[dict]: ...
    @task
    def transform(rows: list[dict]) -> None: ...
    transform(extract())

orders_pipeline()

# --- DAGSTER 1.13 ---
import dagster as dg

@dg.asset
def raw_orders() -> list[dict]:
    return extract_from_api()

@dg.asset
def clean_orders(raw_orders: list[dict]) -> list[dict]:   # dependency inferred from the parameter
    return transform(raw_orders)

# --- PREFECT 3.x ---
from prefect import flow, task as ptask

@ptask(retries=2)
def extract() -> list[dict]: ...

@ptask
def transform(rows: list[dict]) -> None: ...

@flow(name="orders-pipeline")
def orders_flow():
    transform(extract())
```

## Quick Reference

| Dimension | Airflow 3.3 | Dagster 1.13 | Prefect 3.8 |
|-----------|-------------|--------------|-------------|
| **Mental model** | Dags of tasks + assets | Software-defined assets | Decorated Python flows/tasks |
| **Scheduling** | Cron/timetables, asset AND/OR, `AssetWatcher` events, partitions (3.2+) | Schedules, sensors, Declarative Automation (`AutomationCondition`) | Cron / Interval / RRule schedules, event-based automations |
| **Partitions** | `CronPartitionTimetable`, `PartitionedAssetTimetable`, rollups (3.3) | Partitions definitions (e.g. `DailyPartitionsDefinition`); partitioned asset checks (1.13) | Model with flow parameters |
| **Dynamic work** | `expand()` / `expand_kwargs()` | Dynamic partitions, `@multi_asset` | `.map()` on tasks |
| **Testing** | `dag.test()`, plain-function tests of `@task` code | `materialize()` in pytest | Call flows/tasks directly |
| **Alerting on lateness** | Deadline Alerts (3.1+, experimental) | Freshness policies + alerts (preview) | Automations (incl. "absence of an expected event" triggers) |
| **Developer tooling** | Task SDK; `airflowctl` for remote ops | `dg` CLI, Components (20+ more in 1.13) | `prefect` CLI |
| **Managed offerings** | Astronomer, Amazon MWAA, Google Cloud Composer | Dagster+ | Prefect Cloud |
| **dbt integration** | Provider operators (dbt Cloud) or `@task.bash` | `dagster-dbt` (asset-level) | `prefect-dbt` |
| **Python** | 3.10–3.14 | See Dagster release notes | 3.8.8 is the last release supporting 3.10 |

## Common Mistakes

### Wrong

```text
Choosing by popularity alone, or citing download counts with no source and date.
Assuming Dagster is going away because of the Prefect acquisition (the companies say it is not).
Calling Dagster's commercial product "Dagster Cloud" (it is Dagster+).
```

### Correct

```text
Decision framework:
1. Existing Airflow estate, many integrations, regulated audit needs -> Airflow 3.3
   (DAG versioning, deadline alerts, retry policies, multi-executor, HITL approvals)
2. Data-asset thinking, lineage and partition-level quality checks first -> Dagster
3. Python-first teams running dynamic, failure-prone workflows with little infra -> Prefect
4. Hybrid / remote execution (workers behind firewalls) -> Airflow EdgeExecutor (edge3 provider)
5. Need both asset modelling and dynamic execution -> watch the combined Prefect + Dagster roadmap,
   but plan on the products staying separate (no merger planned as of 2026-10-17)
```

## Related

- [dag-design](dag-design.md)
- [dag-factory](../patterns/dag-factory.md)

## Sources

- https://www.prefect.io/prefect-acquires-dagster
- https://dagster.io/prefect
- https://dagster.io/dagster-customer-faq
- https://dagster.io/blog/dagster-1-13-octopuss-garden
- https://docs.dagster.io/guides/automate/declarative-automation
- https://docs.dagster.io/guides/observe/asset-freshness-policies
- https://docs.prefect.io/v3/concepts/automations
- https://docs.prefect.io/v3/concepts/schedules
- https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/debug.html (`dag.test()`)
- https://docs.prefect.io/v3/release-notes/oss/version-3-8
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/prerequisites.html
