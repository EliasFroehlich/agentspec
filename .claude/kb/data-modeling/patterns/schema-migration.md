# Schema Migration

> **Purpose**: Backward-compatible migrations, blue-green deploys, column deprecation, testing
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Schema migrations in data warehouses must be backward-compatible to avoid breaking downstream consumers. The golden rule: additive changes are safe; removals require a deprecation lifecycle. Blue-green deployments allow testing new schemas before cutover.

## The Pattern

```sql
-- ============================================================
-- Safe Migration: Add column + deprecate old column
-- ============================================================

-- Step 1: Additive change (always safe). On Delta, ADD COLUMN cannot carry a DEFAULT;
-- set it afterwards with ALTER COLUMN ... SET DEFAULT (applies to new rows only).
-- ADD COLUMN(S) IF NOT EXISTS is accepted by Databricks Runtime in practice but is
-- not in the Databricks/Spark/Delta references and fails on OSS Spark/Delta 4.1.
-- For documented, portable reruns, check information_schema.columns first.
ALTER TABLE orders ADD COLUMN customer_tier STRING;

-- Step 2: Backfill new column
UPDATE orders
SET customer_tier = CASE
    WHEN total_amount > 10000 THEN 'platinum'
    WHEN total_amount > 5000  THEN 'gold'
    WHEN total_amount > 1000  THEN 'silver'
    ELSE 'bronze'
END;

-- Step 3: Mark old column as deprecated (Databricks COMMENT ON COLUMN: DBR 16.1+;
-- older runtimes: ALTER TABLE orders ALTER COLUMN customer_level COMMENT '...')
COMMENT ON COLUMN orders.customer_level IS
    'DEPRECATED <date>: Use customer_tier instead. Removal: <date>';

-- Step 4: Create view for backward compatibility during transition
CREATE OR REPLACE VIEW orders_compat AS
SELECT
    *,
    customer_tier AS customer_level  -- alias for old name
FROM orders;

-- Step 5: After migration window (30+ days), drop old column
-- Only after confirming no consumers reference it
ALTER TABLE orders DROP COLUMN customer_level;

-- ============================================================
-- Blue-Green Schema Deploy
-- ============================================================

-- Green (new schema): create alongside existing.
-- CTAS cannot define a schema, identity columns or PK/FK constraints (Databricks docs).
-- For those, CREATE TABLE orders_v2 (...full DDL...) then INSERT INTO orders_v2 SELECT ...
CREATE TABLE orders_v2 AS
SELECT
    order_id,
    customer_id,
    customer_tier,      -- new column
    order_date,
    total_amount
FROM orders;

-- Validation: compare row counts and checksums
SELECT
    (SELECT COUNT(*) FROM orders) AS blue_count,
    (SELECT COUNT(*) FROM orders_v2) AS green_count,
    (SELECT SUM(total_amount) FROM orders) AS blue_sum,
    (SELECT SUM(total_amount) FROM orders_v2) AS green_sum;

-- Cutover: two renames (not one atomic swap). Re-apply grants, tags, comments and
-- clustering on orders_v2 before the swap, and check views that reference the name.
ALTER TABLE orders RENAME TO orders_deprecated;
ALTER TABLE orders_v2 RENAME TO orders;

-- Rollback if needed
ALTER TABLE orders RENAME TO orders_v2;
ALTER TABLE orders_deprecated RENAME TO orders;
```

## Quick Reference

| Change | Risk | Delta 4.x Strategy | Iceberg v3 Strategy |
|--------|------|-------------------|-------------------|
| Add nullable column | None | `ALTER TABLE ADD COLUMN`, `MERGE/INSERT WITH SCHEMA EVOLUTION`, `mergeSchema` | `ALTER TABLE ADD` |
| Add column with default | None | `ADD COLUMN`, then `ALTER COLUMN ... SET DEFAULT` | Default values (v3) |
| Add NOT NULL column | Low | Add nullable -> backfill -> `ALTER COLUMN ... SET NOT NULL` | Add with default value (v3) |
| Rename column | Medium | Column mapping mode (name) | `ALTER TABLE RENAME` |
| Change type (widen) | Low | **Type widening** (no rewrite; Delta 4.0 GA, DBR 15.4 LTS+) | `ALTER TABLE ALTER TYPE` |
| Change type (narrow) | High | New column -> CAST -> validate -> swap | Not allowed (by design) |
| Drop column | High | Column mapping + deprecation lifecycle | `ALTER TABLE DROP` + deprecation |
| Drop table | Critical | Rename to _deprecated -> wait -> drop | Same pattern |
| Add Variant column | None | Native (Delta 4.0+, DBR 15.4+) | Native (v3) |
| Replace whole schema | High | `CREATE OR REPLACE TABLE` with full DDL (keeps history and grants) — never `overwriteSchema` on shared tables | `CREATE OR REPLACE TABLE` |

## Common Mistakes

### Wrong

```sql
-- Immediate drop without deprecation — breaks all downstream consumers
ALTER TABLE orders DROP COLUMN customer_level;
```

### Correct

```sql
-- Deprecation lifecycle: mark → communicate → wait → validate → drop
COMMENT ON COLUMN orders.customer_level IS 'DEPRECATED <date>. Removal: <date>';
-- Wait 30+ days, verify no consumers, then drop
```

## Related

- [schema-evolution](../concepts/schema-evolution.md)
- [data-contracts](../../data-quality/concepts/data-contracts.md)
- [scd-types](../concepts/scd-types.md)
- [medallion schema-evolution](../../medallion/patterns/schema-evolution.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table-manage-column
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-comment
- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/tables/features/generated-columns
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-using
- https://docs.databricks.com/aws/en/tables/features/type-widening
