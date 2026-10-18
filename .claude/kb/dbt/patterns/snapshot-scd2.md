# Snapshot SCD Type 2

> **Purpose**: Track historical changes in mutable source data with dbt snapshots defined in YAML (dbt 1.9+), including hard-delete handling and custom validity columns
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Source rows are updated in place (e.g. customer address changes)
- You need "what was the value at point in time X?"
- Building slowly changing dimensions (Type 2) for a dimensional model
- Audit trails for compliance

On Databricks Lakeflow pipelines, `AUTO CDC … STORED AS SCD TYPE 2` is the native alternative; snapshots are
the dbt-side tool.

## Implementation

```yaml
# snapshots/snapshots.yml — YAML snapshots (dbt 1.9+); target_schema is optional
snapshots:
  - name: snap_customers
    relation: source('crm', 'customers')          # or ref('stg_customers')
    config:
      schema: snapshots                            # resolved via generate_schema_name, like models
      unique_key: customer_id                      # a list is allowed for composite keys
      strategy: timestamp
      updated_at: updated_at
      hard_deletes: invalidate                     # ignore (default) | invalidate | new_record
      dbt_valid_to_current: "cast('9999-12-31' as timestamp)"   # instead of NULL for current rows

  - name: snap_products                            # no reliable updated_at → compare columns
    relation: source('catalog', 'products')
    config:
      schema: snapshots
      unique_key: product_id
      strategy: check
      check_cols: ['price', 'category', 'is_active']
      hard_deletes: new_record                     # adds dbt_is_deleted rows
      snapshot_meta_column_names:
        dbt_valid_from: valid_from
        dbt_valid_to: valid_to
        dbt_scd_id: scd_id
        dbt_updated_at: modified_at
        dbt_is_deleted: is_deleted
```

To snapshot a transformed query, put the logic in an ephemeral or view model and point `relation:` at
`ref('that_model')` — the YAML form has no inline `select`.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `relation` | — | `source(...)` or `ref(...)` to snapshot (required in YAML snapshots) |
| `strategy` | — | `timestamp` (uses `updated_at`) or `check` (compares `check_cols`) |
| `unique_key` | — | Business key; column, expression or list of columns |
| `updated_at` | — | Timestamp column (required for `timestamp`) |
| `check_cols` | — | List of columns or `all` (for `check`) |
| `hard_deletes` | `ignore` | `invalidate` closes the row (`dbt_valid_to` set); `new_record` inserts a row with `dbt_is_deleted = True` (and `False` when restored). Replaces `invalidate_hard_deletes` |
| `dbt_valid_to_current` | `NULL` | Literal used for current rows' `dbt_valid_to` (e.g. far-future date) |
| `snapshot_meta_column_names` | dbt names | Rename `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`, `dbt_updated_at`, `dbt_is_deleted` |
| `schema` / `database` | target | Same resolution as models; legacy `target_schema` still accepted |

Legacy `{% snapshot %}` SQL blocks in `snapshots/*.sql` still run, but new snapshots should use YAML.
Snapshots ignore `--full-refresh`.

## Example Usage

```bash
dbt snapshot                       # run all snapshots
dbt snapshot --select snap_customers
dbt build                          # snapshots, models and tests in DAG order
```

```sql
-- Point-in-time state (with dbt_valid_to_current set, current rows are not NULL)
select *
from snapshots.snap_customers
where customer_id = 'C-123'
  and timestamp'2025-06-15 00:00:00' >= dbt_valid_from
  and timestamp'2025-06-15 00:00:00' <  dbt_valid_to
```

## See Also

- [incremental-model](../patterns/incremental-model.md)
- [model-types](../concepts/model-types.md)
- [SCD types (data-modeling KB)](../../data-modeling/concepts/scd-types.md)

## Sources

- https://docs.getdbt.com/docs/build/snapshots
- https://docs.getdbt.com/reference/snapshot-properties
- https://docs.getdbt.com/reference/resource-configs/hard-deletes
- https://docs.getdbt.com/reference/resource-configs/dbt_valid_to_current
- https://docs.getdbt.com/reference/resource-configs/snapshot_meta_column_names
- https://docs.databricks.com/aws/en/ldp/cdc
