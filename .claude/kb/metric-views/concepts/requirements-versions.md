# Requirements & Versions

> **Purpose**: YAML spec versions, runtime minimums per feature, permissions, and the ways to create and edit a metric view
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Two independent axes gate what a metric view can use: the **YAML specification version** (`version:`, required, versioned independently of the runtime) and the **compute runtime** (Databricks Runtime on clusters; the Databricks SQL version on SQL warehouses, which update automatically by channel). Only two spec versions are documented: `0.1` (initial) and `1.1`. Write `version: 1.1` for anything new.

## The Concept

```yaml
version: 1.1                      # required; spec format version, not your own revision number
source: main.sales.orders
comment: "Orders KPIs"            # spec 1.1
fields:
  - name: order_month
    expr: DATE_TRUNC('MONTH', order_date)
    display_name: Order Month     # DBR 17.3+ and spec 1.1
    synonyms: [month, period]     # DBR 17.3+ and spec 1.1; up to 10, 255 chars each
measures:
  - name: total_revenue
    expr: SUM(total_price)
    format:                       # DBR 17.3+ and spec 1.1; `type` is required
      type: currency
      currency_code: USD
```

## Spec Versions

| Version | Adds | Runtime |
|---------|------|---------|
| `0.1` | Initial specification | DBR 16.4+ |
| `1.1` | `comment` on the view, fields and measures; agent metadata (`display_name`, `format`, `synonyms`) | DBR 17.3+ |
| `1.1` | `cardinality` on joins (incl. `one_to_many`), window `offset`, `inclusive`/`exclusive` on `trailing`/`leading` | DBR 18.1+ |
| `1.1` | Wildcards in `fields`/`measures` | DBR 18.2+ |
| `1.1` | Unitless `offset`/`range` on a consecutive integer index | DBR 19+ |

The official `bundle-examples/knowledge_base/metric_view` sample writes `version: 1.0`, which the feature-availability page does not list. Don't copy it; use `1.1`.

## Runtime Matrix (minimum Databricks Runtime)

| DBR | Features |
|-----|----------|
| 16.4 | Metric views introduced; `MEASURE()`; `ALTER VIEW ... AS $$yaml$$` |
| 17.3 | Snowflake schema; agent metadata; `TEMPORARY` metric views; materialization; JDBC/ODBC through Thrift |
| 18.0 | BI compatibility mode; `REFRESH MATERIALIZED VIEW` |
| 18.1 | One-to-many joins; window `offset`; `inclusive`/`exclusive`; `rely.at_most_one_match`; `MEASURE(...) FILTER (WHERE ...)`; `agg()` synonym |
| 18.2 | Parameters (Public Preview); wildcard fields/measures |
| 18.3 | `COPY *` / `NOCOPY *` on `CREATE OR REPLACE VIEW` |
| 19 | Derived column names (`name` optional); numeric-index windows; `OR REPLACE` preserves metadata by default |

Window measures (GA 2026-10-06) have no separate row in the official table; their `offset`/`inclusive` modifiers follow the 18.1 row. Clusters pinned to DBR 17.3 LTS (this project's job runtime) cannot use any 18.x/19 feature: run metric view DDL and queries on a SQL warehouse, or check the warehouse channel first.

## Permissions

| To | Needs |
|----|-------|
| Create | `SELECT` on the source asset or tables in the SQL source; `USE CATALOG` on the catalog; `USE SCHEMA` + `CREATE TABLE` on the target schema; `CAN USE` on a warehouse/compute (DBR 17.3+ for the official example) |
| Query | `SELECT` on the metric view (UC privileges cascade from catalog/schema) |
| Edit the definition | Ownership (transfer to a group for collaborative editing; not supported for materialized metric views) |
| `CREATE OR REPLACE` | Ownership or `MANAGE`; the replacing user becomes the new owner |
| Tag | `APPLY TAG` (`ALTER VIEW ... SET TAGS`), no ownership needed |

## Create and Edit Methods

| Method | Notes |
|--------|-------|
| SQL DDL | `CREATE [OR REPLACE] VIEW ... WITH METRICS LANGUAGE YAML AS $$ ... $$` with built-in YAML validation |
| Catalog Explorer | UI tab (Builder/Custom expressions) or YAML editor (`<>`); the low-code editor emits `dimensions:` |
| Genie Code | From the metric view editor, from natural language; can import Tableau or Power BI files (AI/BI `/importBI` accepts `.tmdl`) |
| Edit in place | `ALTER VIEW <name> AS $$ <complete YAML> $$` — replaces the whole definition, keeps privileges |
| dbt-databricks 1.12.0+ | `{{ config(materialized='metric_view') }}` model with the YAML body |

Upgrading `0.1` → `1.1` with `ALTER VIEW` removes Unity Catalog comments that are not repeated in YAML `comment:` fields; the YAML editor UI preserves them. YAML `#` comments are unreliable under 1.1 (the agent-metadata page says they are removed on save; the YAML reference shows `ALTER VIEW` keeping them) — keep durable descriptions in `comment:`.

## Common Mistakes

### Wrong

```yaml
version: 0.1                      # offset and inclusive/exclusive require spec 1.1
measures:
  - name: monthly_sales_py
    expr: SUM(sales)
    window:
      - order: month
        range: current
        semiadditive: last
        offset: -12 month
```

### Correct

```yaml
version: 1.1                      # window measures are GA and documented with 1.1 (offset: DBR 18.1+)
measures:
  - name: monthly_sales_py
    expr: SUM(sales)
    window:
      - order: month
        range: current
        semiadditive: last
        offset: -12 month
```

## Related

- [Fundamentals](fundamentals.md)
- [Lifecycle and replace semantics](lifecycle-replace-semantics.md)
- [Window measures](../patterns/window-measures.md)
- [Agent metadata](../patterns/agent-metadata.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (top-level fields, upgrade to 1.1)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/create (prerequisites, Genie Code, import)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage (ownership, collaborative editing, `ALTER VIEW`)
- https://docs.databricks.com/aws/en/uc-semantics/agent-metadata
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/advanced-techniques
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view
- https://docs.databricks.com/aws/en/release-notes/product/2026/october (window measures GA)
- https://docs.databricks.com/aws/en/ai-bi/release-notes/2026 (`/importBI` with `.tmdl`)
- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view
- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view_dbt
