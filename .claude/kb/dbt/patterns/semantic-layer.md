# Semantic Layer

> **Purpose**: Centralized metric definitions with MetricFlow using the latest YAML spec (dbt 1.12+ / v2) — semantic models embedded in `models:`, simple metrics instead of measures — and how it compares with Unity Catalog metric views
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Several BI tools, apps or AI agents must get the same answer for "revenue"
- Metrics should live next to the dbt models that produce their data, versioned in Git
- You run dbt on the dbt platform (hosted Semantic Layer APIs: Starter, Enterprise, Enterprise+ plans) or
  self-host MetricFlow (`mf`, Apache 2.0 since October 2025)
- On Databricks, compare with **Unity Catalog metric views** first (see the table below)

## Implementation

```yaml
# models/marts/finance/_finance__models.yml — latest Semantic Layer spec (dbt 1.12+, dbt v2)
models:
  - name: fct_orders
    semantic_model:
      enabled: true                     # required to turn the model into a semantic model
    agg_time_dimension: order_date      # default time dimension for this model's metrics
    columns:
      - name: order_id
        entity:
          type: primary
          name: order
      - name: customer_id
        entity:
          type: foreign
          name: customer
      - name: order_date
        granularity: day                # replaces type_params.time_granularity
        dimension:
          type: time
      - name: order_status
        dimension:
          type: categorical
      - name: payment_method
        dimension:
          type: categorical
    metrics:                            # simple metrics replace measures
      - name: order_count
        type: simple
        agg: count
        expr: 1
      - name: gross_revenue
        type: simple
        agg: sum
        expr: order_total
      - name: net_revenue
        type: simple
        agg: sum
        expr: net_revenue
      - name: customers_with_orders
        type: simple
        agg: count_distinct
        expr: customer_id

# Metrics built from other metrics live under a top-level metrics: key
metrics:
  - name: revenue_per_customer
    type: ratio
    numerator: net_revenue
    denominator: customers_with_orders
  - name: net_revenue_wow_change
    type: derived
    expr: current_revenue - revenue_7_days_ago
    input_metrics:
      - name: net_revenue
        alias: current_revenue
      - name: net_revenue
        offset_window: 7 days
        alias: revenue_7_days_ago
```

```yaml
# MetricFlow needs a daily (or finer) time spine model in the project
models:
  - name: time_spine_daily
    time_spine:
      standard_granularity_column: date_day
    columns:
      - name: date_day
        granularity: day
```

Validated 2026-10-17: `dbt parse --show-all-deprecations` (dbt-core 1.12.6, no deprecations),
`mf validate-configs` (dbt-metricflow 0.15.0, 0 errors) and `mf query` returned results.

## Configuration

| Latest spec (1.12+, v2, dbt platform "Latest") | Legacy spec (1.6–1.11) |
|-----------------------------------------------|------------------------|
| `semantic_model: {enabled: true}` on the model | Top-level `semantic_models:` with `model: ref(...)` (warning `SemanticModelDeprecated` on v2) |
| `entity:` / `dimension:` on `columns:` | `entities:` / `dimensions:` lists with `expr` |
| `granularity: day` on the column | `type_params.time_granularity` |
| `agg_time_dimension` on the model (override per metric) | `defaults.agg_time_dimension` |
| `type: simple` metrics with `agg` + `expr` in the model | `measures:` + `type: simple` metric pointing at a measure |
| Direct keys: `numerator`/`denominator`, `input_metrics`, `input_metric`, `window`, `base_metric`… | Same keys wrapped in `type_params` |
| **No cross-project references yet** | Supports cross-project refs (dbt Mesh) |

Migrate with `uvx dbt-autofix deprecations --semantic-layer`.

## Example Usage

```bash
python -m pip install "dbt-metricflow[dbt-databricks]"      # self-hosted MetricFlow for your adapter
mf validate-configs                                         # dbt platform: dbt sl validate
mf query --metrics net_revenue,revenue_per_customer \
         --group-by metric_time__month,order__order_status \
         --where "{{ TimeDimension('metric_time', 'day') }} >= '2026-01-01'"
dbt sl query --metrics net_revenue --group-by metric_time__month   # dbt platform CLI / Studio IDE
```

Group-by names are `<entity>__<dimension>` (`order__order_status`) and `metric_time__<grain>`.

### MetricFlow or Unity Catalog metric views?

| Aspect | dbt Semantic Layer (MetricFlow) | Unity Catalog metric views |
|--------|---------------------------------|----------------------------|
| Defined in | dbt model YAML | YAML in `CREATE VIEW … WITH METRICS` (DBR 16.4+; YAML 1.1 and materialization need DBR 17.3+) |
| Queried by | `dbt sl` / `mf`, Semantic Layer APIs and integrations | Any Databricks SQL client with `MEASURE()`; AI/BI dashboards, Genie, alerts, JDBC/ODBC, BI tools |
| Governance | dbt project + dbt platform | Unity Catalog grants, lineage, tags |
| Runs where | dbt platform service or self-hosted `mf` | Natively in the warehouse; optional materialization |
| From dbt | Native | `materialized='metric_view'` in dbt-databricks 1.12.0+ ([adapter](../concepts/dbt-databricks-adapter.md)) |

For Databricks-first stacks whose consumers are Databricks SQL, dashboards and Genie, UC metric views avoid a
second semantic service; see the [metric-views KB](../../metric-views/index.md).

```yaml
# Exposures document downstream consumers (unchanged in the latest spec)
exposures:
  - name: executive_dashboard
    type: dashboard
    maturity: high
    url: https://bi.example.com/exec-dashboard
    depends_on:
      - ref('fct_orders')
    owner:
      name: Finance Analytics
      email: finance-data@example.com
```

## See Also

- [mesh-architecture](../concepts/mesh-architecture.md)
- [dbt-databricks-adapter](../concepts/dbt-databricks-adapter.md)
- [Metric views fundamentals](../../metric-views/concepts/fundamentals.md)

## Sources

- https://docs.getdbt.com/docs/build/latest-metrics-spec
- https://docs.getdbt.com/docs/build/semantic-models
- https://docs.getdbt.com/reference/semantic-model-properties
- https://docs.getdbt.com/reference/metric-properties
- https://docs.getdbt.com/docs/build/metricflow-commands
- https://docs.getdbt.com/docs/build/metricflow-time-spine
- https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl
- https://www.getdbt.com/licenses-faq
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/
- https://github.com/databricks/dbt-databricks/releases
