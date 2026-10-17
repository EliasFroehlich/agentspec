# Testing Framework

> **Purpose**: dbt test types — generic data tests (`data_tests:` + `arguments:` + `config:`), singular tests, custom generic tests, unit tests (v1.8+), severity, stored failures
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

dbt has two families of tests. **Data tests** query built models and fail when they return rows: generic
tests declared in YAML (`unique`, `not_null`, `accepted_values`, `relationships`, package tests), singular
SQL files in `tests/`, and custom generic tests written as `{% test %}` macros. **Unit tests** (v1.8+)
validate a SQL model's logic on static input rows before it is materialized. YAML syntax changed twice:
`tests:` → `data_tests:` (1.8, `tests:` remains an alias) and test inputs moved under `arguments:` with
framework options under `config:` (1.10.5; required on dbt v2).

## The Concept

```yaml
# models/staging/stripe/_stripe__models.yml
models:
  - name: stg_stripe__payments
    description: "Staging model for Stripe payments"
    columns:
      - name: payment_id
        description: "Primary key"
        data_tests:
          - unique
          - not_null
      - name: payment_status
        data_tests:
          - accepted_values:
              arguments:
                values: ['succeeded', 'pending', 'failed', 'refunded']
              config:
                severity: warn
      - name: order_id
        data_tests:
          - relationships:
              arguments:
                to: ref('stg_orders')
                field: order_id
```

## Unit Tests (v1.8+)

```yaml
# models/marts/finance/_finance__unit_tests.yml
unit_tests:
  - name: test_order_total_calculation
    description: "order_total = quantity * unit_price; net_revenue subtracts discount"
    model: fct_orders
    given:                                   # mock every ref()/source() the model reads
      - input: ref('stg_stripe__orders')
        rows:
          - {order_id: 1, quantity: 3, unit_price: 10.00, discount_amount: 5.00}
          - {order_id: 2, quantity: 1, unit_price: 25.00, discount_amount: 0}
    expect:
      rows:
        - {order_id: 1, order_total: 30.00, net_revenue: 25.00}
        - {order_id: 2, order_total: 25.00, net_revenue: 25.00}
```

## Quick Reference

| Test type | Location | Returns | Use when | Min version |
|-----------|----------|---------|----------|-------------|
| Generic (built-in / package) | model YAML `data_tests:` | Failing rows | Standard column checks | `data_tests:` 1.8; `arguments:` 1.10.5 |
| Singular | `tests/*.sql` | Failing rows | Multi-table business logic | All |
| Custom generic | `tests/generic/*.sql` or `macros/` | Failing rows | Reusable parameterized checks | All |
| **Unit test** | model YAML `unit_tests:` | Pass/fail on expected rows | Logic on static inputs | **1.8** |
| Source freshness | source `config: freshness:` | warn/error | Upstream staleness | `config:` form 1.9 |

| Key | Goes under | Examples |
|-----|-----------|----------|
| Test inputs | `arguments:` | `values`, `to`, `field`, `min_value`, `interval` |
| Framework options | `config:` | `severity`, `where`, `store_failures`, `store_failures_as`, `limit`, `error_if`, `warn_if`, `tags`, `meta` |

### Unit Test Limitations

- SQL models only (not Python models); models in the current project only
- No `materialized view` models; no recursive SQL or introspective queries
- Alias table names to unit test `JOIN` logic; versioned models run against all versions by default
- dbt v2 can run unit tests locally on DuckDB (`config: {compute: local}`, experimental flag
  `DBT_ENGINE_EXPERIMENTAL_LOCAL_UNIT_TESTS=true`) — Snowflake and BigQuery projects only, not Databricks

## Common Mistakes

### Wrong

```yaml
# Pre-1.10.5 shape: inputs and severity mixed at the test level (validation error on dbt v2)
models:
  - name: fct_orders
    tests:
      - dbt_utils.recency:
          datepart: hour
          field: updated_at
          interval: 24
          severity: warn
```

### Correct

```yaml
models:
  - name: fct_orders
    data_tests:
      - dbt_utils.recency:
          arguments:
            datepart: hour
            field: updated_at
            interval: 24
          config:
            severity: warn
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: order_total
        data_tests:
          - dbt_utils.accepted_range:
              arguments:
                min_value: 0
                inclusive: true
```

## Related

- [generic-tests](../patterns/generic-tests.md)
- [fusion-engine](fusion-engine.md) — dbt v2 strictness and `dbt-autofix`
- [model-types](model-types.md)

## Sources

- https://docs.getdbt.com/docs/build/data-tests
- https://docs.getdbt.com/reference/resource-properties/data-tests
- https://docs.getdbt.com/reference/data-test-configs
- https://docs.getdbt.com/docs/build/unit-tests
- https://docs.getdbt.com/docs/build/sources
- https://docs.getdbt.com/reference/deprecations
