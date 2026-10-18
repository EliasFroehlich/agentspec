# dbt Quick Reference

> **Purpose**: Fast lookup tables for dbt v1.12 / v2.0 and dbt-databricks 1.12.7. For code examples, see linked files.
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Model Types

| Prefix | Layer | Materialization | Purpose |
|--------|-------|----------------|---------|
| `stg_` | Staging | view | 1:1 with source, rename + cast only |
| `int_` | Intermediate | ephemeral/view | Business logic transformations |
| `fct_` | Marts (facts) | table/incremental | Measurable events at a grain |
| `dim_` | Marts (dimensions) | table | Descriptive attributes |
| `snap_` | Snapshots | snapshot (YAML, 1.9+) | SCD Type 2 history tracking |

## Incremental Strategy Decision Matrix

| Strategy | Best for | `unique_key` | Databricks | Since |
|----------|----------|--------------|------------|-------|
| `append` | Immutable events | Not used | Yes | All |
| `merge` | Updatable rows | Required | **Default** (Delta/Hudi) | All |
| `delete+insert` | Large facts with updates | Required | adapter 1.11+, DBR 12.2 LTS+ | All (core) |
| `insert_overwrite` | Partition replacement | Not used | Needs `partition_by`/`liquid_clustered_by` or replaces the table | All |
| `replace_where` | Replace a predicate-defined slice | Not used | Databricks only (`incremental_predicates`) | — |
| `microbatch` | Large time series, backfills | Not used (`event_time`) | Via `replace_where` | **v1.9+** |

## CLI Cheat Sheet

| Command | Purpose |
|---------|---------|
| `dbt build` | run + test + snapshot + seed in DAG order |
| `dbt build --empty` | Zero-row refs/sources: validate SQL and contracts cheaply (1.8+) |
| `dbt build --sample="3 days"` | Time-based sample of refs/sources (1.10+, needs `event_time`) |
| `dbt test --select test_type:unit` / `test_type:data` | Only unit tests / only data tests |
| `dbt run --select fct_events --event-time-start "2026-01-01" --event-time-end "2026-02-01"` | Microbatch backfill window |
| `dbt source freshness` | Source staleness (`config: freshness:`) |
| `dbt parse --no-partial-parse --show-all-deprecations` | List every deprecation before upgrading (v2: drop `--no-partial-parse`) |
| `uvx dbt-autofix deprecations` | Rewrite YAML/SQL to current syntax (`packages`, `jobs` subcommands too) |
| `dbt system update [--version canary]` | Update a dbt v2 binary install |
| `mf query …` / `dbt sl query …` | Query metrics self-hosted / on dbt platform |

## dbt v1 vs dbt v2

| | dbt v1 (Python) | dbt v2 (Rust, ex-Fusion) |
|---|---|---|
| Current | 1.12 (active to 2027-07-15); 1.13 beta = last 1.x minor | 2.0.8 stable; 2.0.9 canary |
| License | Apache 2.0 | `dbt` distribution: dbt Product Licensing Agreement; `dbt OSS`: Apache 2.0 |
| YAML strictness | Deprecation warnings | Some are errors (test inputs must be under `arguments:`) |
| Free | Everything | Syntax errors, `dbt lint`, docs v2, LSP lite |
| With dbt platform login (free tier works) | — | Full LSP, SQL comprehension, precise column-level lineage |
| Databricks adapter | dbt-databricks 1.12.7 | GA |

## Test YAML (1.10.5+ / v2)

| Key | Holds |
|-----|-------|
| `data_tests:` | List of tests (`tests:` is an alias) |
| `arguments:` | Test inputs: `values`, `to`, `field`, `min_value`… |
| `config:` | `severity`, `where`, `store_failures`, `store_failures_as`, `tags`, `meta` |

## Unit Testing (v1.8+)

| Aspect | Details |
|--------|---------|
| Definition | `unit_tests:` YAML in `models/` |
| Scope | SQL models only (not Python models) |
| Inputs | Static `rows` for every `ref()` / `source()` the model reads |
| Exclusions | No materialized views, recursive SQL or introspective queries |
| dbt v2 local run | Experimental `compute: local` (DuckDB) — Snowflake/BigQuery only |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `tests:` with inputs next to `severity` | `data_tests:` → `arguments:` + `config:` |
| `access:` / `group:` / `meta:` as top-level model keys | Put them under `config:` (1.10+) |
| `{% snapshot %}` SQL blocks for new snapshots | YAML snapshots with `relation:` and `hard_deletes` |
| `semantic_models:` + `measures:` in new projects | Latest spec: `semantic_model: {enabled: true}` + simple metrics |
| `cluster_by` on Databricks | `liquid_clustered_by` or `auto_liquid_cluster` |
| SQL `--` comments inside `{{ config(...) }}` | Comments outside the Jinja call |
| `calogica/dbt_expectations` | `metaplane/dbt_expectations` 0.10.10 (maintained, v2-compatible) |

## Related Documentation

| Topic | Path |
|-------|------|
| Getting Started | `concepts/model-types.md` |
| dbt v2 | `concepts/fusion-engine.md` |
| Databricks | `concepts/dbt-databricks-adapter.md` |
| Full Index | `index.md` |

## Sources

- https://docs.getdbt.com/docs/dbt-versions
- https://docs.getdbt.com/reference/resource-configs/databricks-configs
- https://docs.getdbt.com/reference/deprecations
