# Soda Core

> **Purpose**: Soda Core v4 data contracts — real contract grammar, thresholds, CLI and Python API (incl. Spark DataFrames on Databricks), and what is Soda Core (OSS) vs Soda Cloud (paid)
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Soda Core is an open-source data-contract verification engine: a contract YAML declares checks for **one
dataset**, and `soda contract verify` evaluates them against a data source. Version 4 (first public release
4.0.5, February 2026; current **4.27.0, 2026-10-13**) replaced the v3 SodaCL language (`checks for ...`),
which is deprecated. Packages are `soda-<datasource>` (`soda-databricks`, `soda-sparkdf`, `soda-duckdb`…),
Python ≥ 3.10. Anomaly detection, metric monitors and AI features belong to **Soda Cloud**, not to Soda Core.

## The Concept

```yaml
# contracts/orders.yml — Soda Core v4 contract (Databricks: datasource/catalog/schema/table)
dataset: my_databricks/main/sales/orders

variables:
  MIN_ROWS:
    default: 1000

checks:                                   # dataset-level checks
  - schema:
      allow_extra_columns: false          # default false: every column must be listed
  - row_count:
      threshold:
        must_be_greater_than_or_equal: ${var.MIN_ROWS}
  - freshness:
      column: updated_at
      threshold:
        unit: hour
        must_be_less_than: 24
        additional:                       # Soda Core 4.22+: second threshold on the same metric
          must_be_less_than: 6
          level: warn

columns:                                  # required, even if empty: columns: []
  - name: order_id
    data_type: varchar
    checks:
      - missing:                          # default threshold: no missing values
      - duplicate:                        # default threshold: all values unique
  - name: customer_id
    checks:
      - missing:
          threshold:
            metric: percent
            must_be_less_than: 1
  - name: status
    checks:
      - invalid:
          valid_values: ['pending', 'completed', 'cancelled', 'refunded']
  - name: amount
    checks:
      - invalid:
          valid_min: 0
      - aggregate:
          function: avg
          threshold:
            must_be_between:
              greater_than: 10
              less_than: 500
  - name: updated_at
```

This contract (with a DuckDB dataset name) passed `soda contract test` and `soda contract verify`
(9/9 checks) on Soda Core 4.27.0.

## Quick Reference

| Check (key) | Level | Key options |
|-------------|-------|-------------|
| `schema` | dataset | `allow_extra_columns`, `allow_other_column_order`; uses `columns[].data_type`, `character_maximum_length`, `numeric_precision`, `numeric_scale` |
| `row_count` | dataset | default: at least 1 row |
| `freshness` | dataset | `column` (required), `threshold.unit` (`minute`, `hour`, `day`) |
| `missing` | column | `missing_values: [...]`, `missing_format: {name, regex}` |
| `invalid` | column | `valid_values`, `valid_format`, `valid_min`/`valid_max`, `valid_length`, `valid_reference_data: {dataset, column}`, `invalid_values` |
| `duplicate` | column / dataset | dataset level: `columns: [a, b]` |
| `aggregate` | column | `function`: `avg`, `sum`, `min`, `max`, `avg_length`, `min_length`, `max_length` |
| `metric` | both | `expression` or `query` + threshold |
| `failed_rows` | both | `expression` (WHERE clause) or `query`; percent needs `rows_tested_query` (licensed) |

Thresholds: `must_be`, `must_not_be`, `must_be_greater_than[_or_equal]`, `must_be_less_than[_or_equal]`,
`must_be_between` / `must_be_not_between` with nested `greater_than[_or_equal]` + `less_than[_or_equal]`;
`metric: count|percent`; `level: fail|warn`. Common keys: `name`, `qualifier` (needed for a second check of
the same type), `filter`, `column_expression`, `attributes`. Dataset-wide `filter:` and `variables:`
(`${var.NAME}`, built-in `${soda.NOW}`, override with `--set NAME=value`).

| Soda Core (open source) | Soda Cloud (licensed) |
|-------------------------|-----------------------|
| Contract language, `soda` CLI, Python API, public packages (`pypi.org`, `pypi.cloud.soda.io`) | Anomaly detection / metric monitors, dashboards, incidents, notifications |
| Local verification, exit codes for pipelines | Soda Runner/Agent, `--publish` results, contract editor, Contract Autopilot/Copilot |
| — | Private-PyPI extensions: `group_by` and reconciliation checks (`soda-groupby`, `soda-reconciliation`), `soda-migration` (v3 → v4), `soda-scopes` |

```bash
pip install soda-databricks                       # or soda-sparkdf for DataFrames in a Spark session
soda data-source create -f ds_config.yml          # scaffold; Databricks: type, host, http_path, catalog, auth
soda data-source test -ds ds_config.yml
soda contract test -c contracts/orders.yml        # syntax dry run
soda contract verify -ds ds_config.yml -c contracts/orders.yml --set MIN_ROWS=500
# exit codes: 0 pass · 1 failed check · 2 warn only · 3 could not run · 4 could not send to Soda Cloud
```

Python API on Databricks (`soda-sparkdf`, `verify_contract_locally`): see
[pyspark-validation-libs](../patterns/pyspark-validation-libs.md#soda-core-v4-on-a-spark-session).

## Common Mistakes

### Wrong

```yaml
dataset: orders              # not a fully qualified name; there is no `source:` key
columns:
  - name: order_id
    checks: [{type: missing, fail: when count > 0}]   # no `type:` key, no `fail: when`
```

### Correct

```yaml
dataset: my_databricks/main/sales/orders
columns:
  - name: order_id
    checks:
      - missing:
          threshold:
            must_be: 0
```

Legacy v3 (`checks for orders:` / `missing_count(col) = 0`) runs only on `soda-core-<ds>~=3.5` packages.

## Related

- [data-contracts](data-contracts.md) — ODCS, the vendor-neutral contract format
- [pyspark-validation-libs](../patterns/pyspark-validation-libs.md)
- [great-expectations](../patterns/great-expectations.md)

## Sources

- https://docs.soda.io/reference/contract-language-reference
- https://docs.soda.io/reference/cli-reference
- https://docs.soda.io/deployment-options/soda-python-libraries
- https://docs.soda.io/reference/data-source-reference-for-soda-core/databricks
- https://docs.soda.io/reference/data-source-reference-for-soda-core/spark-dataframe
- https://docs.soda.io/release-notes/soda-core
- https://github.com/sodadata/soda-core
- https://pypi.org/project/soda-core/
