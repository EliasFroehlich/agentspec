# CDC with AUTO CDC (formerly APPLY CHANGES)

> **Purpose**: SCD Type 1/2 targets from change feeds and snapshots with `dp.create_auto_cdc_flow`, `dp.create_auto_cdc_from_snapshot_flow` and `CREATE FLOW ... AS AUTO CDC INTO`
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- A source emits insert/update/delete events (database CDC, Debezium, Lakeflow Connect output)
- Only periodic full snapshots exist and you need the diff applied
- Several feeds (or a backfill plus a live feed) must land in one target

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, expr, struct

@dp.temporary_view
def users_feed():
    return spark.readStream.table("main.cdc.users_cdf")

# SCD Type 1 with deletes and truncates
dp.create_streaming_table("users_current")
dp.create_auto_cdc_flow(
    target="users_current",
    source="users_feed",
    keys=["user_id"],
    sequence_by=col("seq"),
    apply_as_deletes=expr("op = 'DELETE'"),
    apply_as_truncates=expr("op = 'TRUNCATE'"),   # SCD Type 1 only
    except_column_list=["op", "seq"],
    stored_as_scd_type=1,
)

# SCD Type 2 tracking history only for selected columns
dp.create_streaming_table("users_history")
dp.create_auto_cdc_flow(
    name="users_history_cdc",
    target="users_history",
    source="users_feed",
    keys=["user_id"],
    sequence_by=struct("event_ts", "seq"),         # tie-breaker
    apply_as_deletes=expr("op = 'DELETE'"),
    except_column_list=["op"],
    stored_as_scd_type="2",
    track_history_column_list=["email", "city"],
)

# Partial updates: keep existing values when the incoming column is null
dp.create_streaming_table("users_profile")
dp.create_auto_cdc_flow(
    target="users_profile", source="users_feed",
    keys=["user_id"], sequence_by=col("seq"),
    ignore_null_updates=True,
)

# Snapshot source: diff consecutive snapshots of a table
dp.create_streaming_table("products_current")
dp.create_auto_cdc_from_snapshot_flow(
    target="products_current",
    source="main.erp.products_snapshot",
    keys=["product_id"],
    stored_as_scd_type=1,
)
```

```sql
-- SQL equivalent of the SCD Type 2 flow
CREATE OR REFRESH STREAMING TABLE users_history;

CREATE FLOW users_history_cdc AS AUTO CDC INTO users_history
FROM STREAM(main.cdc.users_cdf)
KEYS (user_id)
APPLY AS DELETE WHEN op = 'DELETE'
SEQUENCE BY STRUCT(event_ts, seq)
COLUMNS * EXCEPT (op)
STORED AS SCD TYPE 2
TRACK HISTORY ON (email, city);

-- Snapshot flow: one-time initial load (add WITH VERSION (...) to process later snapshots)
CREATE OR REFRESH STREAMING TABLE products_current;

CREATE FLOW products_snapshot AS AUTO CDC ONCE INTO products_current
FROM SNAPSHOT (SELECT * FROM main.erp.products_snapshot)
KEYS (product_id)
STORED AS SCD TYPE 1;
```

## Configuration

| Setting | Default | Notes |
|---------|---------|-------|
| `stored_as_scd_type` | `1` | `"2"` adds `__START_AT`/`__END_AT`; `"bitemporal"` (Beta) needs `system_sequence_by` |
| `sequence_by` | required | Sortable, non-null; one distinct update per key per sequence value |
| `apply_as_deletes` | none | Tombstones kept 2 days; property `pipelines.cdc.tombstoneGCThresholdInSeconds` |
| `apply_as_truncates` | none | SCD Type 1 only |
| `ignore_null_updates` | `False` | Or `ignore_null_updates_column_list` / `..._except_column_list` |
| `columns_to_update` | none | Not combinable with `ignore_null_updates*`; not for bitemporal |
| `name` | target name | Required to run several AUTO CDC flows into one target |
| `once` | `False` | One-time backfill; re-runs on full refresh |
| Edition | - | Serverless, or classic `Pro`/`Advanced` |

Snapshot SQL without `WITH VERSION` only loads while the target is empty; to process later snapshots
add `WITH VERSION (query)` (see the CREATE FLOW reference). Snapshot flows take no expectations.

## Example Usage

```sql
-- Current rows of an SCD Type 2 table
SELECT * FROM users_history WHERE __END_AT IS NULL;
```

Combining one snapshot flow with AUTO CDC flows on the same target works only for SCD Type 1 in
triggered pipelines, with unique flow names, identical keys in the same order, and the same data type
for the snapshot version and `sequence_by`.

## See Also

- [CDC fundamentals](../concepts/cdc-fundamentals.md)
- [CDC example](cdc-example.md)
- [SQL streaming](sql-streaming.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes-from-snapshot
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-apply-changes-into
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-flow
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
