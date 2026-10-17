# Modern Stack Quick Reference

> **Purpose**: Fast lookup tables for DuckDB, DuckLake, Polars, SQLMesh and dbt v2 — versions as of 2026-10-17; code in linked files
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Current Versions

| Tool | Current | Notes |
|------|---------|-------|
| DuckDB | 1.5.6 (2026-09-28); LTS 1.4.5 (2026-06-17) | 1.4 LTS EOL 2026-11-17; **2.0.0 scheduled for 2026-10-21** (not released) |
| DuckLake | spec/extension v1.0 (2026-04-13) | Needs DuckDB 1.5.2+ |
| Polars (Python) | 2.0.0 (2026-10-06); last 1.x: 1.44.2 (2026-09-09) | 2.0 changes `collect()` defaults (see below) |
| SQLMesh | 0.236.3 (2026-10-06) | Linux Foundation project since 2026-03-25 |
| dbt | v2.0 (initial release 2026-09-14); 1.13 in beta | 1.13 will be the last 1.x minor |
| Evidence | Evidence Core (MIT, 2026-08-26) + Evidence Studio (hosted) | Legacy Svelte framework replaced by Markdoc-based Core |

## "Do I Need Spark?" (rules of thumb, benchmark your own data)

| Question | If Yes | If No |
|----------|--------|-------|
| Data too large for one machine (RAM + local disk)? | Spark / warehouse | DuckDB or Polars |
| Need distributed compute or multi-writer concurrency? | Spark, warehouse, or DuckLake on PostgreSQL | DuckDB (single node) |
| Real-time streaming? | Spark Structured Streaming or Flink | DuckDB/Polars (batch) |
| CI/CD data tests? | DuckDB (instant startup) | — |
| Analytics on a laptop? | DuckDB or Polars | — |

## DuckDB Features by Version

| Feature | Since |
|---------|-------|
| `CREATE SECRET` (replaces settings-based S3 auth) | 0.10.0 |
| Friendlier SQL, `map['k']` returns the value | 1.2.0 |
| `lambda x: x + 1` (arrow `->` deprecated) | 1.3.0 |
| AES-256 database encryption (`ENCRYPTION_KEY`) | 1.4.0 |
| `MERGE INTO` | 1.4.0 |
| Iceberg writes via `ATTACH ... (TYPE iceberg)` REST catalog | 1.4.0 (`UPDATE`/`DELETE` 1.4.2; `MERGE INTO`/`ALTER` 1.5.3) |
| `VARIANT` type, `GEOMETRY` in core, new CLI | 1.5.0 |
| DuckLake 1.0 | 1.5.2 |
| Arrow lambdas off by default | 2.0 (planned) |

## Polars 2.0 Migration (1.x → 2.0)

| Area | 1.x | 2.0 |
|------|-----|-----|
| `LazyFrame.collect()` | in-memory engine unless configured | **streaming engine** by default |
| Row order after `join` / `group_by` / `unpivot` | often preserved by accident | **not guaranteed**; use `maintain_order` |
| `collect(streaming=True)` | deprecated (1.25) | removed (`ArgumentRemovedError`); use `engine="streaming"` |
| Pin old engine | — | `collect(engine="in-memory")` or `pl.Config.set_engine_affinity("in-memory")` |
| Arrow/Parquet/Iceberg maps | `List(Struct)` | native `Map` dtype |
| `LazyFrame.profile()` | available | removed |
| `pl.read_csv` | own reader | dispatches to `scan_csv(...).collect()` |
| Spill to disk (out-of-core) | not on by default | enabled by default (sort, windows, many expressions) |

## SQLMesh vs dbt (2026-10)

| Feature | SQLMesh | dbt (v2, Fusion-based) | dbt OSS (v2) |
|---------|---------|------------------------|--------------|
| License | Apache 2.0 | Not 100% Apache 2.0 (license FAQ) | 100% Apache 2.0 |
| Steward | Linux Foundation (TSC) | dbt Labs (merged with Fivetran) | dbt Labs |
| Dev environments | Virtual (views over shared physical tables) | Schema per developer/target | Same |
| Change detection | Semantic diff (SQLGlot); breaking / non-breaking / forward-only | State comparison (`state:modified`) | Same |
| Backfill | Automatic, only affected intervals/models | `--full-refresh` / incremental strategies | Same |
| Scheduler | `sqlmesh run` (cron per model) | External orchestrator or dbt platform | External |
| Column-level lineage | SQLMesh UI | With `static_analysis: strict` (VS Code ext., Docs v2) | No |
| SQL comprehension / LSP | Parse-time (SQLGlot) | Yes | No |
| Run dbt projects | Yes (`sqlmesh init -t dbt`) | Native | Native |

## Ownership Changes (2025–2026)

| Date | Event |
|------|-------|
| 2025-09-03 | Fivetran acquires Tobiko Data (SQLMesh, SQLGlot) |
| 2025-10-13 | Fivetran + dbt Labs merger announced |
| 2026-03-25 | Fivetran contributes SQLMesh to the Linux Foundation |
| 2026-06-01 | Fivetran + dbt Labs merger completed |
| 2026-07-13 | Prefect announces it is acquiring Dagster Labs |

## Common Pitfalls (Wrong → Correct)

| Wrong / deprecated | Correct |
|-------|-----|
| `read_csv_auto` / `read_json_auto` | `read_csv` / `read_json` (auto-detect is built in) |
| `SET s3_access_key_id = ...` | `CREATE SECRET (TYPE s3, PROVIDER credential_chain)` |
| `iceberg_scan(..., version => id)` for time travel | `snapshot_from_id = ...` / `snapshot_from_timestamp = ...` |
| `collect(streaming=True)` | `collect()` on 2.0, `collect(engine="streaming")` on 1.x |
| Rely on join/group_by output order in Polars 2.0 | `maintain_order=...` or an explicit `sort` |
| `sqlmesh apply dev` (no such command) | `sqlmesh plan dev` (prompts) or `sqlmesh plan dev --auto-apply` |
| Document DuckDB 2.0 features as available | Treat 2.0 as scheduled until it ships |

## Related Documentation

`concepts/duckdb.md` · `concepts/polars.md` · `concepts/sqlmesh.md` · `index.md` (each file lists its sources)

## Sources

- https://duckdb.org/release_calendar · https://ducklake.select/2026/04/13/ducklake-10/
- https://docs.pola.rs/releases/upgrade/2/ · https://github.com/SQLMesh/sqlmesh/releases
- https://docs.getdbt.com/docs/dbt-versions · https://docs.getdbt.com/blog/comparing-dbt-and-dbt-oss
- https://www.linuxfoundation.org/press/linux-foundation-welcomes-sqlmesh-project · https://www.prefect.io/prefect-acquires-dagster
