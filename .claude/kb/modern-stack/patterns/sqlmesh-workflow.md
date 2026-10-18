# SQLMesh Workflow

> **Purpose**: SQLMesh project setup, MODEL definitions with audits, the plan cycle (dev → prod), virtual environments, scheduler and linter
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Starting a new SQLMesh project, or running an existing dbt project through SQLMesh
- Defining models with automatic change categorization and minimal backfills
- Testing changes in a virtual dev environment before promoting to prod
- Running data quality audits as part of model execution

Tested end to end on SQLMesh 0.236.3 with the DuckDB engine (2026-10-17).

## Implementation

```bash
# ============================================================
# Project setup
# ============================================================
pip install sqlmesh                 # add extras per engine, e.g. "sqlmesh[dbt]" for dbt projects
sqlmesh init duckdb                 # ENGINE argument; omit it for the interactive scaffold
# Existing dbt project: run from the dbt project root (creates sqlmesh.yaml with a start date)
sqlmesh init -t dbt                 # templates: dbt, dlt, default, empty

# Generated layout:
# config.yaml  models/  seeds/  audits/  tests/  macros/
```

```yaml
# config.yaml
gateways:
  local:
    connection:
      type: duckdb
      database: data/warehouse.db
  prod:
    connection:
      type: snowflake
      account: myorg-myaccount
      database: analytics
      warehouse: transform_wh

default_gateway: local

model_defaults:
  dialect: duckdb
  start: '2026-01-01'          # required for backfills

linter:
  enabled: true                # linter is off by default
  rules: ["ambiguousorinvalidcolumn", "invalidselectstarexpansion"]
```

```sql
-- ============================================================
-- models/staging/stg_orders.sql
-- ============================================================
MODEL (
    name staging.stg_orders,
    kind INCREMENTAL_BY_TIME_RANGE (
        time_column order_date,
        batch_size 7                       -- max intervals per backfill task
    ),
    grain (order_id),
    cron '@daily',
    owner 'data-team',
    tags ['staging', 'orders'],
    audits (
        not_null(columns := (order_id, customer_id, order_date)),
        unique_values(columns := (order_id)),
        accepted_range(column := net_amount, min_v := 0)
    )
);

SELECT
    order_id,
    customer_id,
    order_date,
    CAST(net_amount AS DECIMAL(18, 2)) AS net_amount,
    status
FROM raw.orders
WHERE order_date BETWEEN @start_ds AND @end_ds;   -- interval macros (@start_date/@end_date also exist)

-- ============================================================
-- models/marts/mart_revenue.sql — @DEF goes AFTER the MODEL block
-- ============================================================
MODEL (
    name marts.mart_revenue,
    kind FULL,
    grain (customer_id, revenue_month),
    cron '@daily',
    audits (
        not_null(columns := (customer_id, total_revenue))
    )
);

@DEF(revenue_threshold, 1000);

SELECT
    customer_id,
    DATE_TRUNC('month', order_date) AS revenue_month,
    SUM(net_amount) AS total_revenue,
    COUNT(DISTINCT order_id) AS order_count
FROM staging.stg_orders
WHERE net_amount >= @revenue_threshold
GROUP BY customer_id, DATE_TRUNC('month', order_date);
```

```bash
# ============================================================
# Plan cycle (there is no `apply` command)
# ============================================================
sqlmesh lint                      # run linter rules
sqlmesh plan                      # first plan targets prod; prompts "Apply - Backfill Tables [y/n]"
sqlmesh plan --auto-apply         # same, without the prompt

# Edit a model, then plan into a virtual dev environment
sqlmesh plan dev
# New environment `dev` will be created from `prod`
# Directly Modified: marts__dev.mart_revenue (Non-breaking)
# Models needing backfill: marts__dev.mart_revenue: [full refresh]
# Apply - Backfill Tables [y/n]:

# Query dev: environment views use the <schema>__<env> naming
sqlmesh fetchdf "SELECT * FROM marts__dev.mart_revenue LIMIT 10"

# Promote: plan prod again; reused work becomes a virtual update
sqlmesh plan                      # prompts "Apply - Virtual Update [y/n]"

# Forward-only change (keep existing prod tables, no backfill)
sqlmesh plan --forward-only

# ============================================================
# Operate
# ============================================================
sqlmesh run                       # run models whose cron interval is due
sqlmesh audit                     # run audits (optionally --start/--end)
sqlmesh test                      # unit tests in tests/*.yaml
sqlmesh ui                        # browser UI incl. column-level lineage
sqlmesh table_diff prod:dev marts.mart_revenue   # data diff between environments
```

## Configuration

| Model Kind | Description | Backfill |
|------------|-------------|----------|
| `FULL` | Rebuild the whole table | Full on change |
| `INCREMENTAL_BY_TIME_RANGE` | Process by time interval | Only missing/affected intervals |
| `INCREMENTAL_BY_UNIQUE_KEY` | Upsert by key | Affected intervals |
| `INCREMENTAL_BY_PARTITION` / `INCREMENTAL_UNMANAGED` | Partition replace / append-only | Kind-specific |
| `SCD_TYPE_2_BY_TIME` / `SCD_TYPE_2_BY_COLUMN` | Slowly changing dimensions | Kind-specific |
| `VIEW` / `EMBEDDED` | Logical view / inlined subquery | N/A |
| `SEED` | Static CSV data | On change |
| `EXTERNAL` / `MANAGED` | Externally managed / engine-managed tables | N/A |

| Built-in audit | Arguments |
|----------------|-----------|
| `not_null` | `columns := (a, b)` |
| `unique_values` | `columns := (a)` |
| `accepted_range` | `column := c, min_v := 0, max_v := 100, inclusive := true` |

## See Also

- [sqlmesh](../concepts/sqlmesh.md)
- [duckdb-patterns](../patterns/duckdb-patterns.md)
- [analytics-engineering](../concepts/analytics-engineering.md)

## Sources

- https://sqlmesh.readthedocs.io/en/latest/reference/cli/
- https://sqlmesh.readthedocs.io/en/latest/quickstart/cli/
- https://sqlmesh.readthedocs.io/en/latest/integrations/dbt/
- https://sqlmesh.readthedocs.io/en/latest/concepts/models/model_kinds/
- https://sqlmesh.readthedocs.io/en/latest/concepts/audits/
- https://sqlmesh.readthedocs.io/en/latest/concepts/macros/sqlmesh_macros/
- https://sqlmesh.readthedocs.io/en/latest/concepts/environments/
- https://sqlmesh.readthedocs.io/en/latest/concepts/plans/
- https://sqlmesh.readthedocs.io/en/latest/guides/linter/
- https://sqlmesh.readthedocs.io/en/latest/faq/faq/
