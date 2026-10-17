# dbt Testing

> **Purpose**: Data tests in dbt 1.10.5+/v2 syntax (`data_tests:` + `arguments:` + `config:`), singular and custom generic tests, model contracts with constraints, source freshness, stored failures, unit tests
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Validating data quality as part of `dbt build`
- Enforcing primary-key uniqueness, not-null and accepted values on models
- Checking referential integrity between models and freshness of sources
- Generating tests from an ODCS contract (`datacontract dbt sync`)

## Implementation

```yaml
# models/marts/finance/_finance__models.yml
models:
  - name: fct_orders
    description: "Order fact table — one row per order"
    config:
      materialized: table              # constraints are ignored on views
      contract:
        enforced: true                 # column names + data_type checked at build
    columns:
      - name: order_id
        data_type: string
        constraints:
          - type: not_null             # enforced on Databricks (applied after build)
          - type: primary_key          # informational on Databricks
        data_tests:
          - unique
          - not_null
      - name: customer_id
        data_type: string
        data_tests:
          - not_null
          - relationships:
              arguments:               # test inputs (1.10.5+; required on dbt v2)
                to: ref('dim_customers')
                field: customer_id
      - name: amount
        data_type: decimal(12,2)
        data_tests:
          - dbt_utils.accepted_range:
              arguments:
                min_value: 0
                inclusive: true
      - name: status
        data_type: string
        data_tests:
          - accepted_values:
              arguments:
                values: ['pending', 'completed', 'cancelled', 'refunded']
              config:                  # framework options
                severity: error
                where: "order_date >= '2024-01-01'"
                store_failures: true
      - name: order_date
        data_type: date
```

```sql
-- tests/assert_no_orphan_orders.sql — singular test: returns failing rows
select o.order_id
from {{ ref('fct_orders') }} o
left join {{ ref('dim_customers') }} c on o.customer_id = c.customer_id
where c.customer_id is null
```

```sql
-- tests/generic/row_count_above.sql — custom generic test
{% test row_count_above(model, min_count) %}
select 1
from (select count(*) as row_count from {{ model }}) sub
where row_count < {{ min_count }}
{% endtest %}
```

```yaml
# models/staging/_sources.yml — freshness and loaded_at_field live under config (1.9 / 1.10)
sources:
  - name: stripe
    config:
      freshness:
        warn_after: {count: 12, period: hour}
        error_after: {count: 24, period: hour}
      loaded_at_field: _loaded_at
    tables:
      - name: payments
      - name: customers
```

## Configuration

| Feature | Where | Description |
|---------|-------|-------------|
| `arguments:` | test entry | Inputs to the test macro (`values`, `to`, `field`, `min_value`…); top-level inputs are deprecated in 1.10.5+ and an error on dbt v2 |
| `severity` | `config:` | `error` (default) or `warn`; `error_if` / `warn_if` refine thresholds |
| `where`, `limit` | `config:` | Filter the tested rows; cap stored failing rows |
| `store_failures` | `config:` or `--store-failures` | Saves failing rows in schema `{{ profile.schema }}_dbt_test__audit` |
| `store_failures_as` | `config:` | `ephemeral` (default, nothing stored), `table`, `view`; takes precedence over `store_failures` |
| `contract.enforced` + `constraints` | model `config:` / columns | Build-time schema contract; on Databricks only `not_null` and `check` are enforced, PK/FK are informational, `unique` is not supported; not for Python models, MVs or ephemeral models |
| `tests:` key | — | Still accepted as an alias of `data_tests:` (renamed in 1.8); cannot be mixed with `data_tests:` on one resource |

## Example Usage

```bash
dbt test                                   # all data + unit tests
dbt test --select fct_orders               # one model
dbt test --select test_type:data           # data tests only (test_type:unit for unit tests)
dbt build --select +fct_orders             # run + test upstream in DAG order
dbt build --empty                          # 1.8+: zero-row refs/sources, validates SQL and contracts cheaply
dbt source freshness
dbt test --store-failures
dbt parse --show-all-deprecations          # find old tests:/argument syntax before moving to dbt v2
uvx dbt-autofix deprecations               # rewrite YAML to the current syntax
```

Unit tests (1.8+, SQL models only) validate logic on static inputs — see
[dbt testing-framework](../../dbt/concepts/testing-framework.md). To generate tests from an ODCS
contract, `datacontract dbt sync <contract> --project-dir .` writes `data_tests` with `arguments` and
`constraints` into the model YAML ([datacontract-cli-workflow](datacontract-cli-workflow.md)).

## See Also

- [great-expectations](great-expectations.md)
- [schema-validation](schema-validation.md)
- [dbt generic tests](../../dbt/patterns/generic-tests.md)
- [dbt-databricks adapter](../../dbt/concepts/dbt-databricks-adapter.md)

## Sources

- https://docs.getdbt.com/docs/build/data-tests
- https://docs.getdbt.com/reference/resource-properties/data-tests
- https://docs.getdbt.com/reference/data-test-configs
- https://docs.getdbt.com/reference/resource-configs/store_failures
- https://docs.getdbt.com/reference/resource-configs/store_failures_as
- https://docs.getdbt.com/docs/mesh/govern/model-contracts
- https://docs.getdbt.com/reference/resource-properties/constraints
- https://docs.getdbt.com/docs/build/sources
- https://docs.getdbt.com/reference/commands/build
- https://docs.getdbt.com/reference/deprecations
