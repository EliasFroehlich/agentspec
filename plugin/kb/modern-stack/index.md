# Modern Stack Knowledge Base

> **Purpose**: Single-node and code-first data tools: DuckDB (+ DuckLake), Polars, SQLMesh (vs dbt), Malloy, Evidence, Observable Framework, local-first analytics
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Current State (2026-10-17)

| Tool | Version | Key point |
|------|---------|-----------|
| DuckDB | 1.5.6 stable; 1.4.5 LTS (EOL 2026-11-17) | 2.0.0 scheduled for 2026-10-21 (not released) |
| DuckLake | v1.0 (2026-04-13) | Needs DuckDB 1.5.2+; catalog in DuckDB/SQLite/PostgreSQL |
| Polars | 2.0.0 (2026-10-06) | `collect()` streams by default; row order opt-in (`maintain_order`) |
| SQLMesh | 0.236.3 (2026-10-06) | Linux Foundation project (2026-03-25); no `apply` command |
| dbt | v2.0 ("dbt" Fusion-based + "dbt OSS") | Fivetran + dbt Labs merged (2026-06-01) |
| Evidence | Evidence Core (MIT) + Evidence Studio | Legacy Svelte framework replaced by Markdoc-based Core |

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/duckdb.md](concepts/duckdb.md) | Release lines, features by version (encryption, MERGE, Iceberg writes, VARIANT, GEOMETRY), extensions, DuckLake |
| [concepts/polars.md](concepts/polars.md) | Lazy/eager, Polars 2.0 changes, streaming engine, Polars Cloud / Distributed |
| [concepts/sqlmesh.md](concepts/sqlmesh.md) | Virtual environments, change categories, governance, SQLMesh vs dbt v2 |
| [concepts/analytics-engineering.md](concepts/analytics-engineering.md) | Malloy + Publisher, Evidence Core/Studio, Observable Framework |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/duckdb-patterns.md](patterns/duckdb-patterns.md) | Files, `CREATE SECRET`, Iceberg read/time travel/REST writes, DuckLake, MERGE INTO, encryption, CI assertions |
| [patterns/polars-patterns.md](patterns/polars-patterns.md) | Lazy pipelines, expressions, windows, joins with `maintain_order`, 2.0 migration |
| [patterns/sqlmesh-workflow.md](patterns/sqlmesh-workflow.md) | `sqlmesh init`, MODEL + audits, plan dev → prod, scheduler, linter |
| [patterns/local-first-analytics.md](patterns/local-first-analytics.md) | DuckDB + Polars + Parquet/DuckLake stack, CI validation, cost model |

---

## Quick Reference

- [quick-reference.md](quick-reference.md): versions, feature-by-version tables, Polars 2.0 changes, SQLMesh vs dbt, pitfalls

## Related Domains

- [lakehouse/concepts/ducklake.md](../lakehouse/concepts/ducklake.md): DuckLake format in depth
- [lakehouse/index.md](../lakehouse/index.md): Iceberg, Delta, catalogs

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| data-platform-engineer | All files | Tool selection, local-first patterns |
| sql-optimizer | patterns/duckdb-patterns.md | DuckDB SQL optimization |

## Sources

- https://duckdb.org/release_calendar
- https://ducklake.select/2026/04/13/ducklake-10/
- https://pola.rs/posts/release-polars-2/
- https://github.com/SQLMesh/sqlmesh/releases
- https://www.linuxfoundation.org/press/linux-foundation-welcomes-sqlmesh-project
- https://docs.getdbt.com/blog/comparing-dbt-and-dbt-oss
- https://www.fivetran.com/press/fivetran-dbt-labs-complete-merger-to-create-the-data-infrastructure-for-trusted-ai-agents
- https://evidence.dev/blog/evidence-core
