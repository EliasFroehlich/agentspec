# Window Measures

> **Purpose**: Rolling, cumulative, period-to-date, semiadditive and period-over-period measures (`window:` block), including non-calendar periods on a numeric index
> **MCP Validated**: 2026-10-17

## When to Use

- Trailing/leading windows (7-day active customers), running totals, year-to-date
- Balances or stocks that must not be summed across dates (semiadditive)
- Period-over-period (MoM, YoY) without self-joins, via `offset`
- Business periods that are not calendar dates (fiscal weeks, 4-4-5, commercial cycles) — numeric index

Window measures are GA (2026-10-06) and documented with `version: 1.1`. `offset` and `inclusive`/`exclusive` need DBR 18.1+ and spec 1.1; unitless ranges/offsets on an integer index need DBR 19+.

## Implementation

```yaml
version: 1.1
source: main.sales.monthly_sales
fields:
  - name: date
    expr: order_date
  - name: month                      # hierarchy level defined on the ORDER FIELD `date`,
    expr: DATE_TRUNC('MONTH', date)  # not on the source column (that breaks the link)
  - name: year
    expr: DATE_TRUNC('YEAR', date)
measures:
  - name: t7d_customers              # rolling 7 days before each date (exclusive by default)
    expr: COUNT(DISTINCT customer_id)
    window:
      - order: date
        range: trailing 7 day
        semiadditive: last
  - name: running_total
    expr: SUM(amount)
    window:
      - order: date
        range: cumulative
        semiadditive: last
  - name: ytd_sales                  # cumulative, reset each year: two window specs
    expr: SUM(amount)
    window:
      - order: date
        range: cumulative
        semiadditive: last
      - order: year
        range: current
        semiadditive: last
  - name: monthly_sales
    expr: SUM(amount)
    window:
      - order: month
        range: current
        semiadditive: last
  - name: monthly_sales_py           # DBR 18.1+: same month, prior year
    expr: SUM(amount)
    window:
      - order: month
        range: current
        semiadditive: last
        offset: -12 month            # prefer month/year units over -365 day
  - name: yoy_growth_pct
    expr: (MEASURE(monthly_sales) - MEASURE(monthly_sales_py)) / NULLIF(MEASURE(monthly_sales_py), 0)
```

### Non-calendar periods on a dense integer index (DBR 19+)

```yaml
version: 1.1
source: main.sales.orders_by_period          # one row per period exists (densify gaps upstream)
fields:
  - name: period_year
    expr: period_year                         # INT
  - name: period_number
    expr: period_number                       # INT, e.g. cycle 1..N within the year
  - name: period_index
    expr: period_index                        # dense INT from a calendar table: +1 per period, across years
measures:
  - name: period_sales
    expr: SUM(amount)
    window:
      - order: period_year                    # every column the index depends on gets range: all
        range: all
        semiadditive: last
      - order: period_number
        range: all
        semiadditive: last
      - order: period_index
        range: current
        semiadditive: last
  - name: period_sales_prev                   # previous period, crossing the year boundary
    expr: SUM(amount)
    window:
      - order: period_year
        range: all
        semiadditive: last
      - order: period_number
        range: all
        semiadditive: last
      - order: period_index
        range: current
        offset: -1
        semiadditive: last
  - name: sales_trailing_3_periods
    expr: SUM(amount)
    window:
      - order: period_year
        range: all
        semiadditive: last
      - order: period_number
        range: all
        semiadditive: last
      - order: period_index
        range: trailing 3                     # unitless: three index positions
        semiadditive: last
```

When the number of periods per year varies (per country, 52/53 weeks), build `period_index` from a calendar lookup table; `year * N + period` drifts. For "same period last year" by position, use two specs (`offset: -1` on `period_year`, `range: current` on `period_number`) and no index.

## Configuration

| Key | Required | Values |
|-----|----------|--------|
| `order` | Yes | A deterministic field; group or filter queries by it (or a hierarchy level defined on it) |
| `range` | Yes | `current`, `cumulative`, `trailing <n> [unit] [inclusive\|exclusive]`, `leading <n> [unit] [inclusive\|exclusive]`, `all`; unit ∈ `day(s)`, `month(s)`, `year(s)` |
| `semiadditive` | Yes | `first` or `last` — value returned when `order` is not in `GROUP BY` |
| `offset` | No | Dated `-12 month` on a date/timestamp order field; unitless integer on an integer index (DBR 19+) |
| Parameter as window size | No | Bare integer parameter name: `trailing window_size day` (DBR 18.2+) |

Only the integral type of an index is validated; gaps or repeated values return wrong numbers, not errors. In a materialization, window measures qualify for exact match only.

## Example Usage

```sql
SELECT month, MEASURE(monthly_sales) AS sales, MEASURE(yoy_growth_pct) AS yoy
FROM main.sales.sales_windows
WHERE date >= DATE'2025-01-01'
GROUP BY ALL
ORDER BY month;

-- Numeric index: group by the aligned breakdown columns, not the raw index
SELECT period_year, period_number, MEASURE(period_sales), MEASURE(period_sales_prev)
FROM main.sales.period_windows
GROUP BY ALL;
```

## See Also

- [Requirements & Versions](../concepts/requirements-versions.md)
- [Parameters](parameters.md)
- [Materialization](../concepts/materialization.md)
- [Ratio and Filtered Measures](ratio-and-filtered-measures.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/advanced-techniques (window measures, offset, anchor row, numeric index, date hierarchy)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (window fields, range values, window size as parameter)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
- https://docs.databricks.com/aws/en/release-notes/product/2026/october (GA 2026-10-06)
- https://docs.databricks.com/aws/en/release-notes/product/2026/september (numeric index, 2026-09-14)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization (window measures: exact match only)
