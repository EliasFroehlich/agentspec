# Ratio, Composed and Filtered Measures

> **Purpose**: Safe ratio measures (inline and composed with `MEASURE()`), `FILTER (WHERE ...)` measures in YAML and at query time, NULL policy on zero denominators, and distinct-count grain rules
> **MCP Validated**: 2026-10-17

## When to Use

- Any KPI defined as numerator ÷ denominator (margin, penetration, average ticket) that must roll up correctly at every field the caller picks
- Metrics that count only a subset of rows ("open orders" out of "all orders") without a separate view per status
- Distinct counts (`COUNT(DISTINCT ...)`) that must stay pinned to the grain they are valid at

## Implementation

### Approach A — inline aggregates (valid; self-contained)

```yaml
version: 1.1
source: main.finance.transactions
fields:
  - name: Department
    expr: department_name
  - name: Quarter
    expr: DATE_TRUNC('QUARTER', transaction_date)
measures:
  - name: Profit Margin                        # SUM/SUM re-aggregates safely
    expr: (SUM(revenue) - SUM(cost)) / SUM(revenue)
  - name: Safe Profit Margin                   # NULL (not an error or 0) when revenue is 0
    expr: TRY_DIVIDE(SUM(revenue) - SUM(cost), SUM(revenue))
  - name: Open Orders                          # filtered count
    expr: COUNT(1) FILTER (WHERE status = 'OPEN')
  - name: Fulfillment Rate                     # filtered ratio
    expr: COUNT(1) FILTER (WHERE status = 'FULFILLED') * 1.0 / COUNT(1)
```

### Approach B — composed measures (recommended by the docs)

```yaml
version: 1.1
source: main.finance.transactions
fields:
  - name: Department
    expr: department_name
measures:
  # atomic measures first
  - name: total_revenue
    expr: SUM(revenue)
  - name: total_cost
    expr: SUM(cost)
  - name: total_orders
    expr: COUNT(1)
  - name: fulfilled_orders
    expr: COUNT(1) FILTER (WHERE status = 'FULFILLED')
  - name: unique_employees
    expr: COUNT(DISTINCT employee_id)          # valid only at the grain of each query
  # composed measures reference earlier measures with MEASURE()
  - name: profit_margin
    expr: (MEASURE(total_revenue) - MEASURE(total_cost)) / NULLIF(MEASURE(total_revenue), 0)
    format:
      type: percentage                         # spec 1.1, DBR 17.3+
  - name: fulfillment_rate
    expr: MEASURE(fulfilled_orders) / MEASURE(total_orders)
    format:
      type: percentage
  - name: revenue_per_employee
    expr: MEASURE(total_revenue) / NULLIF(MEASURE(unique_employees), 0)
```

If `total_revenue` changes (for example, to exclude tax), every composed measure follows automatically. Don't repeat `SUM(a) / COUNT(b)` when both measures already exist.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `FILTER (WHERE ...)` in YAML | — | Conditional aggregation per measure |
| `MEASURE(m) FILTER (WHERE cond)` in a query | — | DBR 18.1+ (errors below); `cond` is ANDed into every aggregate of `m`, recursively through referenced measures; for window measures it applies after the window aggregation |
| `TRY_DIVIDE(a, b)` | — | Returns `NULL` when `b = 0` |
| `NULLIF(b, 0)` | — | The divide-by-zero guard used in the official composed-measure examples |
| `COUNT(DISTINCT ...)` | — | Never sum a distinct count across a coarser grain; in materializations it is non-additive (exact match only) |
| Measures with several aggregates | — | `SUM(x) - SUM(y)` is valid but not rollup-eligible in a materialization; single-aggregate atoms stay rollup-eligible (rollup of a composed measure is not documented) |

## Example Usage

```sql
SELECT
  Department,
  MEASURE(profit_margin)    AS margin,
  MEASURE(fulfillment_rate) AS fulfillment_rate,
  MEASURE(total_revenue) FILTER (WHERE Department <> 'Internal') AS external_revenue  -- DBR 18.1+
FROM main.finance.efficiency_metrics
GROUP BY ALL
ORDER BY ALL
```

## See Also

- [Limitations](../concepts/limitations.md)
- [Materialization — additive measures](../concepts/materialization.md)
- [Agent metadata — format](agent-metadata.md)
- [Create and Query](create-and-query.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/advanced-techniques (composability, best practices, `NULLIF`)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/basic-modeling (measure patterns)
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure (`FILTER` clause behavior, DBR 18.1)
- https://docs.databricks.com/aws/en/sql/language-manual/functions/try_divide
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization (additive measures)
- https://docs.databricks.com/aws/en/uc-semantics/agent-metadata (format types)
