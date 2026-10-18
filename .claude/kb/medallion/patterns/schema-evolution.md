# Schema Evolution

> **Purpose**: Change schemas across Medallion layers without breaking consumers — additive evolution in Bronze, explicit DDL migrations in Silver/Gold, Delta defaults, type widening, column mapping, Iceberg equivalents
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## When to Use

- A source adds columns that must flow through the layers
- A column needs a wider type (INT → BIGINT, DECIMAL(10,2) → DECIMAL(12,2))
- Columns are renamed or removed upstream
- A Gold table's contract changes and consumers must not break

## Implementation

```python
def existing_columns(table: str) -> set[str]:
    return {f.name.lower() for f in spark.table(table).schema.fields}


def add_column(table: str, name: str, dtype: str, default_sql: str | None = None, comment: str = ""):
    """Idempotent ADD COLUMN for an existing Delta table (Databricks or Delta OSS).
    DEFAULT cannot be part of ADD COLUMN on an existing Delta table
    (WRONG_COLUMN_DEFAULTS_FOR_DELTA_ALTER_TABLE_ADD_COLUMN_NOT_SUPPORTED):
    add the column, then SET DEFAULT, which applies to rows inserted afterwards only."""
    if name.lower() in existing_columns(table):
        return                                          # rerun-safe without relying on IF NOT EXISTS
    comment_sql = f" COMMENT '{comment}'" if comment else ""
    spark.sql(f"ALTER TABLE {table} ADD COLUMN {name} {dtype}{comment_sql}")   # existing rows: NULL
    if default_sql is not None:
        # Needs the allowColumnDefaults table feature; otherwise the command fails with
        # WRONG_COLUMN_DEFAULTS_FOR_DELTA_FEATURE_NOT_ENABLED.
        spark.sql(f"ALTER TABLE {table} SET TBLPROPERTIES "
                  f"('delta.feature.allowColumnDefaults' = 'supported')")
        spark.sql(f"ALTER TABLE {table} ALTER COLUMN {name} SET DEFAULT {default_sql}")  # DBR 11.3 LTS+


def widen_type(table: str, name: str, new_type: str):
    """Metadata-only widening (DBR 15.4 LTS+ / Delta 4.0+): INT->BIGINT, DECIMAL(p,s)->wider, etc."""
    spark.sql(f"ALTER TABLE {table} SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true')")
    spark.sql(f"ALTER TABLE {table} ALTER COLUMN {name} TYPE {new_type}")


def rename_or_drop(table: str, sql_clause: str):
    """Rename/drop are metadata-only with column mapping. New tables: create with
    'delta.columnMapping.mode' = 'id' (recommended; id mode can only be set at creation);
    existing tables: set 'name' mode. Streaming readers then need a schemaTrackingLocation."""
    spark.sql(f"ALTER TABLE {table} {sql_clause}")    # e.g. "RENAME COLUMN amt TO amount"


add_column("main.silver.orders", "loyalty_tier", "STRING", default_sql="'standard'")
widen_type("main.silver.orders", "amount", "DECIMAL(12,2)")
```

## Configuration

| Setting | Default | Description |
|---|---|---|
| `mergeSchema` (writer option) | `false` | Adds new source columns on append/overwrite writes; Bronze only |
| `MERGE WITH SCHEMA EVOLUTION` / `INSERT WITH SCHEMA EVOLUTION` | — | Per-statement evolution: MERGE DBR 15.2+, INSERT DBR 18.1+ (Delta OSS 4.3+) |
| `spark.databricks.delta.schema.autoMerge.enabled` | `false` | Legacy session-wide switch; prefer per-statement options |
| `delta.columnMapping.mode` | `none` | `id` (new tables, recommended) or `name` (existing tables); reader 2 / writer 5 |
| `delta.enableTypeWidening` | `false` | Enables `ALTER COLUMN ... TYPE` widenings without rewrite |
| `delta.feature.allowColumnDefaults` | not enabled | Needed before `SET DEFAULT` on tables created without defaults |
| `cloudFiles.schemaEvolutionMode` | `addNewColumns` | Auto Loader; `addNewColumnsWithTypeWidening` (DBR 16.4+, Public Preview), `rescue`, `failOnNewColumns`, `none` |

## Schema Evolution by Layer

| Capability | Bronze | Silver | Gold |
|---|---|---|---|
| New columns | Automatic (`mergeSchema`, Auto Loader `addNewColumns`) or VARIANT payload | Explicit `ADD COLUMN`, or `MERGE WITH SCHEMA EVOLUTION` when intended | Explicit `ALTER TABLE ADD COLUMN` (DDL-first) |
| Defaults | — | `ADD COLUMN`, then `ALTER COLUMN ... SET DEFAULT` | Same |
| Rename / drop | Not needed | Column mapping + `RENAME/DROP COLUMN`; `REORG TABLE ... APPLY (PURGE)` rewrites dropped data | Same, after consumers migrate |
| Type change | Not needed (strings/VARIANT) | Type widening; otherwise new column + backfill + swap | Same |
| Full schema replace | Never | Never on shared tables | New table via DDL + `INSERT`, validate, swap; or `CREATE OR REPLACE TABLE` (keeps history and grants) |

```python
# Wrong: implicit contract change. Replaces schema and partitioning under every consumer in
# one commit; streaming readers fail on the schema change; comments on recomputed columns
# vanish. Do not use for Silver/Gold.
df.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable("main.gold.orders")
```

### Iceberg Schema Evolution at Each Layer

```sql
-- Iceberg (Spark + Iceberg runtime): add, rename, widen and drop are metadata operations
ALTER TABLE catalog.bronze.raw_events ADD COLUMN new_field STRING;
ALTER TABLE catalog.silver.orders RENAME COLUMN amt TO amount;
ALTER TABLE catalog.silver.orders ALTER COLUMN quantity TYPE BIGINT;
ALTER TABLE catalog.silver.orders DROP COLUMN deprecated_flag;
```

## SQL Schema Migration Pattern (Delta on Databricks)

```sql
-- 1. One-time: column mapping on an existing table (new tables: 'id' at CREATE)
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');

-- 2. Add column, then its default for future rows, then backfill existing rows
ALTER TABLE main.silver.orders ADD COLUMN loyalty_tier STRING COMMENT 'Customer tier';
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('delta.feature.allowColumnDefaults' = 'supported');
ALTER TABLE main.silver.orders ALTER COLUMN loyalty_tier SET DEFAULT 'standard';
UPDATE main.silver.orders
SET loyalty_tier = CASE WHEN lifetime_revenue > 10000 THEN 'gold'
                        WHEN lifetime_revenue > 5000  THEN 'silver'
                        ELSE 'standard' END
WHERE loyalty_tier IS NULL;

-- 3. Rename, widen, drop
ALTER TABLE main.silver.orders RENAME COLUMN amt TO amount;
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');
ALTER TABLE main.silver.orders ALTER COLUMN amount TYPE DECIMAL(12,2);
ALTER TABLE main.silver.orders DROP COLUMN IF EXISTS legacy_field;   -- DBR 11.3 LTS+
```

`ADD COLUMN(S) IF NOT EXISTS` is **not** in the Databricks SQL, Apache Spark 4.2 or Delta Lake 4.4 `ALTER TABLE` references (only `DROP COLUMN IF EXISTS` and `ADD IF NOT EXISTS PARTITION` are). Databricks Runtime accepts it in practice (used in production DDL), but it is undocumented and fails with `PARSE_SYNTAX_ERROR` on open-source Spark 4.1 / Delta 4.1. If you rely on it, pin that to Databricks and re-test on each runtime upgrade; for portable, documented idempotency, check `information_schema.columns` or the table schema first, as in `add_column` above.

## Example Usage

```python
for col_name, dtype, default in [("channel", "STRING", "'web'"), ("is_test", "BOOLEAN", "false")]:
    add_column("main.silver.orders", col_name, dtype, default_sql=default)
spark.sql("DESCRIBE TABLE main.silver.orders").show()
```

## See Also

- [Bronze Layer](../concepts/bronze-layer.md)
- [Silver Layer](../concepts/silver-layer.md)
- [Incremental Loading](incremental-loading.md)
- [Schema migration (data-modeling)](../../data-modeling/patterns/schema-migration.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table-manage-column
- https://docs.databricks.com/aws/en/error-messages/error-classes
- https://docs.databricks.com/aws/en/tables/update-schema
- https://docs.databricks.com/aws/en/tables/features/type-widening
- https://docs.databricks.com/aws/en/tables/features/column-mapping
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-using
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/structured-streaming/delta-lake
- https://spark.apache.org/docs/latest/sql-ref-syntax-ddl-alter-table.html
- https://docs.delta.io/delta-batch/
