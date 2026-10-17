# Parameterized Metric Views

> **Purpose**: Bind values at query time so one metric view serves many variants (rates, thresholds, window sizes) instead of duplicated views
> **MCP Validated**: 2026-10-17

## When to Use

- The same KPI logic with a caller-supplied input (discount rate, exchange-rate scenario, threshold)
- A rolling window whose size the caller chooses (`trailing window_size day`)
- AI/BI dashboards where a filter widget should drive a value inside the measure logic

Status: **Public Preview**. Defining and querying need DBR 18.2+ (or a SQL warehouse on a version that supports it). A metric view that defines parameters **can't be materialized**, and AI/BI dashboards can't extend it locally.

## Implementation

```sql
CREATE OR REPLACE VIEW main.sales.sales_scenarios
WITH METRICS
LANGUAGE YAML
AS $$
version: 1.1
source: main.sales.orders
parameters:
  - name: discount              # referenced by name, like a constant
    data_type: double
    default: 0
  - name: window_size           # integral type required when used as a window size
    data_type: int
    default: 7                  # once one parameter has a default, all that follow must have one
fields:
  - name: product
    expr: product
  - name: order_date
    expr: order_date
measures:
  - name: total_sales
    expr: SUM(amount)
  - name: discounted_sales
    expr: SUM((1 - discount) * amount)
  - name: rolling_customers
    expr: COUNT(DISTINCT customer_id)
    window:
      - order: order_date
        range: trailing window_size day   # bare parameter name; the unit stays literal
        semiadditive: last
$$
```

## Configuration

| Field | Required | Rule |
|-------|----------|------|
| `name` | Yes | Used in expressions and as the named argument; can't be a window keyword (`trailing`, `leading`, `day`, `month`, `year`, `inclusive`, `exclusive`, `offset`) when used as a window size |
| `data_type` | Yes | SQL type (`double`, `int`, `string`, `date`, ...) |
| `default` | No | Castable to `data_type`; no reference to another parameter; no subquery |
| Where usable | — | Anywhere a constant SQL expression is valid: `filter`, field and measure `expr`, `source`, joins, window `range`/`offset` |
| Window size | — | Bare name only (`trailing window_size day`, not `window_size + 1`); no sign before the name — put the sign in the value |
| Change parameters | — | `ALTER VIEW ... AS $$ ... $$` with the new YAML adds, drops or changes them |

## Example Usage

```sql
-- Named arguments
SELECT product, MEASURE(discounted_sales)
FROM main.sales.sales_scenarios(discount => 0.15, window_size => 28)
GROUP BY product;

-- Positional arguments (definition order)
SELECT product, MEASURE(discounted_sales) FROM main.sales.sales_scenarios(0.15) GROUP BY product;

-- Defaults: both forms are equivalent
SELECT product, MEASURE(total_sales) FROM main.sales.sales_scenarios() GROUP BY product;
SELECT product, MEASURE(total_sales) FROM main.sales.sales_scenarios GROUP BY product;
```

In an AI/BI dashboard, reference the metric view in a SQL dataset as a table-valued function and pass a dashboard parameter connected to a filter widget. To see which parameters a view defines, read the `parameters` block from `DESCRIBE TABLE EXTENDED <view> AS JSON`.

## See Also

- [Window measures](window-measures.md)
- [Materialization](../concepts/materialization.md)
- [Limitations](../concepts/limitations.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/use-parameters
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (parameters, pass a window size as a parameter)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view (Public Preview, DBR 18.2)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view (re-specifying parameters)
- https://docs.databricks.com/aws/en/ai-bi/release-notes/2026 (UC metric views with parameters can't be extended)
