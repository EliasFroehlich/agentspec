# SQL Patterns Knowledge Base

> **Purpose**: Cross-dialect SQL — window functions, CTEs, deduplication, gaps and islands, pivots — for Snowflake, BigQuery, DuckDB, PostgreSQL, Spark SQL (OSS 4.x) and Databricks SQL
> **Version Coverage**: Spark 4.0–4.2 / Databricks Runtime 17.3 LTS–19; DuckDB 1.4 LTS / 1.5 stable (2.0 scheduled 2026-10-21); Snowflake and BigQuery current docs
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/window-functions.md](concepts/window-functions.md) | ROW_NUMBER, RANK, LAG/LEAD, frames, QUALIFY by dialect |
| [concepts/cte-patterns.md](concepts/cte-patterns.md) | Chained and recursive CTEs, materialization behaviour per engine |
| [concepts/set-operations.md](concepts/set-operations.md) | UNION/INTERSECT/EXCEPT, LATERAL, UNNEST/explode, ASOF JOIN, type coercion |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/deduplication.md](patterns/deduplication.md) | Keep-latest dedup, QUALIFY, MERGE dedup, cross-source priority |
| [patterns/gap-and-island.md](patterns/gap-and-island.md) | Date spines, islands, sessionization (incl. Spark/Databricks variants) |
| [patterns/pivot-unpivot.md](patterns/pivot-unpivot.md) | PIVOT/UNPIVOT per dialect, dynamic pivot |
| [patterns/cross-dialect.md](patterns/cross-dialect.md) | Function translation, pipe syntax, ASOF, time bucketing, DuckDB specifics |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Window Functions** | Ranking, offsets and running aggregates; `QUALIFY` filters them (Databricks DBR 10.4+, OSS Spark 4.2+) |
| **CTEs** | Chained and recursive (`WITH RECURSIVE`: OSS Spark 4.1+, Databricks DBR 17.0+) |
| **ASOF JOIN** | "Latest value as of this time" — native in DuckDB and Snowflake; window workaround elsewhere |
| **Time bucketing** | `time_bucket` (DuckDB, Spark 4.2+), `TIME_SLICE` (Snowflake), `TIMESTAMP_BUCKET` (BigQuery), `window()` (Spark) |
| **PIVOT/UNPIVOT** | Native in Snowflake, BigQuery, DuckDB, Spark/Databricks (Spark `UNPIVOT` since 3.4); Postgres via `crosstab()` |
| **Pipe syntax** | `FROM t \|> WHERE … \|> AGGREGATE …` — Spark 4.0+, Databricks DBR 16.2+, BigQuery |
| **ANSI coercion** | Spark 4 / DBR 17+: mixing STRING with numeric in UNION/`coalesce`/CASE casts the string and can fail |

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| sql-optimizer | All files | Query optimization, dialect translation |
| code-reviewer | patterns/cross-dialect.md | SQL review |
| dbt-specialist | patterns/deduplication.md | Model SQL patterns |

## Sources

- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://duckdb.org/release_calendar
