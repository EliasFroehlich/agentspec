# dbt-databricks Adapter

> **Purpose**: Databricks-specific dbt configs — liquid clustering, incremental strategies (`replace_where`, `microbatch`), tags, Python model submission, constraints behavior and the `metric_view` materialization
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

`dbt-databricks` is the Databricks adapter (current **1.12.7, 2026-10-12**; Databricks is also GA on dbt
v2). It materializes models as Unity Catalog Delta tables/views by default and adds Databricks-only configs:
liquid clustering, a `replace_where` incremental strategy, governed tags, serverless Python models and —
since **1.12.0 (2026-05-18)** — Unity Catalog **metric views** as a dbt materialization. Incremental models
require DBR 12.2 LTS+ since adapter 1.11 (`INSERT BY NAME`).

## The Concept

```sql
-- models/marts/fct_orders.sql
-- merge is the dbt-databricks default (Delta/Hudi only); liquid_clustered_by (1.6.2+) runs OPTIMIZE
-- after each run; databricks_tags are applied with ALTER and never removed by dbt
{{ config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id',
    liquid_clustered_by=['order_date'],
    databricks_tags={'domain': 'sales'},
    tblproperties={'delta.enableChangeDataFeed': 'true'}
) }}
select order_id, customer_id, order_date, amount, updated_at
from {{ ref('stg_orders') }}
{% if is_incremental() %}
where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

```sql
{# models/marts/orders_metrics.sql — UC metric view (dbt-databricks 1.12.0+; YAML 1.1 needs DBR 17.3+).
   The model body is YAML: use Jinja comments here, not SQL comments. #}
{{ config(materialized='metric_view') }}
version: 1.1
source: "{{ ref('fct_orders') }}"
filter: status = 'completed'
dimensions:
  - name: order_date
    expr: order_date
measures:
  - name: total_orders
    expr: count(1)
  - name: total_revenue
    expr: sum(amount)
    synonyms: [revenue, sales]
```

dbt runs `CREATE OR REPLACE VIEW … WITH METRICS LANGUAGE YAML`; set `view_update_via_alter: true` to apply
changes with `ALTER VIEW … AS` instead (keeps grants). Query with `MEASURE(total_revenue)`.

## Quick Reference

| Config | Values / notes | Since |
|--------|----------------|-------|
| `incremental_strategy` | `merge` (default), `append`, `insert_overwrite`, `replace_where`, `delete+insert`, `microbatch` | — |
| `replace_where` | Delta + DBR 12.0+; predicate in `incremental_predicates`; inserts **by column order**, not name | — |
| `delete+insert` | Delta only, needs `unique_key`, DBR 12.2 LTS+ (uses `INSERT INTO … REPLACE ON` on DBR 17.1+) | 1.11 |
| `microbatch` | Implemented with `replace_where` predicates built from `event_time` | 1.9 (dbt) |
| `insert_overwrite` | Without `partition_by`/`liquid_clustered_by` it replaces the **whole table** | — |
| `liquid_clustered_by` | List of columns; skip the post-run OPTIMIZE globally with `DATABRICKS_SKIP_OPTIMIZE=true` or `--vars "{'databricks_skip_optimize': true}"` | 1.6.2 |
| `auto_liquid_cluster: true` | Databricks picks clustering keys; do not combine with `liquid_clustered_by` | 1.10.0 |
| `skip_optimize: true` | Per model/folder: skip OPTIMIZE but keep clustering in the table definition (e.g. when predictive optimization runs OPTIMIZE) | 1.12.2 |
| `databricks_tags` | Table tags; column tags 1.10.4+; additive merge across config levels and key-only tags in 1.12 | — |
| `tblproperties` | Dict passed as-is; changing it on an incremental model needs `--full-refresh` | — |
| `file_format` / `table_format` | `delta` default; `table_format: iceberg` (1.9+) requires `file_format: delta` | — |
| `submission_method` (Python models) | `all_purpose_cluster` (default), `job_cluster`, `serverless_cluster`, `workflow_job`; extra settings in `python_job_config` | 1.9 |
| `grants` | `grants: {select: ['analysts']}`; works on SQL warehouses; `READ_METADATA`/`USAGE` need post-hooks | — |

| Constraint (model contract) | Definable | Enforced on Databricks |
|-----------------------------|-----------|------------------------|
| `not_null`, `check` | Yes | Yes — applied with `ALTER` **after** the table is built; a failing constraint can leave the new table in place |
| `primary_key`, `foreign_key` | Yes | No (informational) |
| `unique` | No | No |

Contracts on Databricks check column **names and order**, not types. With `use_materialization_v2`
(1.12.1+), column constraints are applied only when `contract: {enforced: true}`.

## Common Mistakes

### Wrong

```sql
{{ config(materialized='incremental', incremental_strategy='insert_overwrite') }}
-- no partition_by / liquid_clustered_by: every run overwrites the entire table
select * from {{ ref('stg_events') }} where event_date = current_date()
```

### Correct

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='replace_where',
    incremental_predicates=["event_date = current_date()"],
    liquid_clustered_by=['event_date']
) }}
select event_id, event_date, payload from {{ ref('stg_events') }}   -- same column order as target
where event_date = current_date()
```

## Related

- [incremental-strategies](incremental-strategies.md)
- [semantic-layer](../patterns/semantic-layer.md) — MetricFlow vs UC metric views
- [Metric views KB](../../metric-views/index.md)
- [testing-framework](testing-framework.md)

## Sources

- https://docs.getdbt.com/reference/resource-configs/databricks-configs
- https://github.com/databricks/dbt-databricks/releases
- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view_dbt
- https://docs.getdbt.com/reference/resource-properties/constraints
- https://docs.getdbt.com/reference/resource-configs/grants
- https://docs.getdbt.com/docs/dbt/dbt-availability
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/
