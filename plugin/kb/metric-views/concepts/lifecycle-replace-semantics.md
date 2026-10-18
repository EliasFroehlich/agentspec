# Lifecycle and Replace Semantics

> **Purpose**: What survives when an existing metric view is changed — owner, grants, tags, comments — under `CREATE OR REPLACE`, `COPY *`/`NOCOPY *` and `ALTER VIEW ... AS $$yaml$$`
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

There are three ways to change an existing metric view and they do not keep the same metadata. `CREATE OR REPLACE` makes the caller the new owner and, depending on the runtime, may drop grants, tags and comments; `COPY *` (DBR 18.3+) makes preservation explicit; `ALTER VIEW ... AS $$ ... $$` replaces the full YAML in place and keeps privileges. A generator that re-emits DDL on every run must pick one deliberately and verify the result.

## The Concept

```sql
-- A) Edit in place: privileges preserved (documented); caller must own the view;
--    always send the COMPLETE YAML (it replaces the whole definition)
ALTER VIEW main.sales.orders_metrics AS $$
version: 1.1
source: main.sales.orders
fields:
  - name: order_month
    expr: DATE_TRUNC('MONTH', order_date)
    comment: "Month of order"          # keep every UC comment in YAML: ALTER VIEW drops the others
measures:
  - name: total_revenue
    expr: SUM(total_price)
$$;

-- B) Replace and keep grants/comments/tags explicitly (DBR 18.3+, UC managed views, OR REPLACE only).
--    The CREATE VIEW grammar allows COPY * next to WITH METRICS / LANGUAGE YAML; the docs show it
--    only on a query view, so try it in dev first.
CREATE OR REPLACE VIEW main.sales.orders_metrics
  WITH METRICS LANGUAGE YAML COPY *
  AS $$ ...complete YAML... $$;

-- C) Whatever path was used: re-apply and verify (idempotent)
ALTER VIEW main.sales.orders_metrics SET TAGS ('domain' = 'sales');   -- needs APPLY TAG
GRANT SELECT ON main.sales.orders_metrics TO `data-consumers`;
SHOW GRANTS ON VIEW main.sales.orders_metrics;
SELECT tag_name, tag_value FROM main.information_schema.table_tags
WHERE schema_name = 'sales' AND table_name = 'orders_metrics';
```

## Quick Reference

| Operation | Owner after | Grants | Tags | UC comments | Runtime |
|-----------|-------------|--------|------|-------------|---------|
| `CREATE OR REPLACE` (no clause) | Caller | Not kept (same as `NOCOPY *`) | Not kept | Not kept | DBR 18.3 |
| `CREATE OR REPLACE` (no clause) | Caller | Kept (same as `COPY *`) | Kept | Kept | DBR 19+ |
| `CREATE OR REPLACE ... COPY *` | Caller | Kept | Kept | Kept | DBR 18.3+ |
| `CREATE OR REPLACE ... NOCOPY *` | Caller | Dropped | Dropped | Only those in the new definition | DBR 18.3+ |
| `CREATE OR REPLACE` on a view that is not UC managed | Caller | Lost (= `DROP` + `CREATE`, new `table_id`) | Lost | Lost | Any |
| `ALTER VIEW ... AS $$yaml$$` | Unchanged (owner only) | Kept | Not stated in docs — re-apply | Only those in YAML `comment:` | DBR 16.4+, UC only |
| `ALTER VIEW ... SET TAGS` / `UNSET TAGS` | Unchanged | — | Changed | — | `APPLY TAG` privilege |
| `ALTER VIEW ... OWNER TO <group>` | Group | — | — | — | Not for materialized metric views |

SQL warehouses: Databricks SQL 2026.35 (Preview, 2026-08-24) changed `CREATE OR REPLACE VIEW` on Unity Catalog views to an atomic replacement that keeps view identity, grants, tags and comments by default; 2026.36 started rolling out to the Current channel on 2026-09-08. The release note does not name metric views explicitly.

## Field Evidence (production project, SQL warehouse, 2026-09)

- A `CREATE OR REPLACE VIEW ... WITH METRICS` removed the view's tags; `ALTER VIEW ... SET TAGS` works on metric views; comments survived because they live in the YAML.
- A later replace also removed direct grants (two service principals, 2 → 0). The fix was a post-deploy step that re-applies tags, then grants. `SHOW GRANTS` reports a metric view with object type `TABLE`.
- Both measurements were taken after the 2026.36 Current rollout started, so treat "preserved by default" as unconfirmed for metric views until a test passes (count grants and tags before and after one replace).

## Recommended Flow for Generated DDL

1. **View exists** → `ALTER VIEW <name> AS $$ <complete YAML> $$` (keeps grants; run as the owner — a service principal or a group owner, not a person).
2. **View doesn't exist** → `CREATE VIEW ... WITH METRICS LANGUAGE YAML AS $$ ... $$`.
3. **Must use `CREATE OR REPLACE`** (e.g., a tool that only emits it, such as dbt-databricks `metric_view`) → add `COPY *` where the runtime is confirmed 18.3+, and always run step 4.
4. **After any DDL**, re-apply tags and grants idempotently and assert them (`SHOW GRANTS`, `information_schema.table_tags`) before marking the deploy green.
5. Run every replace as the **same principal**: the replacing user becomes the owner, and a materialized metric view can't change owner after its materialization exists.

## Common Mistakes

### Wrong

```sql
-- Daily job re-applies the definition with a plain replace (DBR 18.3 or an unverified warehouse)
CREATE OR REPLACE VIEW main.sales.orders_metrics WITH METRICS LANGUAGE YAML AS $$ ... $$;
-- Grants and tags silently disappear; consumers inheriting from the schema keep working,
-- so nobody notices until the view moves to a catalog without that inheritance.
```

### Correct

```sql
ALTER VIEW main.sales.orders_metrics AS $$ ...complete YAML... $$;
ALTER VIEW main.sales.orders_metrics SET TAGS ('domain' = 'sales');
GRANT SELECT ON main.sales.orders_metrics TO `data-consumers`;
SHOW GRANTS ON VIEW main.sales.orders_metrics;   -- assert expected principals
```

## Related

- [Requirements & Versions](requirements-versions.md)
- [Deploy with bundles](../patterns/deploy-with-bundles.md)
- [Migrate from a view](../patterns/migrate-from-view.md)
- [Materialization](materialization.md)

## Sources

- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view (`OR REPLACE`, `COPY *`/`NOCOPY *`, defaults by runtime, owner)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view (`AS yaml_definition`, privileges preserved, `SET TAGS`, `OWNER TO`)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage (edit with `ALTER VIEW`, collaborative editing)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (upgrade to 1.1: `ALTER VIEW` removes UC comments not in YAML)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization (owner can't change after materialization)
- https://docs.databricks.com/aws/en/sql/release-notes/2026 (2026.35 atomic replace; 2026.36 Current rollout)
- https://docs.databricks.com/aws/en/sql/language-manual/security-show-grant
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-privileges (securable object syntax)
- https://docs.databricks.com/aws/en/sql/language-manual/information-schema/table_tags
