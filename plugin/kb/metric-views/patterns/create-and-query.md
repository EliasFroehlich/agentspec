# Create and Query a Metric View

> **Purpose**: The canonical `CREATE VIEW ... WITH METRICS LANGUAGE YAML` DDL, the matching `SELECT`/`MEASURE()` query shape, and how to inspect the definition
> **MCP Validated**: 2026-10-17

## When to Use

- Defining a new governed metric layer over a fact table, view, metric view, or SQL query
- Replacing ad-hoc `GROUP BY` queries that get copy-pasted with slightly different aggregations each time
- Any measure that AI/BI dashboards, Genie Agents, alerts and SQL users must compute identically

## Implementation

```sql
CREATE OR REPLACE VIEW main.sales.orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
  version: 1.1
  source: main.sales.orders
  comment: "Orders KPIs for sales analysis"
  filter: order_date > '2020-01-01'          # optional global WHERE, applied to every query
  fields:
    - name: Order Month
      expr: DATE_TRUNC('MONTH', order_date)
      comment: "Month of order"
    - name: Order Status
      expr: CASE
        WHEN status = 'O' THEN 'Open'
        WHEN status = 'P' THEN 'Processing'
        WHEN status = 'F' THEN 'Fulfilled'
        END
      comment: "Human-readable order status"
  measures:
    - name: Order Count
      expr: COUNT(1)
    - name: Total Revenue
      expr: SUM(total_price)
      comment: "Sum of total price"
    - name: Unique Customers
      expr: COUNT(DISTINCT customer_id)
    - name: Revenue per Customer
      expr: MEASURE(`Total Revenue`) / NULLIF(MEASURE(`Unique Customers`), 0)
      comment: "Average revenue per unique customer"
$$
```

On an existing view, prefer `ALTER VIEW ... AS $$ ... $$` (keeps grants) — see [lifecycle-replace-semantics.md](../concepts/lifecycle-replace-semantics.md).

```sql
-- Fields are selected plain; measures MUST be wrapped in MEASURE() (agg() on DBR 18.1+)
SELECT
  `Order Month`,
  `Order Status`,
  MEASURE(`Total Revenue`) AS total_revenue,
  MEASURE(`Order Count`)   AS order_count,
  MEASURE(`Total Revenue`) FILTER (WHERE `Order Status` = 'Fulfilled') AS fulfilled_revenue  -- DBR 18.1+
FROM main.sales.orders_metrics
WHERE extract(year FROM `Order Month`) = 2024   -- WHERE filters rows by fields
GROUP BY ALL
ORDER BY ALL
```

To filter by a measure's value or combine with another table, wrap the metric view query in a CTE and filter or join the CTE (the documented pattern for joins).

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `version` | none (required) | `1.1`; agent metadata needs DBR 17.3+ |
| `fields` / `dimensions` | — | Equivalent keywords; `fields` preferred |
| `filter` | none | Global boolean expression applied to every query |
| `name` | required below DBR 19 | DBR 19+ derives it from `expr` (`SUM(o_totalprice)` → `` `SUM(o_totalprice)` ``); prefer explicit names |
| Wildcards | — | `expr: source.*` / `customer.* EXCEPT (id)` in `fields` (DBR 18.2+, spec 1.1); no `name` or metadata on a wildcard |
| Names with spaces | — | Backtick-quote in every `SELECT`/`WHERE`/`GROUP BY`/`MEASURE()` reference |
| Multi-line `expr` | — | `expr: |` block scalar, lines indented two spaces past `expr` |

## Example Usage

```sql
GRANT SELECT ON main.sales.orders_metrics TO `data-consumers`;

-- Full YAML (incl. joins and agent metadata) in the "View Text" field
DESCRIBE TABLE EXTENDED main.sales.orders_metrics AS JSON;

-- DDL with the fully qualified three-part name (supported for metric views since DBSQL 2026.15)
SHOW CREATE TABLE main.sales.orders_metrics;
```

## See Also

- [Ratio and Filtered Measures](ratio-and-filtered-measures.md)
- [Joins / Star Schema](joins-star-schema.md)
- [Agent metadata](agent-metadata.md)
- [Deploy with bundles](deploy-with-bundles.md)
- [Fundamentals](../concepts/fundamentals.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/create
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (fields, derived names, wildcards, formatting)
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure (`FILTER`, `agg`)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view
- https://docs.databricks.com/aws/en/sql/release-notes/2026 (`SHOW CREATE TABLE` supports metric views)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage (`GRANT SELECT`)
