# datacontract-cli Workflow (ODCS on Databricks)

> **Purpose**: End-to-end lifecycle of an ODCS contract with Data Contract CLI 1.2.x — bootstrap from Unity Catalog, lint, test against a SQL warehouse or Spark session, gate breaking changes in CI, export to dbt / SQL / Soda / GX
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Bootstrapping contracts for existing Unity Catalog tables instead of writing YAML by hand
- Running the contract's schema and quality rules on a schedule or in CI
- Blocking pull requests that break consumers (removed/renamed/retyped properties)
- Keeping dbt tests, DDL or check suites generated from one source of truth

Current release: **datacontract-cli 1.2.7 (2026-10-16)**, Python 3.10–3.14. Since 1.2.0 the CLI writes
`apiVersion: v3.2.0` contracts. Legacy Data Contract Specification files (`dataContractSpecification:`)
still work in `lint`/`test`/`export`; convert them with `datacontract export odcs`.

## Implementation

```bash
# 1. Install (Databricks extra; on a cluster it reuses the runtime's PySpark)
uv tool install --python python3.11 --upgrade 'datacontract-cli[databricks]'

# 2. Authenticate (.env file or exported variables) — SQL warehouse or cluster HTTP path
export DATACONTRACT_DATABRICKS_SERVER_HOSTNAME=<workspace-host>
export DATACONTRACT_DATABRICKS_HTTP_PATH=/sql/1.0/warehouses/<warehouse-id>
export DATACONTRACT_DATABRICKS_TOKEN=<token>          # OAuth M2M and profiles: see Databricks reference

# 3. Bootstrap from Unity Catalog (generates schema + servers block); or start empty
datacontract import databricks --table main.sales.orders --output contracts/sales/orders.odcs.yaml
datacontract init contracts/sales/new.odcs.yaml

# 4. Edit: add quality[], slaProperties[], team, support — then validate the document
datacontract lint contracts/sales/orders.odcs.yaml

# 5. Test the real data (exit code 1 when a check fails)
datacontract test contracts/sales/orders.odcs.yaml --server dev
datacontract test contracts/sales/orders.odcs.yaml --server prod \
  --output test-results/TEST-orders.xml --output-format junit
datacontract test contracts/sales/orders.odcs.yaml --server dev --dry-run      # plan only, no data read
datacontract test contracts/sales/orders.odcs.yaml --server prod \
  --filter "updated_at >= current_date() - INTERVAL 1 DAY"                     # only recent rows

# 6. Compare versions (two files: before, after)
datacontract changelog main/orders.odcs.yaml contracts/sales/orders.odcs.yaml
datacontract breaking  main/orders.odcs.yaml contracts/sales/orders.odcs.yaml --perspective consumer

# 7. Generate artifacts from the contract
datacontract export dbt-models  contracts/sales/orders.odcs.yaml --output models/sales/_orders.yml
datacontract export dbt-sources contracts/sales/orders.odcs.yaml --output models/sales/_sources.yml
datacontract export sql --dialect databricks contracts/sales/orders.odcs.yaml --output ddl/orders.sql
datacontract export html contracts/sales/orders.odcs.yaml --output docs/orders.html
datacontract dbt sync contracts/sales/orders.odcs.yaml --project-dir ./warehouse --dry-run
```

```python
# Inside a Databricks notebook or job: reuse the session, no extra credentials
# (requires a runtime with Python >= 3.10; DBR 17.3 LTS ships Python 3.12)
from datacontract.data_contract import DataContract

run = DataContract(
    data_contract_file="/Volumes/main/contracts/files/orders.odcs.yaml",
    server="prod",
    spark=spark,
).test()                                    # returns datacontract.model.run.Run
if not run.has_passed():
    raise RuntimeError(run.pretty())
```

```yaml
# .github/workflows/contracts.yml — gate pull requests
name: data-contracts
on: pull_request
jobs:
  contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: {fetch-depth: 0}
      - uses: astral-sh/setup-uv@v6
      - run: uv tool install 'datacontract-cli[databricks]'
      - run: datacontract lint contracts/sales/orders.odcs.yaml
      - run: |
          git show origin/main:contracts/sales/orders.odcs.yaml > /tmp/orders.main.odcs.yaml
          datacontract breaking /tmp/orders.main.odcs.yaml contracts/sales/orders.odcs.yaml
```

## Configuration

| Command / option | Notes |
|------------------|-------|
| `test --server <key>` | Key of `servers[].server`; `all` (default) tests every server |
| `test --checks properties,quality,slaProperties,custom` | Run a subset of check categories |
| `test --quality-id`, `--tag`, `--dimension` | Run only selected rules |
| `test --include-failed-samples` | Collect a small sample of failing rows (sensitive columns omitted) |
| `breaking --perspective consumer\|producer` | `consumer` (default): changes that surprise consumers; `producer`: changes that make tests fail. Observed in 1.2.7: exit code 1 when an ERROR-level change exists |
| `export <format>` | `dbt-models`, `dbt-sources`, `dbt-staging-sql`, `sql --dialect databricks`, `spark`, `jsonschema`, `pydantic-model`, `great-expectations`, `sodacl`, `dqx`, `html`, `markdown`, `odcs`, `excel`… (`export dbt` no longer exists) |
| `export sodacl` | Emits **SodaCL (Soda v3) checks**, not a Soda v4 contract |
| Test engine | Since 1.0 checks compile to ibis expressions; `quality: [{type: custom, engine: soda}]` rules are **no longer executed** (warning) — use `type: sql` or library metrics |
| `dbt sync` / `dbt test` | Writes `data_tests:` (with `arguments:`) and `constraints` into dbt model YAML, then runs them |

## Example Usage

Exported dbt YAML (excerpt, 1.2.7): `contract: {enforced: true}`, `constraints: [{type: not_null}]`,
`data_tests: [{accepted_values: {arguments: {values: [...]}}}]` — already in the dbt 1.10.5+/v2 shape.

## See Also

- [data-contract-authoring](data-contract-authoring.md) — the contract this workflow operates on
- [data-contracts](../concepts/data-contracts.md)
- [dbt-testing](dbt-testing.md)

## Sources

- https://github.com/datacontract/datacontract-cli
- https://pypi.org/project/datacontract-cli/
- https://docs.datacontract.com/release-notes
- https://docs.datacontract.com/testing/databricks
- https://docs.datacontract.com/commands/export
- https://docs.datacontract.com/migrate-dcs-to-odcs
- CLI `--help` output of datacontract-cli 1.2.7 (run 2026-10-17)
