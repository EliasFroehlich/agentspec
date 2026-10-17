# Schema Evolution

> **Purpose**: Iceberg/Delta schema evolution, Avro/Protobuf rules, backward/forward compatibility, Delta defaults and type widening by version
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Schema evolution handles column additions, type changes and renames without breaking downstream consumers. Open table formats (Iceberg, Delta) support evolution natively. The key principle: additive changes are safe; removals, renames and type narrowing are breaking. For shared tables, apply changes explicitly (DDL) rather than letting a write replace the schema.

## The Concept

```sql
-- Iceberg: add, rename, widen and drop are metadata operations
ALTER TABLE catalog.db.orders ADD COLUMN loyalty_tier STRING;
ALTER TABLE catalog.db.orders RENAME COLUMN loyalty_tier TO customer_tier;
ALTER TABLE catalog.db.orders ALTER COLUMN quantity TYPE BIGINT;  -- widening: safe
ALTER TABLE catalog.db.orders DROP COLUMN deprecated_flag;

-- Delta on Databricks: explicit DDL
ALTER TABLE main.sales.orders ADD COLUMN loyalty_tier STRING;             -- existing rows: NULL
ALTER TABLE main.sales.orders ALTER COLUMN loyalty_tier SET DEFAULT 'standard';
-- (SET DEFAULT needs the allowColumnDefaults table feature; DEFAULT inside ADD COLUMN is rejected)

-- Delta: evolve from the data in one statement
MERGE WITH SCHEMA EVOLUTION INTO main.sales.orders AS t                   -- DBR 15.2+
USING main.sales.orders_updates AS s ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
INSERT WITH SCHEMA EVOLUTION INTO main.sales.orders SELECT * FROM src;   -- DBR 18.1+ / Delta OSS 4.3+
-- DataFrame writers: .option("mergeSchema", "true") (writer option, not a JSON reader option)

-- Avro: backward-compatible evolution
-- Safe: add field with default, remove field with default
-- Breaking: add field without default, change type, rename field
```

## Quick Reference

| Change Type | Iceberg v3 | Delta 4.x / Databricks | Avro | Breaking? |
|---|---|---|---|---|
| Add column (nullable) | `ALTER TABLE ADD COLUMN` | `ALTER TABLE ADD COLUMN`, `MERGE/INSERT WITH SCHEMA EVOLUTION`, `mergeSchema` | Add with default | No |
| Add column with default | `ADD COLUMN`; v3 stores `initial-default`/`write-default` | `ADD COLUMN`, then `ALTER COLUMN ... SET DEFAULT` (new rows only) | Add with default | No |
| Add NOT NULL column | Depends on engine | Add nullable → backfill → `ALTER COLUMN ... SET NOT NULL` | No default = breaking | Yes |
| Drop column | `ALTER TABLE DROP COLUMN` | Column mapping + `DROP COLUMN` | Remove with default | Depends |
| Rename column | `ALTER TABLE RENAME COLUMN` | Column mapping + `RENAME COLUMN` | N/A (aliases) | Yes (for consumers) |
| Widen type (INT→BIGINT) | `ALTER COLUMN ... TYPE` | **Type widening**: Delta preview 3.2, GA 4.0; DBR 15.4 LTS+ | Promotion rules | No |
| Narrow type (BIGINT→INT) | Not allowed | Not allowed | Not allowed | Yes |
| Reorder columns | Supported | `ALTER COLUMN ... FIRST/AFTER` | Positional in Avro | No |
| Variant type | v3 `variant` | Delta 4.0+, DBR 15.4+ | N/A | No (additive) |
| Geospatial types | v3 `geometry`/`geography` | Delta Kernel 4.2+ (Spark support not documented) | N/A | No (additive) |

### Delta Type Widening

```sql
-- Enable once (DBR 15.4 LTS+ / Delta 4.0+); no data files rewritten
ALTER TABLE main.sales.orders SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true');

-- Supported widenings:
-- BYTE -> SHORT, INT, BIGINT, DECIMAL, DOUBLE
-- SHORT -> INT, BIGINT, DECIMAL, DOUBLE
-- INT -> BIGINT, DECIMAL, DOUBLE      (INT -> DECIMAL/DOUBLE: manual ALTER only)
-- BIGINT -> DECIMAL
-- FLOAT -> DOUBLE
-- DECIMAL -> DECIMAL (greater precision/scale)
-- DATE -> TIMESTAMP_NTZ
-- VOID -> any type (DBR 18.2+)
ALTER TABLE main.sales.orders ALTER COLUMN quantity TYPE BIGINT;
```

## Common Mistakes

### Wrong

```sql
-- Breaking change with no deprecation period: consumers break immediately
ALTER TABLE orders DROP COLUMN customer_email;
```

```python
# Replacing the schema from a DataFrame: changes the contract and partitioning in one commit,
# fails streaming readers, drops comments the DataFrame no longer carries
df.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable("main.sales.orders")
```

### Correct

```sql
-- Deprecation lifecycle
ALTER TABLE main.sales.orders ADD COLUMN customer_email_v2 STRING;
COMMENT ON COLUMN main.sales.orders.customer_email IS
  'DEPRECATED: use customer_email_v2. Removal: <date>';      -- Databricks: DBR 16.1+
-- Older runtimes: ALTER TABLE main.sales.orders ALTER COLUMN customer_email COMMENT '...';
-- Migrate consumers, then drop (needs column mapping on Delta)
ALTER TABLE main.sales.orders DROP COLUMN customer_email;
```

## Related

- [scd-types](scd-types.md)
- [schema-migration pattern](../patterns/schema-migration.md)
- [medallion schema-evolution pattern](../../medallion/patterns/schema-evolution.md)
- [data-contracts](../../data-quality/concepts/data-contracts.md)

## Sources

- https://docs.databricks.com/aws/en/tables/update-schema
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table-manage-column
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-dml-insert-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-comment
- https://docs.databricks.com/aws/en/tables/features/type-widening
- https://docs.delta.io/latest/delta-type-widening.html
- https://docs.delta.io/delta-batch/
- https://iceberg.apache.org/spec/
- https://iceberg.apache.org/docs/latest/spark-ddl/
