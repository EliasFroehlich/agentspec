# Data Contracts

> **Purpose**: What a data contract is, the real structure of ODCS v3.2.0, how it relates to ODPS, dbt model contracts and Soda contracts
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

A data contract is a versioned, machine-readable agreement between a data producer and its consumers:
schema, quality rules, service levels, ownership and support channels. The vendor-neutral format is the
**Open Data Contract Standard (ODCS)**, maintained by the Bitol project under the LF AI & Data Foundation.
Current release: **ODCS v3.2.0 "Peter Flook" (2026-09-08)**; v3.1.0 was released 2025-12-08. Its sibling
**ODPS (Open Data Product Standard) v1.1.0** (same day) describes the data product that exposes one or more
contracts. Tool-specific "contracts" (dbt model contracts, Soda contracts) are different formats — they
enforce part of an agreement, but they are not ODCS documents.

## The Concept

```yaml
apiVersion: v3.2.0          # required — standard version
kind: DataContract          # required
id: sales-orders            # required — stable identifier
version: 1.2.0              # required — contract version (semver)
name: orders
status: active              # proposed | draft | active | deprecated | retired
domain: sales
team:
  name: sales-data
  members:
    - username: owner@example.com
      role: owner
servers:
  - server: prod
    type: databricks
    catalog: main
    schema: sales
schema:
  - name: orders
    physicalType: table
    properties:
      - name: order_id
        logicalType: string
        physicalType: STRING
        primaryKey: true
        required: true
        quality:
          - metric: duplicateValues
            mustBe: 0
      - name: amount
        logicalType: number
        physicalType: DECIMAL(12,2)
        logicalTypeOptions:
          minimum: 0
slaProperties:
  - property: latency
    value: 1
    unit: h
```

Validated against the official `odcs-json-schema-v3.2.0.json` and with `datacontract lint` (1.2.7).

## Quick Reference

| ODCS section (top-level key) | Holds |
|------------------------------|-------|
| Fundamentals: `apiVersion`, `kind`, `id`, `version` (required); `name`, `status`, `domain`, `tenant`, `tags`, `description{purpose,limitations,usage}` | Identity and lifecycle |
| `servers[]` | Where the data lives (`type: databricks` + `catalog`, `schema`, optional `host`) |
| `schema[]` → `properties[]` | Objects (tables) and their properties: `logicalType`, `physicalType`, `required`, `unique`, `primaryKey`, `classification`, `logicalTypeOptions`, `enum[]` (3.2) |
| `quality[]` (on an object or a property) | `type: library` (`metric: nullValues/missingValues/invalidValues/duplicateValues/rowCount`), `sql`, `text`, `custom`; operators `mustBe`, `mustBeLessThan`, `mustBeBetween`…; `unit: rows/percent` |
| `relationships[]` (3.1+) | Foreign keys: `to:` at property level, `from:` + `to:` at object level |
| `slaProperties[]` | `property` + `value` (+ `unit`, `element`): `latency` (preferred to "freshness"), `frequency`, `availability`, `retention`, `timeToDetect`… |
| `support[]`, `team{}`, `roles[]`, `price` | Channels (`channel`, `tool`, `url`), owners, access roles, pricing |
| `customProperties[]`, `authoritativeDefinitions[]`, `context` (3.2) | Extensions, links, guidance for AI agents |

| Version | Notable additions |
|---------|-------------------|
| v3.1.0 (2025-12-08) | `relationships` (FK), stable `id`s and references, `team` as an object; `dataProduct` deprecated |
| v3.2.0 (2026-09-08) | `enum[]`, `logicalType: map`/`vector`, `semanticType: measure/dimension`, `synonyms`, `deprecated` flag, `context` block, `${VAR}` variables, `iceberg`/`hana` servers |

| Format | Scope | Enforced by |
|--------|-------|-------------|
| ODCS v3.2.0 | Full agreement (schema, quality, SLA, owners) | `datacontract lint` / `test`, any ODCS-aware tool |
| ODPS v1.1.0 | The data product (ports, owners) that exposes contracts | Catalogs / marketplaces |
| dbt model contract (`contract: {enforced: true}`) | Column names and types (+ constraints) at build time | `dbt build` |
| Soda contract (Soda Core v4) | Executable quality checks for one dataset | `soda contract verify` |

## Common Mistakes

### Wrong

```yaml
# Not ODCS — fails the official JSON Schema ('metadata', 'references', 'sla' are not allowed;
# 'schema' must be an array; 'id' and 'version' are required)
apiVersion: v3.1.0
kind: DataContract
metadata: {name: orders, owner: data-engineering@company.com}
schema: {type: object, properties: {order_id: {type: string, format: uuid}}}
references: [{column: customer_id, referencedDataset: customers}]
sla: {availability: 99.9%}
```

### Correct

```yaml
# Owners go in team.members, channels in support[], SLAs in slaProperties[], FKs in relationships[]
team: {name: sales-data, members: [{username: owner@example.com, role: owner}]}
support: [{channel: "#sales-data", tool: slack}]
slaProperties: [{property: availability, value: 99.9, unit: percent}]
```

## Related

- [data-contract-authoring](../patterns/data-contract-authoring.md) — full validated contract
- [datacontract-cli-workflow](../patterns/datacontract-cli-workflow.md) — lint, test, breaking, export
- [soda-core](soda-core.md) — Soda's own contract language
- [dbt-testing](../patterns/dbt-testing.md) — dbt model contracts

## Sources

- https://github.com/bitol-io/open-data-contract-standard/releases
- https://github.com/bitol-io/open-data-contract-standard/blob/main/CHANGELOG.md
- https://bitol-io.github.io/open-data-contract-standard/latest/fundamentals/
- https://bitol-io.github.io/open-data-contract-standard/latest/schema/
- https://bitol-io.github.io/open-data-contract-standard/latest/data-quality/
- https://bitol-io.github.io/open-data-contract-standard/latest/service-level-agreement/
- https://raw.githubusercontent.com/bitol-io/open-data-contract-standard/v3.2.0/schema/odcs-json-schema-v3.2.0.json
- https://github.com/bitol-io/open-data-product-standard/releases
- https://docs.getdbt.com/docs/mesh/govern/model-contracts
- https://docs.soda.io/reference/contract-language-reference
