# Incremental Strategies

> **Purpose**: Choose the right incremental materialization strategy for your data shape
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Incremental models process only new or changed data instead of rebuilding entire tables. dbt supports five strategies: **append** (insert-only), **merge** (upsert), **delete+insert** (replace matching rows), **insert_overwrite** (replace partitions), and **microbatch** (v1.9+, time-partitioned batch processing). Adapters add their own — `dbt-databricks` adds **replace_where**. Which strategies exist, and which is the default, depends on the adapter (`merge` on dbt-databricks). Strategy selection depends on data mutability, volume, and platform capabilities.

## The Concept

```sql
-- Merge strategy: upsert rows by unique key
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        incremental_strategy='merge',
        on_schema_change='append_new_columns'
    )
}}

select
    order_id,
    customer_id,
    order_total,
    status,
    updated_at
from {{ ref('stg_orders') }}

{% if is_incremental() %}
    -- Only process rows newer than the latest in target
    where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

## Microbatch Strategy (v1.9+)

```sql
-- Microbatch: dbt runs separate queries for each time batch
-- No is_incremental() logic — dbt filters every upstream that has event_time configured
-- event_time (required): time column used to batch; begin (required): earliest data;
-- batch_size (required): hour | day | month | year; lookback (default 1): batches re-processed for late data
{{
    config(
        materialized='incremental',
        incremental_strategy='microbatch',
        event_time='event_timestamp',
        begin='2024-01-01',
        batch_size='day',
        lookback=1
    )
}}

select
    event_id,
    user_id,
    event_type,
    event_timestamp,
    properties
from {{ ref('stg_events') }}                  -- stg_events must also set event_time,
                                               -- or every batch scans it in full
```

Use `ref('dim_users').render()` to opt a ref out of batch filtering (for example, a small dimension).

Backfill a window with `dbt run --select fct_events --event-time-start "2026-01-01" --event-time-end "2026-02-01"`
(both flags together).

## Quick Reference

| Strategy | Mutability | Unique Key | Best Warehouse | Volume Threshold |
|----------|-----------|------------|---------------|-----------------|
| `append` | Immutable events | Not needed | Any | Any |
| `merge` | Mutable rows | Required | Snowflake, BigQuery, Databricks (default) | < 100M rows |
| `delete+insert` | Mutable rows | Required | Redshift, Postgres; Databricks 1.11+ | > 100M rows |
| `insert_overwrite` | Partition-level | Partition key | BigQuery, Spark/Databricks (needs `partition_by` or `liquid_clustered_by`) | Any (partition) |
| `replace_where` | Predicate-defined slice | Not needed (`incremental_predicates`) | Databricks only | Any |
| `microbatch` | Time-series | `event_time` col (model + upstreams) | Any (v1.9+); Databricks via `replace_where` | Any |

`unique_key` matters only for `merge` and `delete+insert`; `append`, `insert_overwrite`, `replace_where`
and `microbatch` ignore it.

### Microbatch vs Traditional Incremental

| Aspect | Traditional Incremental | Microbatch |
|--------|------------------------|------------|
| Query structure | Single SQL for all new data | Separate SQL per time batch |
| User responsibility | Write `is_incremental()` logic | No conditional logic needed |
| Backfill | `--full-refresh` (rebuilds all) | Automatic per-batch backfill |
| Late-arriving data | Manual lookback window | `lookback` parameter |
| Batch granularity | User-defined | `hour`, `day`, `month` or `year` |
| Retry on failure | Reruns all new data | Retries only the failed batch |

## Common Mistakes

### Wrong

```sql
-- No is_incremental() guard = full table scan every run
{{
    config(materialized='incremental', unique_key='event_id')
}}
select * from {{ ref('stg_events') }}
-- Missing WHERE clause for incremental filter!
```

### Correct

```sql
{{
    config(materialized='incremental', unique_key='event_id')
}}
select * from {{ ref('stg_events') }}
{% if is_incremental() %}
    where event_timestamp > (select max(event_timestamp) from {{ this }})
{% endif %}
```

## Related

- [model-types](../concepts/model-types.md)
- [incremental-model](../patterns/incremental-model.md)
- [dbt-databricks-adapter](dbt-databricks-adapter.md)

## Sources

- https://docs.getdbt.com/docs/build/incremental-strategy
- https://docs.getdbt.com/docs/build/incremental-microbatch
- https://docs.getdbt.com/reference/resource-configs/batch-size
- https://docs.getdbt.com/reference/resource-configs/event-time
- https://docs.getdbt.com/reference/resource-configs/databricks-configs
