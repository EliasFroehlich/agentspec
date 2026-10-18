# dbt Knowledge Base

> **Purpose**: dbt development patterns — dbt v1 and v2 (ex-Fusion), the dbt-databricks adapter, Mesh, Semantic Layer (latest spec) vs Unity Catalog metric views, models, macros, tests
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17 | dbt v1.12, dbt v2.0 GA (2026-09-14), dbt-databricks 1.12.7, `data_tests` + `arguments`, YAML snapshots

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/model-types.md](concepts/model-types.md) | Staging, intermediate, mart, and snapshot models |
| [concepts/incremental-strategies.md](concepts/incremental-strategies.md) | append, merge, delete+insert, insert_overwrite, microbatch, replace_where |
| [concepts/testing-framework.md](concepts/testing-framework.md) | Data tests (`data_tests`/`arguments`/`config`), singular, custom, unit tests |
| [concepts/fusion-engine.md](concepts/fusion-engine.md) | dbt v2 (ex-Fusion): distributions, licenses, v1 → v2 migration |
| [concepts/dbt-databricks-adapter.md](concepts/dbt-databricks-adapter.md) | Liquid clustering, `replace_where`, tags, Python submission, constraints, `metric_view` |
| [concepts/mesh-architecture.md](concepts/mesh-architecture.md) | Multi-project refs, access modifiers, versions, contracts |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/incremental-model.md](patterns/incremental-model.md) | Production incremental model with merge strategy |
| [patterns/snapshot-scd2.md](patterns/snapshot-scd2.md) | YAML snapshots, `hard_deletes`, `dbt_valid_to_current` |
| [patterns/generic-tests.md](patterns/generic-tests.md) | Custom generic test macros and parameterization |
| [patterns/macro-patterns.md](patterns/macro-patterns.md) | Jinja control flow, cross-db macros, `grants:` config |
| [patterns/semantic-layer.md](patterns/semantic-layer.md) | MetricFlow latest spec, time spine, MetricFlow vs UC metric views |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Model Types** | stg_, int_, fct_, dim_ naming with ref()/source() chains |
| **Incremental** | Avoid full refresh — merge (Databricks default), append, delete+insert, insert_overwrite, replace_where, **microbatch** (v1.9+) |
| **dbt v2** | Rust engine (ex-Fusion), GA 2026-09-14; `dbt` and `dbt OSS` distributions; stricter YAML |
| **dbt platform** | New name of dbt Cloud (Semantic Layer APIs, Mesh cross-project refs on Enterprise plans) |
| **Mesh** | Multi-project architecture with cross-project refs, contracts, and access modifiers |
| **Semantic Layer** | MetricFlow metrics in model YAML (latest spec 1.12+/v2); UC metric views are the Databricks-native alternative |
| **Unit Testing** | Validate SQL logic on static inputs before materialization (v1.8+) |
| **Sample / empty** | `--sample` (1.10+) and `--empty` (1.8+) for cheap dev builds |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/model-types.md, concepts/testing-framework.md |
| **Intermediate** | patterns/incremental-model.md, concepts/dbt-databricks-adapter.md |
| **Advanced** | concepts/mesh-architecture.md, patterns/semantic-layer.md, concepts/fusion-engine.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| dbt-specialist | All files | Model creation, macro dev, test strategy, v2 migration, Databricks configs |
| data-quality-analyst | patterns/generic-tests.md, concepts/testing-framework.md | Test generation for dbt models |
| build-agent | patterns/incremental-model.md | Verification via dbt build |

## Sources

- https://docs.getdbt.com/docs/dbt-versions
- https://docs.getdbt.com/blog/dbt-core-v2-is-here
- https://github.com/databricks/dbt-databricks/releases
- https://docs.getdbt.com/reference/resource-properties/data-tests
