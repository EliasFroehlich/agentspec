# Materialization

> **Purpose**: How metric view materialization pre-computes aggregations, how queries are rewritten to use it, and what blocks it
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Materialization declares materialized views inside the metric view YAML. Databricks creates a managed Lakeflow pipeline that builds and refreshes them, and the optimizer rewrites `SELECT ... FROM <metric_view>` to read the best materialization (fast path) or the source (fallback). Requirements: serverless compute enabled for Lakeflow pipelines and a warehouse/compute on DBR 17.3+. Refreshes are billed as Lakeflow pipelines usage.

## The Concept

```yaml
version: 1.1
source: main.sales.orders_enriched
filter: revenue > 0                 # no invoker-dependent expressions (current_user(), is_member())
fields:
  - name: category
    expr: category
  - name: order_date
    expr: order_date
measures:
  - name: total_revenue
    expr: SUM(revenue)              # additive: rollup-eligible
  - name: number_of_suppliers
    expr: COUNT(DISTINCT supplier_id)   # non-additive: exact match only
materialization:
  schedule: every 6 hours           # same syntax as the MV SCHEDULE clause; no TRIGGER ON UPDATE
  mode: relaxed                     # required, only value
  materialized_views:
    - name: baseline
      type: unaggregated            # at most one per metric view
    - name: daily_category
      type: aggregated
      dimensions: [order_date, category]   # `dimensions:` here even if the view uses `fields:`
      measures: [total_revenue, number_of_suppliers]
      cluster_by:
        cols: [order_date]
```

## Quick Reference

| Topic | Rule |
|-------|------|
| Types | `aggregated` (named fields + measures) or `unaggregated` (source, joins, filter and fields; one per view) |
| Rewrite order | Exact match → rollup match → unaggregated match → source |
| Exact match | Query `GROUP BY` equals the materialization dimensions; measures are a subset |
| Rollup match | Coarser grain; every measure additive; deterministic filters |
| Additive | `SUM`, `COUNT`, `MIN`, `MAX`, `BIT_AND/OR/XOR`, `BOOL_AND/OR`; exactly one aggregate; deterministic `FILTER` |
| Not additive | Any `DISTINCT`, `MEDIAN`, percentiles, multiple aggregates in one expr, window measures |
| One-to-many joins | Every materialization drops to exact match only |
| Relaxed mode | Doesn't check freshness, SQL settings (`TIMEZONE`, `ANSI_MODE`) or determinism |
| Refresh | Schedule, pipeline UI, or `REFRESH MATERIALIZED VIEW <metric_view>` (DBR 18.0+); incremental when possible |
| Create/alter | Definition changes immediately; materializations refresh asynchronously; a schedule change doesn't trigger a refresh |
| Verify use | `EXPLAIN EXTENDED` shows `__materialization_mat_<pipeline id>___metric_view_mat_` and the materialization name; query profile insight |
| Inspect pipeline | `DESCRIBE EXTENDED <metric_view>` → "Refresh Information" |

## Blockers (create, alter or refresh fails)

- Row-level security, column masks or ABAC on the view or any source.
- Invoker-dependent expressions (`current_user()`, `is_member()`): error `METRIC_VIEW_MATERIALIZATION_WITH_INVOKER_DEPENDENT_EXPRESSIONS_NOT_SUPPORTED`.
- `parameters` in the definition.
- `TRIGGER ON UPDATE` in the schedule.
- Ownership: no group owner, and the owner can't change after a materialization exists.

## Common Mistakes

### Wrong

```yaml
materialization:
  mode: relaxed
  materialized_views:
    - name: by_second
      type: aggregated
      dimensions: [event_timestamp_ms, region]   # near-unique grain: storage cost, no benefit
      measures: [unique_customers]               # COUNT(DISTINCT): only exact match
```

### Correct

```yaml
materialization:
  schedule: every 6 hours            # align with the upstream pipeline refresh
  mode: relaxed
  materialized_views:
    - name: by_day_region
      type: aggregated
      dimensions: [event_day, region]  # most detailed grain queries need; serves coarser rollups
      measures: [total_revenue, order_count]
```

Without a schedule the pipeline runs once at creation and then only on manual refresh, so data goes stale.

## Related

- [Limitations](limitations.md)
- [Requirements & Versions](requirements-versions.md)
- [Ratio and Filtered Measures](../patterns/ratio-and-filtered-measures.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (materialization fields)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
