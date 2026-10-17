# Data Quality Quick Reference

> **Purpose**: Fast lookup tables for data quality tools, versions, decisions and pitfalls. For code, see linked files.
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Quality Dimensions

| Dimension | Definition | Measurement (Databricks SQL) |
|-----------|-----------|------------------------------|
| Completeness | Required fields populated | `COUNT(*) FILTER (WHERE col IS NULL) / NULLIF(COUNT(*), 0)` (ANSI: guard division) |
| Accuracy | Matches real-world truth | Reconcile with a source of record |
| Consistency | Same value across systems | Orphan check with `LEFT JOIN … IS NULL` |
| Timeliness | Available within SLA | `timestampdiff(SECOND, MAX(updated_at), current_timestamp())` |
| Uniqueness | No duplicate keys | `COUNT(*) = COUNT(DISTINCT pk)` |
| Validity | Values within accepted range/set | Accepted values, regex, range |

## Tool Status (2026-10)

| Tool | Current | What changed |
|------|---------|--------------|
| ODCS | v3.2.0 (2026-09-08) | `enum`, `context`, `${VAR}` variables, `deprecated`, `semanticType`; v3.1 added `relationships` |
| ODPS | v1.1.0 (2026-09-08) | Data product descriptor, sibling of ODCS |
| datacontract-cli | 1.2.7 (2026-10-16) | Writes ODCS v3.2.0; `export <format>` subcommands (`export dbt` removed); test engine is ibis since 1.0 |
| Soda Core | 4.27.0 (2026-10-13) | v4 contracts (`soda contract verify`); SodaCL v3 deprecated; anomaly detection = Soda Cloud |
| GX Core | 1.24.0 (2026-10-07) | Spark 4 since 1.19; Fivetran stewards GX Core; GX Cloud (FICO) shut down 2026-06-01 → no ExpectAI |
| dbt | v1.12 / v2.0 (GA 2026-09-14) | `data_tests:` + `arguments:`; dbt Cloud → dbt platform |
| Databricks | — | SDP expectations; `NOT NULL`/`CHECK`; anomaly detection (Public Preview); data profiling (ex-Lakehouse Monitoring) |
| pandera | 0.34.1 (2026-10-06) | `pandera.pyspark`; Spark 4 since 0.28 |
| PyDeequ | 1.7.0 (2026-09-14) | Spark 3.5 and 4.1 only — **not Spark 4.0 / DBR 17.3** |
| Elementary | CLI 0.24.x, dbt pkg 0.26.x | dbt-native observability (OSS + Cloud) |
| SYNQ | — | Acquired by Coalesce (2026-03-10) → Coalesce Quality |

## Decision Matrix

| Use case | Choose |
|----------|--------|
| Drop/quarantine/fail on bad rows in a Lakeflow pipeline | SDP expectations |
| Invariant for every writer of a Delta table | `NOT NULL` / `CHECK` constraint |
| "Which tables are stale or short of rows?" without rules | Databricks anomaly detection |
| Drift and profile statistics per table | Databricks data profiling |
| Validate a DataFrame in a plain Spark job | GX Core, pandera, Soda `soda-sparkdf` |
| Tests inside a dbt project | dbt data tests + model contracts |
| Vendor-neutral agreement between teams | ODCS + `datacontract lint`/`test` |
| One-dataset executable checks with warn/fail thresholds | Soda Core v4 contract |

## Contract Formats

| | ODCS v3.2.0 | Soda contract (v4) | dbt model contract |
|---|---|---|---|
| Format | `apiVersion`/`kind`/`id`/`version` + `schema[]`, `quality[]`, `slaProperties[]`, `team`, `support[]` | `dataset:` + `checks:` + `columns:` | `contract: {enforced: true}` + `columns[].data_type` + `constraints` |
| Run with | `datacontract test --server <key>` | `soda contract verify -ds … -c …` | `dbt build` |
| SLAs | `slaProperties` (`latency`, `frequency`, `availability`, `retention`) | `freshness` check | source `config.freshness` |
| Breaking changes | `datacontract breaking v1 v2` | — | model `versions` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `metadata:` / `sla:` / `references:` blocks in ODCS | Top-level fundamentals, `slaProperties[]`, `relationships[]`; lint against the JSON Schema |
| Soda `type: missing` + `fail: when …` | `- missing:` with `threshold: {must_be_less_than: …}` |
| `great_expectations checkpoint run` (CLI) | Python `checkpoint.run(batch_parameters={"dataframe": df})` |
| `tests:` with top-level test inputs | `data_tests:` with `arguments:` and `config:` |
| Rely on PK/FK/UNIQUE constraints to stop duplicates | They are informational on Databricks — assert uniqueness with a test |
| PyDeequ on DBR 17.3 LTS | GX Core ≥ 1.19, pandera ≥ 0.28 or Soda Core |
| Fixed volume thresholds | Rolling baselines or anomaly detection |

## Related Documentation

| Topic | Path |
|-------|------|
| Databricks-native quality | `concepts/databricks-data-quality.md` |
| Contracts | `concepts/data-contracts.md`, `patterns/data-contract-authoring.md` |
| Full Index | `index.md` |

## Sources

Version sources are listed in each linked concept/pattern file (ODCS releases, datacontract-cli PyPI,
Soda release notes, GX changelog, Databricks data quality monitoring docs, pandera/PyDeequ releases).
