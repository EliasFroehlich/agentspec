# SQLMesh

> **Purpose**: SQLMesh virtual data environments, change categories, plan-based deploys, built-in scheduler, governance (Linux Foundation), and the 2026 comparison with dbt v2
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

SQLMesh is an Apache-2.0 data transformation framework for SQL and Python models. It parses SQL with SQLGlot, so it can classify changes, compute minimal backfills, and show column-level lineage. Its virtual data environments let dev environments reuse production tables. It also runs existing dbt projects. Current release: **0.236.3 (2026-10-06)**. The repository is `github.com/SQLMesh/sqlmesh`.

## Ownership and Governance (2025–2026)

| Date | Event |
|------|-------|
| 2025-09-03 | Fivetran acquires Tobiko Data, the company behind SQLMesh and SQLGlot |
| 2026-03-10 | First meeting of the SQLMesh Technical Steering Committee (TSC) |
| 2026-03-25 | Fivetran contributes SQLMesh to the Linux Foundation ("will remain vendor neutral") |
| 2026-06-01 | Fivetran completes its merger with dbt Labs (announced 2025-10-13) |

SQLMesh is now "a project of the Linux Foundation" (a Series of LF Projects, LLC). A TSC with a consensus-based charter runs it. The company that funds SQLMesh development also owns dbt. So the choice between the two is about technical fit, not about competing vendors.

## The Concept

```bash
sqlmesh init duckdb          # scaffold a project for an engine (interactive if omitted)
sqlmesh plan dev             # diff vs prod, categorize changes, prompt "Apply - Backfill Tables [y/n]"
sqlmesh fetchdf "SELECT * FROM marts__dev.mart_revenue LIMIT 10"   # dev views live in <schema>__dev
sqlmesh plan                 # target prod: promote with a virtual update (no recompute when reused)
sqlmesh run                  # scheduler: run models whose cron interval is due
sqlmesh plan dev --auto-apply   # non-interactive (CI)
```

There is no separate apply command. The SQLMesh FAQ says: "There is no 'apply' command in SQLMesh". `sqlmesh plan` both builds the plan and applies it, after a prompt or with `--auto-apply`.

## Key Concepts

### Virtual Data Environments

Every model version is stored in a physical table keyed by a fingerprint. Environments are sets of views over those tables. A model `marts.mart_revenue` in environment `dev` is queried as `marts__dev.mart_revenue`. Unchanged models reuse production tables. When a dev plan is promoted, prod gets a **virtual update**: views are re-pointed, with no backfill.

### Change Categories

| Category | Directly modified model | Downstream models |
|----------|-------------------------|-------------------|
| **Breaking** | Backfilled | Backfilled |
| **Non-breaking** (e.g. new column) | Backfilled | Not backfilled |
| **Forward-only** (`--forward-only` or model setting) | Keeps the existing physical table in prod | Not backfilled |

SQLMesh categorizes changes automatically by default (`auto_categorize_changes`). Indirectly modified models inherit the category from upstream changes.

### Lineage, Linting, Tooling

- Column-level lineage is shown in the SQLMesh browser UI (`sqlmesh ui`). It also works for dbt projects run through SQLMesh. There is no `sqlmesh lineage` CLI command.
- Linter (off by default; `linter: enabled: true`). Built-in rules: `ambiguousorinvalidcolumn`, `invalidselectstarexpansion`, `noselectstar`, `nomissingaudits`, `nomissingunittest`. Custom rules go in `linter/`.
- VS Code extension (preview): model lineage view, completions, linter diagnostics, formatting.
- Engines include DuckDB, Databricks, Snowflake, BigQuery, Fabric, Spark, Trino, Postgres, Redshift and others. Tool integrations: dbt, dlt, GitHub Actions, Kestra. The current CLI reference has no Airflow integration flag.

## SQLMesh vs dbt (2026-10)

dbt v2.0 had its initial release on 2026-09-14 (version table). dbt Labs announced GA on 2026-09-16. There are two distributions:

- **"dbt"**: the former Fusion engine. The default distribution, free with no login required, with code that is not 100% Apache 2.0.
- **"dbt OSS"**: the former dbt Core v2. A subset distribution, 100% Apache 2.0.

dbt Core **1.13** (in beta) will be the last 1.x minor release.

| Feature | SQLMesh | dbt (v2) | dbt OSS (v2) |
|---------|---------|----------|--------------|
| Dev environments | Virtual (views over fingerprinted tables) | Separate target schemas | Same as dbt |
| Change impact | Semantic diff + automatic categorization | State comparison (`state:modified`) | Same |
| Backfill | Automatic, only affected models/intervals | `--full-refresh` / incremental strategies | Same |
| Scheduling | Built-in (`sqlmesh run`, cron per model) | External orchestrator or dbt platform jobs | External |
| Column-level lineage | SQLMesh UI | Local with `static_analysis: strict` (VS Code extension, Docs v2) | Not included |
| SQL comprehension, linting, LSP | SQLGlot parse + linter | Included | Not included |
| Runs dbt projects | Yes (`sqlmesh init -t dbt`) | Native | Native |
| License | Apache 2.0 | Mixed (see dbt licenses FAQ) | Apache 2.0 |

## When to Use

- **SQLMesh**: you want cheap dev environments (virtual updates), automatic change classification and backfills, or a built-in scheduler. It is also a good fit when starting a new project, or when you want to run an existing dbt project with these features.
- **dbt**: the team and its packages are already on dbt, you need dbt platform features (e.g. dbt Mesh, dbt Catalog), or you want the larger ecosystem of adapters and packages.

## Common Mistakes

### Wrong

```bash
sqlmesh apply dev                 # no such command
sqlmesh plan --enable-airflow     # not in the CLI reference
sqlmesh lineage model.column      # no such command
```

### Correct

```bash
sqlmesh plan dev                  # review, then answer the apply prompt
sqlmesh plan dev --auto-apply     # CI
sqlmesh ui                        # column-level lineage in the browser UI
```

## Trade-offs

| Pros | Cons |
|------|------|
| Virtual environments avoid copying data for dev | Smaller package/adapter ecosystem than dbt |
| Automatic change categorization and minimal backfills | State database to operate (environments, intervals) |
| Built-in scheduler and audits | Learning curve for dbt users (MODEL blocks, macros) |
| Neutral governance (Linux Foundation, TSC) | Frequent 0.x releases; pin the version |

## Related

- [sqlmesh-workflow](../patterns/sqlmesh-workflow.md)
- [analytics-engineering](../concepts/analytics-engineering.md)
- [duckdb](../concepts/duckdb.md)

## Sources

- https://github.com/SQLMesh/sqlmesh/releases
- https://github.com/SQLMesh/sqlmesh/blob/main/GOVERNANCE.md
- https://www.fivetran.com/press/fivetran-acquires-tobiko-data-to-power-the-next-generation-of-advanced-ai-ready-data-transformation
- https://www.linuxfoundation.org/press/linux-foundation-welcomes-sqlmesh-project
- https://www.fivetran.com/press/fivetran-dbt-labs-complete-merger-to-create-the-data-infrastructure-for-trusted-ai-agents
- https://sqlmesh.readthedocs.io/en/latest/reference/cli/
- https://sqlmesh.readthedocs.io/en/latest/faq/faq/
- https://sqlmesh.readthedocs.io/en/latest/concepts/plans/
- https://sqlmesh.readthedocs.io/en/latest/concepts/environments/
- https://sqlmesh.readthedocs.io/en/latest/guides/linter/
- https://sqlmesh.readthedocs.io/en/latest/guides/vscode/
- https://sqlmesh.readthedocs.io/en/latest/integrations/overview/
- https://docs.getdbt.com/docs/dbt-versions
- https://docs.getdbt.com/blog/dbt-v2-is-ga
- https://docs.getdbt.com/blog/comparing-dbt-and-dbt-oss
- https://docs.getdbt.com/docs/explore/column-level-lineage
- https://docs.getdbt.com/reference/node-selection/methods
- https://docs.getdbt.com/docs/core/connect-data-platform/connection-profiles
- https://www.getdbt.com/licenses-faq
