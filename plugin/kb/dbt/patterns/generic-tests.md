# Generic Tests

> **Purpose**: Custom reusable test macros with parameters, severity and tagging — applied with `data_tests:` + `arguments:` + `config:`
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- A test pattern repeats across many models (e.g., positive values, date ranges)
- Built-in tests (unique, not_null, accepted_values) don't cover your validation
- You want parameterized, reusable tests shared across your dbt project

## Implementation

```sql
-- macros/tests/test_positive_values.sql
-- Custom generic test: assert all values in a column are > 0

{% test positive_values(model, column_name, allow_zero=false) %}

    select {{ column_name }}
    from {{ model }}
    where {{ column_name }} {% if allow_zero %}< 0{% else %}<= 0{% endif %}
      and {{ column_name }} is not null

{% endtest %}
```

```sql
-- macros/tests/test_row_count_ratio.sql
-- Assert output row count is within expected ratio of input

{% test row_count_ratio(model, input_model, min_ratio=0.9, max_ratio=1.1) %}

    with output_count as (
        select count(*) as cnt from {{ model }}
    ),
    input_count as (
        select count(*) as cnt from {{ ref(input_model) }}
    )
    select
        o.cnt as output_rows,
        i.cnt as input_rows,
        o.cnt::float / nullif(i.cnt, 0) as ratio
    from output_count o
    cross join input_count i
    where o.cnt::float / nullif(i.cnt, 0) not between {{ min_ratio }} and {{ max_ratio }}

{% endtest %}
```

```yaml
# Apply custom tests in a properties file (dbt 1.10.5+ / v2 shape)
models:
  - name: fct_orders
    data_tests:
      - row_count_ratio:
          arguments:                  # inputs to the test macro
            input_model: stg_stripe__orders
            min_ratio: 0.95
            max_ratio: 1.05
          config:                     # framework options
            severity: warn
    columns:
      - name: order_total
        data_tests:
          - positive_values:
              arguments:
                allow_zero: true
              config:
                severity: error
                tags: ['finance', 'critical']
                store_failures: true
                schema: test_results
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `severity` | `error` | `error` fails the pipeline, `warn` logs only |
| `store_failures` | `false` | Save failing rows to a table (default schema `{{ profile.schema }}_dbt_test__audit`) |
| `schema` | `dbt_test__audit` suffix | Schema for stored failure tables |
| `tags` | `[]` | Tags for selective test execution (`dbt test --select tag:finance`) |
| `enabled` | `true` | Disable test without removing it |
| `store_failures_as` | `ephemeral` | `table` or `view` to persist failures; overrides `store_failures` |

All of the above go under `config:`; macro parameters (`allow_zero`, `min_ratio`) go under `arguments:`. Top-level
inputs are deprecated since 1.10.5 (`MissingArgumentsPropertyInGenericTestDeprecation`) and rejected by dbt v2.

## Example Usage

```bash
# Run only critical tests
dbt test --select tag:critical

# Run tests for specific model
dbt test --select fct_orders

# Run tests and store failures
dbt test --store-failures
```

## See Also

- [testing-framework](../concepts/testing-framework.md)
- [macro-patterns](../patterns/macro-patterns.md)

## Sources

- https://docs.getdbt.com/docs/build/data-tests
- https://docs.getdbt.com/reference/data-test-configs
- https://docs.getdbt.com/reference/resource-configs/store_failures
- https://docs.getdbt.com/reference/resource-configs/store_failures_as
- https://docs.getdbt.com/reference/deprecations
