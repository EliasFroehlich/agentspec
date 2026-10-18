# Mesh Architecture

> **Purpose**: dbt Mesh multi-project setup with cross-project refs, access modifiers, Semantic Layer
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

dbt Mesh enables domain teams to own independent dbt projects with **cross-project references**, **access modifiers** (public/protected/private), and **model versioning**. Combined with the **Semantic Layer** (MetricFlow), it implements Data Mesh principles: federated ownership with centralized metric definitions. Each project publishes its public models as a data product. Cross-project refs require a **dbt platform** (formerly dbt Cloud) **Enterprise or Enterprise+** plan, and the producer project needs `access: public` models and a successful production job. Catalog provides cross-project lineage.

## The Concept

```yaml
# models/marts/orders/_orders__models.yml
# Publishing a model as a public API for cross-project consumption
groups:
  - name: finance_team
    owner:
      email: finance-data@example.com

models:
  - name: fct_orders
    config:
      access: public          # visible to other projects (config: since 1.10)
      group: finance_team     # ownership boundary
      contract:
        enforced: true
    latest_version: 2         # consumers default to this version
    columns:
      - name: order_id
        data_type: string
        data_tests:
          - unique
          - not_null
      - name: order_total
        data_type: decimal(12,2)
    versions:
      - v: 2
        columns:
          - include: all
          - name: order_total_usd
            data_type: decimal(12,2)
            description: "Order total in USD (added in v2)"
      - v: 1
        defined_in: fct_orders_v1  # old version still available
```

## Quick Reference

| Access Level | Visibility | Use When |
|-------------|-----------|----------|
| `public` | Any project can ref() | Stable API, data product |
| `protected` | Same project + same group | Shared within team |
| `private` | Same project only | Internal implementation detail |

| Mesh Concept | dbt Feature |
|-------------|------------|
| Domain ownership | `config: {group: ...}` + top-level `groups:` |
| Data product API | `config: {access: public}` models |
| Contract | `contract: {enforced: true}` |
| Versioning | `versions:` with `latest_version` |
| Centralized metrics | Semantic Layer (MetricFlow) — cross-project refs only in the legacy `semantic_models:` spec |

### Considerations

- Governance configs (`access`, `group`, `contract`, `versions`) apply to models only — not to snapshots, seeds or sources
- `constraints` are not supported on views; on Databricks only `not_null` and `check` are enforced
- The latest Semantic Layer YAML spec (dbt 1.12+/v2) does not yet support cross-project references

## Migration Heuristics (Production Mesh Adoption)

| Step | Question | Action |
|------|----------|--------|
| 1 | Is there a single downstream team? | Start there as a consumer |
| 2 | Are people working on separate transformation levels? | Split by staging vs marts |
| 3 | Different data sources handled separately? | Split by domain boundary |
| 4 | Existing logical groupings in your project? | Formalize as Mesh interfaces |

## Common Mistakes

### Wrong

```yaml
# Exposing internal models as public without contracts
models:
  - name: int_orders_pivoted
    config:
      access: public  # Internal model exposed — no contract, will break consumers
```

### Correct

```yaml
# Only publish stable mart models with enforced contracts
models:
  - name: fct_orders
    config:
      access: public
      contract:
        enforced: true  # Schema changes must be backward-compatible
    columns:
      - name: order_id
        data_type: string  # Enforced type contract
        data_tests: [unique, not_null]
```

## Related

- [fusion-engine](../concepts/fusion-engine.md)
- [semantic-layer](../patterns/semantic-layer.md)

## Sources

- https://docs.getdbt.com/docs/mesh/govern/project-dependencies
- https://docs.getdbt.com/docs/mesh/govern/model-access
- https://docs.getdbt.com/docs/mesh/govern/model-versions
- https://docs.getdbt.com/docs/mesh/govern/model-contracts
- https://docs.getdbt.com/docs/build/latest-metrics-spec
- https://docs.getdbt.com/reference/resource-properties/constraints
