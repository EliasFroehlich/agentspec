# Deploy Metric Views as Code

> **Purpose**: Version-controlled, repeatable deployment of metric views with Declarative Automation Bundles (`sql_task` + `.sql` file), dbt-databricks, or the Statement Execution API — including the grants/tags step that replaces can drop
> **MCP Validated**: 2026-10-17

## When to Use

- Metric view definitions live in Git and must reach dev and prod through CI/CD
- A generator (KPI registry) emits the DDL and a job applies it
- You need the same definition re-applied on a schedule and verified afterwards

Bundles have **no metric view resource type**: the official pattern is a job whose `sql_task` runs the DDL from a `.sql` file on a SQL warehouse.

## Implementation

```yaml
# databricks.yml
bundle:
  name: metric_views

include:
  - resources/*.yml

variables:
  catalog:
    description: Catalog that holds the metric views
  schema:
    description: Schema that holds the metric views
  warehouse_id:
    description: SQL warehouse that runs the DDL (databricks warehouses list)

targets:
  dev:
    mode: development            # pauses the job trigger in dev
    default: true
    variables:
      catalog: main
      schema: ${workspace.current_user.short_name}
      warehouse_id: <dev-warehouse-id>
  prod:
    mode: production
    variables:
      catalog: main
      schema: sales
      warehouse_id: <prod-warehouse-id>
```

```yaml
# resources/orders_metrics.job.yml
resources:
  jobs:
    orders_metrics_view:
      name: orders_metrics_view
      parameters:                       # substituted into {{catalog}} / {{schema}} in the .sql files
        - name: catalog
          default: ${var.catalog}
        - name: schema
          default: ${var.schema}
      tasks:
        - task_key: create_metric_view
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file:
              path: ../src/orders_metrics.metric_view.sql
        - task_key: apply_grants_and_tags      # replaces may drop grants/tags: re-apply every run
          depends_on:
            - task_key: create_metric_view
          sql_task:
            warehouse_id: ${var.warehouse_id}
            file:
              path: ../src/orders_metrics.grants.sql
```

```sql
-- src/orders_metrics.metric_view.sql
USE CATALOG IDENTIFIER({{catalog}});
USE IDENTIFIER({{schema}});

CREATE OR REPLACE VIEW orders_metrics
WITH METRICS
LANGUAGE YAML
AS $$
version: 1.1
comment: Order KPIs
source: main.sales.orders
filter: status = 'confirmed'
fields:
  - name: order_month
    expr: date_trunc('MONTH', order_date)
    comment: Month of order
measures:
  - name: total_orders
    expr: COUNT(1)
    comment: Number of confirmed orders
  - name: total_revenue
    expr: SUM(total_amount)
    comment: Revenue of confirmed orders
$$;
```

```sql
-- src/orders_metrics.grants.sql (idempotent)
USE CATALOG IDENTIFIER({{catalog}});
USE IDENTIFIER({{schema}});
ALTER VIEW orders_metrics SET TAGS ('domain' = 'sales', 'tier' = 'gold');
GRANT SELECT ON orders_metrics TO `data-consumers`;
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `sql_task.file.path` | — | Path to the `.sql` file, relative to the resource YAML |
| Job `parameters` | — | Values substituted for `{{name}}` in the SQL file at run time |
| `mode: development` | — | Pauses the job's trigger in dev; the official example uses a daily `periodic` trigger that only fires after a prod deploy |
| `version:` | required | The official example repo writes `version: 1.0`, which the docs don't list; use `1.1` |
| DDL verb | `CREATE OR REPLACE` | Below DBR 19 (or on an unverified warehouse) it can drop grants/tags; use `COPY *` on 18.3+ or keep the grants task |

### Alternatives

| Route | How |
|-------|-----|
| dbt-databricks ≥ 1.12.0 | Model file `.sql` with `{{ config(materialized='metric_view') }}` followed by the YAML body; dbt issues `CREATE OR REPLACE VIEW ... LANGUAGE YAML` (same replace caveat) |
| Statement Execution API | `POST /api/2.0/sql/statements` with `{"statement", "warehouse_id", "wait_timeout": "0s"-"50s", "on_wait_timeout": "CONTINUE"\|"CANCEL"}`; build the JSON with an encoder, send with `databricks api post /api/2.0/sql/statements --json @body.json` |

## Example Usage

```bash
databricks bundle validate --target dev
databricks bundle deploy --target dev
databricks bundle run orders_metrics_view --target dev

# Verify after the run
#   DESCRIBE TABLE EXTENDED <catalog>.<schema>.orders_metrics AS JSON;   -- YAML in "View Text"
#   SHOW GRANTS ON VIEW <catalog>.<schema>.orders_metrics;
```

## See Also

- [Lifecycle and replace semantics](../concepts/lifecycle-replace-semantics.md)
- [Migrate from a view](migrate-from-view.md)
- [Create and Query](create-and-query.md)

## Sources

- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view (job, `sql_task.file.path`, `{{catalog}}`/`{{schema}}` parameters, daily trigger)
- https://github.com/databricks/bundle-examples/tree/main/knowledge_base/metric_view_dbt (dbt-databricks 1.12.0 `metric_view`)
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources (resource types; `depends_on`)
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-view
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-view (`SET TAGS`)
- https://docs.databricks.com/api/workspace/statementexecution/executestatement
- https://docs.databricks.com/aws/en/dev-tools/cli/reference/api-commands
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability (spec versions)
