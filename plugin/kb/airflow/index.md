# Airflow Knowledge Base

> **Purpose**: Orchestration with Apache Airflow 3.x (stable 3.3.2): Task SDK authoring, asset-driven and partitioned scheduling, deadline alerts, retry policies, and orchestrator selection
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/dag-design.md](concepts/dag-design.md) | Idempotency, atomicity, retries, timetables, `@dag`/`@asset` |
| [concepts/operators.md](concepts/operators.md) | TaskFlow, provider operators, deferrable, HITL, executors |
| [concepts/task-dependencies.md](concepts/task-dependencies.md) | Trigger rules, XCom (and its limits), asset conditions |
| [concepts/deadline-alerts.md](concepts/deadline-alerts.md) | Deadline Alerts (3.1+), the SLA replacement |
| [concepts/orchestrator-comparison.md](concepts/orchestrator-comparison.md) | Airflow 3.3 vs Dagster 1.13 vs Prefect 3.8 |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/dag-factory.md](patterns/dag-factory.md) | YAML-driven Dag generation, parse-time optimization |
| [patterns/sensors-triggers.md](patterns/sensors-triggers.md) | Assets, `AssetWatcher`, partitions, deferrable sensors |
| [patterns/dynamic-task-mapping.md](patterns/dynamic-task-mapping.md) | `expand()`, `partial()`, `expand_kwargs()`, `.map()/.zip()/.concat()` |
| [patterns/error-handling.md](patterns/error-handling.md) | Retries, retry policies, notifiers, deadlines, resumable tasks |

---

## Quick Reference

- [quick-reference.md](quick-reference.md): imports, versions, schedules, removed features, config, executors

---

## Key Concepts

| Concept | Description | Since |
|---------|-------------|-------|
| **Task SDK** | Author with `from airflow.sdk import ...`; tasks reach Airflow through the Task Execution API, not the metadata DB | 3.0 |
| **Assets** | `Asset`, `@asset`, `outlets=`, AND/OR schedules, `AssetWatcher` for external events | 3.0 |
| **Asset partitioning** | `CronPartitionTimetable`, `PartitionedAssetTimetable`, mappers; rollups/fan-out in 3.3 | 3.2 |
| **Deadline Alerts** | `deadline=DeadlineAlert(...)` replaces the removed SLA feature (experimental) | 3.1 |
| **Human-in-the-loop** | `HITLOperator`, `ApprovalOperator`, `HITLBranchOperator`, `HITLEntryOperator` | 3.1 |
| **Retry policies** | `retry_policy=ExceptionRetryPolicy(...)` per exception type | 3.3 |
| **Task/Asset State Store** | `context["task_state_store"]` for job IDs, checkpoints, watermarks | 3.3 |
| **Multi-executor / Edge** | `executor = A,B` per deployment; `EdgeExecutor` for remote sites | 2.10 / 3.0 |
| **Orchestrator choice** | Airflow vs Dagster vs Prefect (Prefect announced the Dagster Labs acquisition on 2026-07-13) | — |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/dag-design.md, concepts/operators.md |
| **Intermediate** | patterns/dynamic-task-mapping.md, patterns/sensors-triggers.md |
| **Advanced** | patterns/error-handling.md, concepts/deadline-alerts.md, concepts/orchestrator-comparison.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| airflow-specialist | All files | DAG development, scheduling, operations |
| pipeline-architect | concepts/orchestrator-comparison.md, patterns/* | Orchestration design, orchestrator selection |
| build-agent | patterns/error-handling.md | Pipeline verification |

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/prerequisites.html
- https://airflow.apache.org/docs/task-sdk/stable/api.html
- https://www.prefect.io/prefect-acquires-dagster
