# Dynamic Task Mapping

> **Purpose**: Runtime task generation with expand(), partial(), expand_kwargs(), mapped task groups and XComArg map/zip/concat; fan-out/fan-in patterns
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Number of tasks unknown until runtime (file lists, partition counts, API pages)
- Same operation applied to a variable-length collection
- Fan-out processing followed by aggregation (fan-in)
- Combining dynamic mapping with shared configuration via partial()

## Implementation

```python
from __future__ import annotations

from datetime import datetime

from airflow.sdk import TaskGroup, dag, task


@dag(
    dag_id="dynamic_task_mapping",
    schedule="0 8 * * *",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["dynamic-mapping"],
)
def dynamic_mapping():

    # --- 1. Basic expand() with a list ---
    @task
    def get_file_list() -> list[str]:
        """Runtime resolution: returns variable-length list."""
        return ["orders_2024.parquet", "returns_2024.parquet", "shipments_2024.parquet"]

    @task
    def process_file(file_path: str) -> dict:
        """Mapped once per item from upstream list."""
        row_count = len(file_path) * 1000  # placeholder
        return {"file": file_path, "rows": row_count}

    files = get_file_list()
    processed = process_file.expand(file_path=files)

    # --- 2. partial() for shared params ---
    @task
    def load_to_warehouse(record: dict, target_schema: str, conn_id: str) -> str:
        """partial() pins target_schema and conn_id; expand() iterates records."""
        return f"Loaded {record['file']} -> {target_schema} via {conn_id}"

    loaded = load_to_warehouse.partial(
        target_schema="staging",
        conn_id="snowflake_default",
    ).expand(record=processed)

    # --- 3. TaskGroup with dynamic mapping ---
    @task
    def get_sources() -> list[dict]:
        return [
            {"name": "postgres_orders", "conn": "pg_main", "table": "orders"},
            {"name": "postgres_returns", "conn": "pg_main", "table": "returns"},
            {"name": "mysql_inventory", "conn": "mysql_wh", "table": "stock"},
        ]

    @task
    def extract(source: dict) -> dict:
        return {"source": source["name"], "rows": 50_000}

    @task
    def validate(result: dict) -> dict:
        result["valid"] = result["rows"] > 0
        return result

    with TaskGroup("ingest_sources") as ingest_group:
        sources = get_sources()
        extracted = extract.expand(source=sources)
        validated = validate.expand(result=extracted)

    # --- 4. Fan-out / fan-in pattern ---
    @task
    def aggregate(results: list[str]) -> str:
        """Fan-in: collects all mapped outputs into a single list."""
        return f"Aggregated {len(results)} loads"

    summary = aggregate(results=loaded)

    # --- 5. expand_kwargs with pre-zipped pairs (expand(a=[..], b=[..]) would build the cross product) ---
    @task
    def merge_datasets(table: str, partition: str) -> str:
        return f"Merged {table}/{partition}"

    tables = ["orders", "returns"]
    partitions = ["2024-01", "2024-02"]

    merge_datasets.expand_kwargs(
        [{"table": t, "partition": p} for t, p in zip(tables, partitions)]
    )

    # Execution order
    validated >> summary


dynamic_mapping()
```

## Configuration

All features below are available in every Airflow 3.x release (they were introduced during 2.x).

| Feature | Notes |
|---------|-------|
| `expand()` | Creates one task instance per item at runtime; only lists and dicts can be expanded |
| `partial()` | Pins shared kwargs; `task_id`, `pool`, `queue` and most BaseOperator args must go here |
| `expand_kwargs()` | Expands over a list of kwargs dicts (zip-style multi-param mapping) |
| `@task_group` + `expand()` | Mapped task group; code in the group function cannot read the runtime value |
| `XComArg.map(fn)` | Plain-function transform applied as pre-processing of the downstream task; not a task, and not a predecessor of `expand()` |
| `XComArg.zip(...)` / `.concat(...)` | Combine upstream outputs before expanding (`concat` since 2.10) |
| `map_index_template` | Human-readable names for mapped instances in the UI |
| `[core] max_map_length` | Default 1024; an upstream that pushes a longer list fails |
| `max_active_tis_per_dag` | Caps how many mapped instances run at once |

## Example Usage

```python
# Minimal dynamic mapping: process S3 objects found at runtime
from datetime import datetime

from airflow.sdk import dag, task


def to_uri(key: str) -> str:            # plain function, NOT a @task
    return f"s3://my-bucket/{key}"


@dag(schedule="@daily", start_date=datetime(2026, 1, 1))
def process_partitions():
    @task
    def list_partitions() -> list[str]:
        from airflow.providers.amazon.aws.hooks.s3 import S3Hook
        return S3Hook(aws_conn_id="aws_default").list_keys(bucket_name="my-bucket", prefix="raw/orders/")

    @task
    def transform(partition_uri: str) -> str:
        return f"Transformed {partition_uri}"

    # .map() reshapes each item right before the downstream task expands over it
    transform.expand(partition_uri=list_partitions().map(to_uri))


process_partitions()
```

## See Also

- [dag-design](../concepts/dag-design.md)
- [task-dependencies](../concepts/task-dependencies.md)
- [dag-factory](../patterns/dag-factory.md)
- [error-handling](../patterns/error-handling.md)

## Sources

- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html
- https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html (`max_map_length`)
- https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (2.10.0 `concat`)
- https://airflow.apache.org/docs/task-sdk/stable/api.html
- https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html
