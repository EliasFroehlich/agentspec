# Metric Views Quick Reference

> Fast lookup tables. For code examples, see linked files.
> **MCP Validated**: 2026-10-17

## YAML Top-Level Fields

| Field | Required | Notes |
|-------|----------|-------|
| `version` | **Yes** | Spec version: `1.1` (current) or `0.1` (initial); no other value is documented |
| `source` | Yes | Table, view, MV, streaming/foreign/system table, metric view, or SQL query |
| `comment` | No | Spec `1.1` |
| `parameters` | No | Public Preview, DBR 18.2+; view becomes a TVF; blocks materialization |
| `filter` | No | Boolean SQL applied to every query |
| `joins` | No | Star/snowflake; `cardinality`, `rely`; join `source` may be a SQL query |
| `fields` | Conditional | Required if no `measures`; `dimensions` is an accepted synonym |
| `measures` | Conditional | Required if no `fields` |
| `materialization` | No | Serverless + DBR 17.3+; `mode: relaxed` is mandatory |

## Feature / Runtime Matrix

SQL warehouses track the latest Databricks SQL version; the DBR column matters for clusters and jobs.

| Min DBR | Feature (spec `1.1` where marked) |
|---------|-----------------------------------|
| 16.4 | Metric views, `MEASURE()`, `ALTER VIEW ... AS $$yaml$$`, spec `0.1` |
| 17.3 | Snowflake (nested) joins; agent metadata `comment`/`display_name`/`format`/`synonyms` (1.1); `TEMPORARY` metric views; materialization; JDBC/ODBC |
| 18.0 | BI compatibility mode; `REFRESH MATERIALIZED VIEW` |
| 18.1 | `cardinality: one_to_many`, window `offset`, `inclusive`/`exclusive` (1.1); `rely.at_most_one_match`; `MEASURE(x) FILTER (WHERE ...)`; `agg()` |
| 18.2 | `parameters`; wildcards `source.*`, `<join>.* EXCEPT (...)` (1.1) |
| 18.3 | `CREATE OR REPLACE ... COPY *` / `NOCOPY *` |
| 19 | Optional `name` (derived names); unitless window on an integer index (1.1); `OR REPLACE` keeps metadata by default |

No 17.1 or 17.2 milestone exists in the official table. A job cluster on DBR 17.3 LTS gets none of the 18.x/19 rows.

## Measure Cookbook

| Need | Expression |
|------|------------|
| Simple sum | `SUM(revenue)` |
| Distinct count | `COUNT(DISTINCT customer_id)` |
| Composed ratio (preferred) | `MEASURE(total_revenue) / MEASURE(order_count)` |
| Zero-safe ratio | `MEASURE(a) / NULLIF(MEASURE(b), 0)` or `TRY_DIVIDE(SUM(a), SUM(b))` |
| Filtered sum | `SUM(revenue) FILTER (WHERE status = 'OPEN')` |
| Query-time filter on a measure | `MEASURE(total_revenue) FILTER (WHERE status = 'F')` (DBR 18.1+) |
| Year over year | window `range: current` + `offset: -12 month` (see window-measures) |

## Decision Matrix

| Use Case | Choose |
|----------|--------|
| KPI that must re-aggregate across grains | Measure (atomic + composed) in a metric view |
| Same logic with per-caller input (rate, window size) | `parameters` (not materializable) |
| Facts at two grains in one view | `cardinality: one_to_many` or a bridge `source` |
| Non-calendar period (commercial cycle, fiscal week) | Window on a dense integer index (DBR 19+) |
| Change an existing view without losing grants | `ALTER VIEW ... AS $$...$$` |
| Replace on DBR 18.3+ keeping grants/tags/comments | `CREATE OR REPLACE ... COPY *` |
| Version-controlled deploy | Bundle job with `sql_task.file.path` |
| Long DDL from a script | Statement Execution API `POST /api/2.0/sql/statements` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Omit `version:` or write `1.0`/`1.2` | `version: 1.1` |
| Use `0.1` for window measures | `1.1` (GA; `offset`/`inclusive` need it) |
| `SELECT *` on a metric view | List fields; wrap each measure in `MEASURE()` |
| Join the metric view in the query | Wrap it in a CTE, then join |
| Join a table with ARRAY/MAP columns | Flatten/aggregate in the join `source` query |
| Assume `CREATE OR REPLACE` keeps grants/tags | See lifecycle-replace-semantics; re-apply and verify |
| Leave `on:` unquoted when a YAML 1.1 library writes or reads the YAML | `'on':` (YAML 1.1 reads `on` as boolean) |
| Start a YAML value with a backtick | Quote the whole expression: ``expr: '`First Name`'`` |

## Related Documentation

| Topic | Path |
|-------|------|
| Versions and permissions | `concepts/requirements-versions.md` |
| Replace/alter semantics | `concepts/lifecycle-replace-semantics.md` |
| Limitations | `concepts/limitations.md` |
| Full Index | `index.md` |

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/advanced-techniques
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/basic-modeling
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure
- https://docs.databricks.com/api/workspace/statementexecution/executestatement
- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view
- https://yaml.org/type/bool.html
