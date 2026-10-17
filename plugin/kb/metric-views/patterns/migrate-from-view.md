# Migrate a Regular View to a Metric View

> **Purpose**: Convert an existing aggregating view into a governed metric view without breaking downstream consumers or losing grants
> **MCP Validated**: 2026-10-17

## When to Use

- An existing `CREATE VIEW ... AS SELECT ... GROUP BY ...` is the de facto source of truth for a KPI and needs governance, Genie Agent access, or safe re-aggregation
- Formalizing a KPI registry / semantic layer where hand-written aggregating views are the known-good baseline to validate against

## Implementation

```sql
-- Existing regular view (keep as-is until parity is confirmed)
-- CREATE VIEW main.sales.orders_summary AS
-- SELECT DATE_TRUNC('MONTH', order_date) AS month,
--        SUM(total_price) AS revenue,
--        COUNT(*) AS order_count
-- FROM main.sales.orders
-- GROUP BY 1;

-- Equivalent metric view under a NEW name: the SELECT's FROM becomes `source`,
-- GROUP BY columns become fields, aggregations become measures
CREATE VIEW main.sales.orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
  version: 1.1
  source: main.sales.orders
  fields:
    - name: Order Month
      expr: DATE_TRUNC('MONTH', order_date)
  measures:
    - name: Revenue
      expr: SUM(total_price)
    - name: Order Count
      expr: COUNT(1)
$$
```

```sql
-- Parity check: numbers must match before repointing consumers
SELECT month, revenue, order_count FROM main.sales.orders_summary ORDER BY month;

SELECT `Order Month`, MEASURE(`Revenue`) AS revenue, MEASURE(`Order Count`) AS order_count
FROM main.sales.orders_metrics GROUP BY ALL ORDER BY ALL;
```

The new metric view sits alongside the original. Only after parity is verified do you grant it, repoint dashboards / Genie Agents / alerts, and drop `orders_summary` (check the Lineage tab first — `DROP VIEW` can't be undone).

If the metric view must take the OLD name (`CREATE OR REPLACE VIEW main.sales.orders_summary WITH METRICS ...`), the replace semantics apply: the caller becomes owner and, below DBR 19 or on an unverified warehouse, grants and tags may be dropped. Prefer the new-name route; if the name must stay, follow [lifecycle-replace-semantics.md](../concepts/lifecycle-replace-semantics.md) (`COPY *` on DBR 18.3+, then re-apply and verify grants). Consumers that run `SELECT *` or select measure columns plainly break either way: metric views need explicit fields and `MEASURE()`.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Migration order | additive | Create under a new name → verify parity → grant → repoint consumers → drop the old view |
| Deployment | file, not heredoc | Keep the DDL in a `.sql` file deployed by a bundle job, or send it through the Statement Execution API; shell heredocs mangle the `$$ ... $$` YAML |
| Statement API | — | `POST /api/2.0/sql/statements` with `statement`, `warehouse_id`, optional `catalog`, `schema`, `wait_timeout`, `on_wait_timeout` |

## Example Usage

```bash
# Build the request body with a JSON encoder (keeps YAML newlines and quotes intact)
python3 -c 'import json,sys; print(json.dumps({"statement": open(sys.argv[1]).read(),
  "warehouse_id": sys.argv[2], "wait_timeout": "50s", "on_wait_timeout": "CONTINUE"}))' \
  orders_metrics.sql "$WAREHOUSE_ID" > orders_metrics_statement.json

databricks api post /api/2.0/sql/statements --json @orders_metrics_statement.json
```

The response carries `statement_id` and `status.state`; poll `GET /api/2.0/sql/statements/{statement_id}` when the state is still `PENDING`/`RUNNING`. For repeatable deploys use [deploy-with-bundles.md](deploy-with-bundles.md).

## See Also

- [Create and Query](create-and-query.md)
- [Lifecycle and replace semantics](../concepts/lifecycle-replace-semantics.md)
- [Deploy with bundles](deploy-with-bundles.md)
- [Fundamentals](../concepts/fundamentals.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/create
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query (`MEASURE()` required, no `SELECT *`)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage (drop, lineage)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view (`OR REPLACE` owner/metadata semantics)
- https://docs.databricks.com/api/workspace/statementexecution/executestatement
- https://docs.databricks.com/aws/en/dev-tools/cli/reference/api-commands (`databricks api post --json @file`)
