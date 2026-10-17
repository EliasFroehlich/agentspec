# Agent Metadata (display names, formats, synonyms)

> **Purpose**: Give AI/BI dashboards and Genie Agents business context — labels, display formats and alternative names — directly in the metric view YAML
> **MCP Validated**: 2026-10-17

## When to Use

- Genie Agents must match user vocabulary ("revenue", "sales amount") to the right measure
- Dashboards should show consistent labels and number/currency/date formats without per-widget setup
- A KPI registry generates metric views and must carry descriptions that survive redeploys

Requires **DBR 17.3+ and `version: 1.1`**. AI/BI dashboards populate display names and formats automatically; Genie Agents import synonyms automatically.

## Implementation

```yaml
version: 1.1
source: main.sales.orders
comment: Sales metrics with semantic metadata          # view-level description
fields:
  - name: order_date
    expr: order_date
    comment: Date when the order was placed            # stored as the UC column comment
    display_name: Order Date                           # <= 255 chars
    format:
      type: date
      date_format: year_month_day
      leading_zeros: true
    synonyms:                                          # <= 10 synonyms, <= 255 chars each
      - order time
      - date of order
  - name: customer_segment
    expr: |
      CASE
        WHEN total_price > 100000 THEN 'Enterprise'
        WHEN total_price > 10000 THEN 'Mid-market'
        ELSE 'SMB'
      END
    display_name: Customer Segment
    synonyms: [segment, customer tier]                 # flow style is also valid
measures:
  - name: total_revenue
    expr: SUM(total_price)
    comment: Total revenue from all orders
    display_name: Total Revenue
    format:
      type: currency
      currency_code: USD                               # ISO-4217, required for currency
    synonyms: [revenue, total sales, sales amount]
  - name: order_count
    expr: COUNT(1)
    display_name: Order Count
    format:
      type: number
      decimal_places:
        type: exact
        places: 0
  - name: fulfillment_rate
    expr: COUNT(1) FILTER (WHERE status = 'F') / COUNT(1)
    format:
      type: percentage
```

## Configuration

| Format `type` | Options |
|---------------|---------|
| `number` | `decimal_places: {type: max\|exact\|all, places: 0-10}` (`places` required for `max`/`exact`), `hide_group_separator`, `abbreviation: none\|compact\|scientific` |
| `currency` | `currency_code` (required), `decimal_places`, `hide_group_separator`, `abbreviation` |
| `percentage` | `decimal_places`, `hide_group_separator` |
| `byte` | `decimal_places`, `hide_group_separator` |
| `date` | `date_format: locale_short_month\|locale_long_month\|year_month_day\|locale_number_month\|year_week`, `leading_zeros` |
| `date_time` | `date_format` (+ `no_date`), `time_format: no_time\|locale_hour_minute\|locale_hour_minute_second`, `leading_zeros`; at least one of date/time must be shown |

In the Catalog Explorer editor, `format` is offered only for numeric, date and datetime columns; the editor also sets governed tags on fields and measures.

## Example Usage

```sql
-- Agent metadata comes back per column in the JSON description
DESCRIBE TABLE EXTENDED main.sales.orders_metrics AS JSON;
```

Keep descriptions in YAML `comment:` rather than only in Unity Catalog: `ALTER VIEW ... AS $$ ... $$` removes UC comments that the YAML doesn't repeat, and YAML `#` comments may be stripped on save under spec 1.1.

## See Also

- [Requirements & Versions](../concepts/requirements-versions.md)
- [Lifecycle and replace semantics](../concepts/lifecycle-replace-semantics.md)
- [Ratio and Filtered Measures](ratio-and-filtered-measures.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/agent-metadata
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (field/measure properties, upgrade to 1.1)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/create (editor metadata, governed tags)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query (`DESCRIBE TABLE EXTENDED ... AS JSON`)
