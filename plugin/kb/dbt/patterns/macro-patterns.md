# Macro Patterns

> **Purpose**: Jinja control flow, cross-database macros, grants (prefer the `grants:` config), and whitespace control
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Generating repetitive SQL across many models (DRY principle)
- Supporting multiple warehouse platforms with a single codebase
- Encapsulating business logic that changes independently of model SQL
- Post-run operations like granting permissions or logging

## Implementation

```sql
-- macros/generate_schema_name.sql
-- Override default schema generation for environment isolation

{% macro generate_schema_name(custom_schema_name, node) %}
    {%- set default_schema = target.schema -%}

    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- elif target.name == 'prod' -%}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{% endmacro %}
```

```sql
-- macros/cents_to_dollars.sql
-- Reusable currency conversion with null safety

{% macro cents_to_dollars(column_name, precision=2) %}
    round(cast({{ column_name }} as decimal(18, {{ precision }})) / 100, {{ precision }})
{% endmacro %}
```

```yaml
# Grants: prefer the native config over a hand-written GRANT post-hook
models:
  - name: fct_orders
    config:
      grants:
        select: ['analysts', 'bi_service']   # the most specific level replaces inherited grants
# dbt_project.yml: `+grants: {select: [...]}`; `'+select'` in a model adds to inherited grants;
# an empty list revokes. `grants:` targets the model's relation; dbt's Databricks notes say READ_METADATA /
# USAGE (legacy hive_metastore privileges) need post-hooks, as do schema-level Unity Catalog privileges.
```

```sql
-- macros/grant_use_schema.sql — schema-level privilege outside `grants:` (Unity Catalog syntax)
{% macro grant_use_schema(principal) %}
    {% if execute %}
        {% do run_query("GRANT USE SCHEMA ON SCHEMA " ~ this.database ~ "." ~ this.schema
                        ~ " TO `" ~ principal ~ "`") %}
    {% endif %}
{% endmacro %}
```

```sql
-- macros/union_sources.sql
-- Dynamically union multiple source tables with the same schema

{% macro union_sources(schema_name, table_prefix, source_list) %}

    {%- for source_name in source_list %}
        select
            '{{ source_name }}' as source_system,
            *
        from {{ source(schema_name, table_prefix ~ '_' ~ source_name) }}
        {% if not loop.last %}union all{% endif %}
    {%- endfor %}

{% endmacro %}
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `macro-paths` | `["macros"]` | Directories to scan for macros |
| `dispatch` | `[]` | Package search order for adapter.dispatch() |
| `post-hook` | `[]` | SQL/macros to run after model builds |
| `pre-hook` | `[]` | SQL/macros to run before model builds |
| `grants` | `{}` | Privilege → list of principals; dbt applies and revokes to match |

## Example Usage

```yaml
# dbt_project.yml — folder-level grants plus a post-hook for a privilege outside `grants:`
models:
  analytics:
    marts:
      +grants:
        select: ['analysts']
      +post-hook:
        - "{{ grant_use_schema('analysts') }}"
```

```sql
-- Using cents_to_dollars in a model
select
    order_id,
    {{ cents_to_dollars('amount_cents') }} as amount_dollars
from {{ ref('stg_payments') }}
```

## See Also

- [generic-tests](../patterns/generic-tests.md)
- [model-types](../concepts/model-types.md)

## Sources

- https://docs.getdbt.com/reference/resource-configs/grants
- https://docs.getdbt.com/docs/build/jinja-macros
- https://docs.getdbt.com/reference/dbt-jinja-functions/dispatch
- https://docs.getdbt.com/docs/build/custom-schemas
- https://docs.databricks.com/aws/en/sql/language-manual/security-grant
