# CDC Fundamentals

> **Purpose**: Change data capture with the AUTO CDC APIs (formerly APPLY CHANGES) in Lakeflow pipelines
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

The `AUTO CDC` APIs replace the `APPLY CHANGES` APIs with the same syntax; `APPLY CHANGES` still
works but Databricks recommends `AUTO CDC`. Two APIs exist, both in SQL and Python:
`AUTO CDC` for a change data feed, and `AUTO CDC FROM SNAPSHOT` for full snapshots. Both support SCD
Type 1 and Type 2; `AUTO CDC` also supports bitemporal storage (Beta). Requires serverless pipelines
or the `Pro`/`Advanced` edition. Not available in open-source Spark Declarative Pipelines beyond the
SCD Type 1 Auto CDC added in Spark 4.2.

## The Concept

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, expr

dp.create_streaming_table("customers")       # target must be a streaming table

dp.create_auto_cdc_flow(
    target="customers",
    source="customers_cdc_feed",             # streaming source
    keys=["customer_id"],
    sequence_by=col("event_ts"),             # required, sortable; struct() to break ties
    apply_as_deletes=expr("op = 'DELETE'"),
    apply_as_truncates=expr("op = 'TRUNCATE'"),  # SCD Type 1 only
    except_column_list=["op", "event_ts"],
    stored_as_scd_type=1,                    # default 1; "2" or "bitemporal" (Beta)
)
```

```sql
CREATE OR REFRESH STREAMING TABLE customers_history;

CREATE FLOW customers_history_cdc AS AUTO CDC INTO customers_history
FROM STREAM(customers_cdc_feed)
KEYS (customer_id)
APPLY AS DELETE WHEN op = 'DELETE'
SEQUENCE BY event_ts
COLUMNS * EXCEPT (op, event_ts)
STORED AS SCD TYPE 2
TRACK HISTORY ON * EXCEPT (last_login);
```

## Quick Reference

| SCD type | History | System columns | Current row |
|----------|---------|----------------|-------------|
| Type 1 | No | none | the row itself |
| Type 2 | Yes | `__START_AT`, `__END_AT` | `__END_AT IS NULL` |
| Bitemporal (Beta) | Business + system time | needs `system_sequence_by` | - |

| Parameter (Python / SQL) | Purpose |
|--------------------------|---------|
| `keys` / `KEYS` | Row identity (required) |
| `sequence_by` / `SEQUENCE BY` | Event order (required, non-null, sortable) |
| `apply_as_deletes` / `APPLY AS DELETE WHEN` | Delete condition (tombstones kept 2 days by default) |
| `apply_as_truncates` / `APPLY AS TRUNCATE WHEN` | Full truncate, SCD Type 1 only |
| `ignore_null_updates` / `IGNORE NULL UPDATES` | Keep existing value when incoming is null |
| `columns_to_update` / `COLUMNS TO UPDATE` | Per-record column list (`array<string>`) |
| `column_list`, `except_column_list` / `COLUMNS` | Columns kept in the target |
| `track_history_column_list` / `TRACK HISTORY ON` | Columns that open a new SCD2 version |
| `name` / flow name | Several AUTO CDC flows into one target (2025.30+) |
| `once=True` / `AUTO CDC ONCE` | One-time backfill flow |

Explicit target schema for SCD Type 2 must include `__START_AT` and `__END_AT` with the same type as
`sequence_by`. Tombstone retention: table property `pipelines.cdc.tombstoneGCThresholdInSeconds`.

### Snapshots

`dp.create_auto_cdc_from_snapshot_flow(target, source, keys, stored_as_scd_type, ..., once)` or SQL
`CREATE FLOW f AS AUTO CDC INTO t FROM SNAPSHOT (query) [WITH VERSION (query)] KEYS (...)`. Keys absent
from a newer snapshot are deleted. Expectations are not supported on snapshot flows. For SCD Type 1 in
triggered pipelines, one snapshot flow can share a target with one or more `AUTO CDC` flows.

## Common Mistakes

### Wrong

```python
dp.create_auto_cdc_flow(target="orders", source="orders_cdc", keys=["order_id"])
# missing sequence_by: out-of-order events cannot be resolved (and it is required)
```

### Correct

```python
dp.create_auto_cdc_flow(
    target="orders", source="orders_cdc",
    keys=["order_id"], sequence_by=col("updated_at"),
)
```

Also wrong: filtering SCD Type 2 with a `__IS_CURRENT` column (it does not exist; use
`__END_AT IS NULL`) and identity columns on an AUTO CDC target (not supported).

## Related

- [AUTO CDC pattern](../patterns/cdc-apply-changes.md)
- [CDC example](../patterns/cdc-example.md)
- [Core concepts](core-concepts.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes-from-snapshot
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-apply-changes-into
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-flow
- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/30/
- https://spark.apache.org/releases/spark-release-4-2-0.html
