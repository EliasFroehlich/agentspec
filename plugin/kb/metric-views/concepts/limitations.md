# Limitations

> **Purpose**: Hard constraints that shape metric view design and query code — know these before modeling
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Metric views trade some flexibility of standard views for governed metrics that re-aggregate safely. The constraints below are documented behavior (or a documented gap) and drive how a metric view is modeled and queried from day one.

## The Concept

```sql
-- INVALID: joining a metric view at query time
SELECT o.*, c.region FROM orders_metrics o JOIN dim_customer c ON ...

-- VALID: wrap the metric view query in a CTE, then join the CTE
WITH m AS (
  SELECT customer_id, MEASURE(total_revenue) AS revenue
  FROM orders_metrics GROUP BY customer_id
)
SELECT c.segment, SUM(m.revenue) FROM m JOIN dim_customer c USING (customer_id) GROUP BY ALL;

-- INVALID: SELECT * (measures need MEASURE())
SELECT * FROM orders_metrics;

-- INVALID: MEASURE() as a window function
SELECT region, MEASURE(total_revenue) OVER (PARTITION BY region) FROM orders_metrics;
-- Model it as a window measure in the YAML instead (patterns/window-measures.md)
```

## Quick Reference

| Limitation | Detail |
|------------|--------|
| No query-time joins | Join the result of a CTE over the metric view; declare reusable joins in YAML `joins:` |
| No `SELECT *` | List every field and wrap every measure in `MEASURE()`. YAML wildcards (`source.*`, DBR 18.2+) only expand columns at create time |
| `MEASURE()` is not a window function | `OVER` is not allowed; use a `window:` measure |
| Outer aggregation over `MEASURE()` | Not documented (neither allowed nor forbidden) — don't build on it; compose measures in YAML |
| ARRAY/MAP in joined tables | Not allowed; explode or aggregate them in the join's `source` SQL query |
| Many-to-many joins | The first matching row of the joined table is used, silently |
| `rely.at_most_one_match: true` | Not validated at runtime; a fan-out returns wrong `SUM`/`COUNT` |
| One-to-many joins (DBR 18.1+) | Their columns can't be fields; descendants must also be `one_to_many`; materializations fall back to exact match |
| Data profiling | Not supported on metric views (formerly Lakehouse Monitoring) |
| Data Classification | Scanning UC views is Beta (2026-09-15); the release note doesn't mention metric views — don't assume coverage |
| `DEFAULT COLLATION`, `WITH SCHEMA` binding | Not supported for metric views |
| `ALTER VIEW ... AS query` | Not supported; metric views take `ALTER VIEW ... AS $$yaml$$` |
| Parameters (DBR 18.2+) | Can't be materialized; defaults must be castable, can't reference other parameters; UC metric views with parameters can't be extended in AI/BI dashboards |
| Materialization | No RLS/column masks/ABAC on the view or sources, no invoker-dependent expressions (`current_user()`, `is_member()`), no `TRIGGER ON UPDATE`, no owner change after creation, no group ownership |
| Editing | Only the owner edits the definition; group ownership enables collaboration (not for materialized metric views) |
| String fields | Always `STRING`; `CHAR(n)` padding is lost |
| Window `order` field | Must be deterministic (no `rand()`, `uuid()`, `current_timestamp()`) |
| YAML values | A value can't start with a backtick: quote the expression (``expr: '`First Name`'``); double-quote expressions that contain colons |
| YAML 1.1 tooling | Unquoted `on`/`off`/`yes`/`no` are booleans in YAML 1.1; the official examples quote `'on':` — do the same when a script generates or parses the YAML |

## Common Mistakes

### Wrong

```sql
-- Relying on an undeclared source column in consumers (dashboards, Genie Agents)
SELECT `Customer Region`, MEASURE(total_revenue) FROM orders_metrics GROUP BY ALL
-- `Customer Region` exists nowhere unless it is declared as a field: unresolved column
```

### Correct

```yaml
fields:
  - name: Customer Region
    expr: customer.region          # declare every attribute consumers group or filter on
    synonyms: [region]             # metadata only exists on declared fields (spec 1.1)
```

The official query page groups by a raw source column (`o_orderpriority`) in one example, so undeclared columns may resolve in plain SQL; that is not a contract to build on. Only declared fields carry `comment`, `display_name`, `format` and `synonyms`, which AI/BI dashboards and Genie Agents consume.

## Design Consequences

- Every attribute that the UI or a Genie Agent needs must be a declared field.
- Cross-grain rollups are recomputed from atomic measures (`SUM`, `COUNT`) and composed with `MEASURE()`; don't pre-divide in the source.
- Multi-grain facts (targets, baskets) belong in `one_to_many` joins or a bridge source, not as extra columns of the fact (they inflate `SUM`).

## Related

- [Fundamentals](fundamentals.md)
- [Materialization](materialization.md)
- [Lifecycle and replace semantics](lifecycle-replace-semantics.md)
- [Joins / Star Schema](../patterns/joins-star-schema.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/basic-modeling (ARRAY/MAP, many-to-many)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/joins (`rely`, one-to-many)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (one-to-many rules, column name references, window order)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage (data profiling, collaborative editing)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization (known restrictions)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/use-parameters
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure (`OVER` not allowed)
- https://docs.databricks.com/aws/en/release-notes/product/2026/september (Data Classification of views, Beta)
- https://docs.databricks.com/aws/en/ai-bi/release-notes/2026 (extending UC metric views)
- https://yaml.org/type/bool.html
