# dbt v2 and the Fusion Engine

> **Purpose**: What dbt v2 is (the Rust engine formerly called Fusion), its distributions and licenses, what changes from dbt v1, and how to migrate safely
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

dbt Labs acquired SDF Labs (a Rust SQL compiler) on 2025-01-14 and built the **Fusion engine** on it. On
2026-06-01 the runtime was opened under **Apache 2.0** as the first alpha of dbt "Core" v2.0, and
**dbt v2.0 reached GA on 2026-09-14** (active support until 2027-09-13). There are now two
engine generations: **dbt v1** (Python; v1.12 is the current minor) and **dbt v2** (Rust). dbt v2 ships as
two distributions: **dbt** (formerly "dbt Fusion engine", recommended, proprietary under the dbt Product
Licensing Agreement) and **dbt OSS** (100 % Apache 2.0 runtime). The name "dbt Core" no longer applies to
any v2 distribution. "dbt Cloud" is now **dbt platform**. dbt Labs and Fivetran are one company (merger
official 2026-06-01); dbt OSS remains Apache 2.0.

## The Concept

```bash
# dbt v2 (the "dbt" distribution) — pip or the install script
python -m pip install dbt
curl -fsSL https://public.cdn.getdbt.com/fs/install/install.sh | sh -s -- --update
dbt system update                    # update a script/binary install (channel: latest)
dbt system update --version canary   # opt into the canary channel

# dbt v1 (Python) — adapter package pulls dbt-core 1.x
python -m pip install dbt-databricks

# Before moving a v1 project to v2: surface and fix every deprecation
dbt parse --no-partial-parse --show-all-deprecations     # on v1 (omit --no-partial-parse on v2)
uvx dbt-autofix deprecations --dry-run                   # preview fixes
uvx dbt-autofix deprecations                             # apply (also: packages, jobs subcommands)
```

## Quick Reference

| Item | dbt v1 (Python) | dbt v2 (Rust) |
|------|-----------------|---------------|
| Current version | 1.12 (2026-07-16, active to 2027-07-15); 1.11 critical support; 1.9/1.10 deprecated; 1.13 in beta = final 1.x minor | 2.0 GA 2026-09-14; 2.0.8 stable/latest (2026-10-14); 2.0.9 canary |
| License | Apache 2.0 (`dbt-core`) | **dbt** distribution: dbt Product Licensing Agreement · **dbt OSS**: Apache 2.0 |
| Release channels | PyPI versions | Local: `latest`, `canary`, `dev`; dbt platform tracks: v2 Nightly, Stable (default), Extended, Fallback |
| YAML strictness | Deprecation warnings | Some deprecations are errors — e.g. generic-test inputs **must** be under `arguments:` |
| Adapters | All dbt v1 adapters | GA: Snowflake, BigQuery, Databricks, Redshift, DuckDB (CLI); Apache Spark (CLI) beta |
| Free features | — | Syntax error detection, `dbt lint`, dbt docs v2, LSP lite (go-to ref/source/macro) |
| With a dbt platform login (free tier included) | — | Full LSP, SQL comprehension / type checking / impact analysis, precise column-level lineage, Catalog tab |
| Unit tests | Run on the warehouse | Same; experimental local DuckDB execution (`config: {compute: local}` + `DBT_ENGINE_EXPERIMENTAL_LOCAL_UNIT_TESTS=true`) only for Snowflake/BigQuery projects |

The open repository `github.com/dbt-labs/dbt` (Apache-2.0) contains the Rust runtime as crates such as
`dbt-jinja` and `dbt-parser`.

### Migration v1 → v2

| Step | Action |
|------|--------|
| 1. Upgrade v1 | Move to 1.12 and enable the behavior-change flags (1.13 makes the 1.9/1.10 flags default `true`) |
| 2. Clear deprecations | `dbt parse --show-all-deprecations`; `dbt-autofix deprecations` (v0.22.x) moves test inputs under `arguments:`, `meta`/`tags` under `config:`, legacy semantic-layer YAML (`--semantic-layer`) |
| 3. Check packages | `dbt-autofix packages`; pin package versions that declare v2 support (`require-dbt-version` < 3.0.0) |
| 4. Compare | Build in dev with v2 and diff compiled SQL and row counts against v1 |
| 5. Promote | Switch the dbt platform environment track or the CI image; keep a v1 fallback until parity is proven |

## Common Mistakes

### Wrong

```yaml
# Accepted by v1 with a deprecation warning; a validation error on dbt v2
columns:
  - name: status
    data_tests:
      - accepted_values:
          values: ['placed', 'shipped']
```

### Correct

```yaml
columns:
  - name: status
    data_tests:
      - accepted_values:
          arguments:
            values: ['placed', 'shipped']
          config:
            severity: warn
```

## Related

- [testing-framework](testing-framework.md) — `data_tests` / `arguments` / `config`
- [dbt-databricks-adapter](dbt-databricks-adapter.md)
- [mesh-architecture](mesh-architecture.md)

## Sources

- https://www.getdbt.com/blog/dbt-labs-announces-sdf-labs-acquisition
- https://docs.getdbt.com/blog/dbt-core-v2-is-here
- https://docs.getdbt.com/docs/dbt-versions
- https://docs.getdbt.com/docs/dbt/dbt-releases
- https://docs.getdbt.com/docs/dbt/dbt-availability
- https://docs.getdbt.com/docs/local/install-dbt
- https://www.getdbt.com/licenses-faq
- https://www.getdbt.com/blog/fivetran-and-dbt-are-one-company-now-here-s-what-that-means
- https://docs.getdbt.com/reference/deprecations
- https://github.com/dbt-labs/dbt-autofix
- https://docs.getdbt.com/docs/build/unit-tests
