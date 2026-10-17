# Data Contract Authoring

> **Purpose**: A complete ODCS v3.2.0 contract for a Databricks table — schema, quality rules, relationships, SLAs, support, team — validated against the official JSON Schema
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Formalizing the interface between a producing team and its consumers
- Declaring quality rules and SLAs once, then executing them with `datacontract test` or exporting them to dbt / Soda / GX
- Gating schema changes in CI with `datacontract breaking`
- Giving catalogs and AI agents an authoritative description of a dataset (`description`, `context`)

## Implementation

```yaml
# contracts/sales/orders.odcs.yaml — ODCS v3.2.0
apiVersion: v3.2.0
kind: DataContract
id: sales-orders
name: orders
version: 2.1.0
status: active
domain: sales
tags: [sales, critical]
description:
  purpose: One row per order (latest state) for revenue reporting.
  limitations: Orders older than 3 years are archived and excluded.
  usage: Join to customers on customer_id; aggregate by order_date.

servers:
  - server: prod
    type: databricks
    environment: prod
    host: ${DATABRICKS_HOST}        # ODCS 3.2 variable, resolved by tooling
    catalog: main
    schema: sales
  - server: dev
    type: databricks
    environment: dev
    host: ${DATABRICKS_HOST}
    catalog: dev
    schema: sales

schema:
  - name: orders
    physicalName: orders
    physicalType: table
    description: Order fact table, one row per order_id.
    quality:                                    # object-level rules
      - id: orders_not_empty
        metric: rowCount
        mustBeGreaterThan: 0
      - id: orders_amount_not_negative
        type: sql
        query: SELECT COUNT(*) FROM {object} WHERE amount < 0
        mustBe: 0
    properties:
      - name: order_id
        logicalType: string
        physicalType: STRING
        primaryKey: true
        primaryKeyPosition: 1
        required: true
        unique: true
      - name: customer_id
        logicalType: string
        physicalType: STRING
        required: true
        classification: confidential
        relationships:
          - type: foreignKey
            to: customers.customer_id           # shorthand <object>.<property>
        quality:
          - id: customer_id_null_pct
            metric: nullValues
            mustBeLessThan: 1
            unit: percent
      - name: status
        logicalType: string
        physicalType: STRING
        required: true
        enum:                                   # ODCS 3.2
          - value: pending
          - value: completed
          - value: cancelled
          - value: refunded
      - name: amount
        logicalType: number
        physicalType: DECIMAL(12,2)
        required: true
        logicalTypeOptions:
          minimum: 0
      - name: updated_at
        logicalType: timestamp
        physicalType: TIMESTAMP
        required: true
  - name: customers
    physicalType: table
    properties:
      - name: customer_id
        logicalType: string
        physicalType: STRING
        primaryKey: true
        required: true

slaProperties:
  - id: latency
    property: latency
    value: 1
    unit: h
    element: orders.updated_at
  - id: update_frequency
    property: frequency
    value: 1
    unit: d
    element: orders.updated_at
  - id: retention
    property: retention
    value: 3
    unit: y

support:
  - channel: "#sales-data"
    tool: slack
    scope: interactive
  - channel: sales-data-team
    tool: email
    url: mailto:sales-data@example.com

team:
  name: sales-data
  members:
    - username: owner@example.com
      role: owner
    - username: steward@example.com
      role: data steward
```

This file passes `check-jsonschema` against `odcs-json-schema-v3.2.0.json` and `datacontract lint` (1.2.7).

## Configuration

| Rule | Why |
|------|-----|
| `apiVersion`, `kind`, `id`, `version` are the only required top-level keys | Everything else is optional; unknown top-level keys are rejected (`additionalProperties: false`) |
| `schema` is a **list** of objects; columns are `properties[]` with `name` | No `columns:`, no `type: object` map, no `format:` (use `logicalTypeOptions.format`) |
| Put physical location in `servers[]` (`catalog`, `schema` for Databricks) | Same contract works for `dev` and `prod`; pick one with `--server` |
| `quality[]` lives on the object (row count, multi-column duplicates, SQL) or on a property | `type` defaults to `library` when `metric` is set; SQL rules use `{object}` / `{property}` placeholders |
| Library metrics: `nullValues`, `missingValues`, `invalidValues` (`arguments.validValues` / `pattern`), `duplicateValues`, `rowCount` | One operator per rule: `mustBe`, `mustNotBe`, `mustBeGreaterThan`, `mustBeGreaterOrEqualTo`, `mustBeLessThan`, `mustBeLessOrEqualTo`, `mustBeBetween`, `mustNotBeBetween` |
| Freshness is an SLA: `property: latency` (plus `frequency`) | ODCS prefers `latency` to "freshness" |
| Owners in `team.members[]` (`username`, `role`); channels in `support[]` | There is no `owner:` or `slack_channel:` key |
| `relationships[].to` at property level (no `from`) | Object-level relationships need both `from` and `to` |
| `${VAR}` / `${VAR:-default}` only in string values (3.2) | Keeps hosts and secrets out of Git; integer fields cannot hold variables (server `port` excepted) |
| Lifecycle: `status` (`proposed`, `draft`, `active`, `deprecated`, `retired`) + `deprecated: true` on objects/properties (3.2) | There are no `evolution:` or `consumers:` sections |

## Example Usage

```bash
datacontract lint contracts/sales/orders.odcs.yaml                    # schema validation
datacontract test contracts/sales/orders.odcs.yaml --server dev       # run quality + schema checks
datacontract breaking v2.1.0/orders.odcs.yaml orders.odcs.yaml        # compatibility impact (two files)
datacontract export dbt-models orders.odcs.yaml --output models/sales/_orders.yml
```

Full CLI workflow (init/import, CI gating, Databricks connection): [datacontract-cli-workflow](datacontract-cli-workflow.md).

## See Also

- [data-contracts](../concepts/data-contracts.md)
- [datacontract-cli-workflow](datacontract-cli-workflow.md)
- [schema-validation](schema-validation.md)

## Sources

- https://bitol-io.github.io/open-data-contract-standard/latest/fundamentals/
- https://bitol-io.github.io/open-data-contract-standard/latest/schema/
- https://bitol-io.github.io/open-data-contract-standard/latest/data-quality/
- https://bitol-io.github.io/open-data-contract-standard/latest/service-level-agreement/
- https://bitol-io.github.io/open-data-contract-standard/latest/infrastructure-servers/
- https://bitol-io.github.io/open-data-contract-standard/latest/references/
- https://bitol-io.github.io/open-data-contract-standard/latest/variables/
- https://raw.githubusercontent.com/bitol-io/open-data-contract-standard/v3.2.0/schema/odcs-json-schema-v3.2.0.json
- https://github.com/datacontract/datacontract-cli
