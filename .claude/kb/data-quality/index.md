# Data Quality Knowledge Base

> **Purpose**: Data quality, contracts and observability — Databricks-native quality (expectations, constraints, anomaly detection, data profiling), ODCS contracts with datacontract-cli, Soda Core v4, GX Core 1.x, pandera, dbt tests
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17
> **Latest**: ODCS v3.2.0 + ODPS v1.1.0 (2026-09-08), datacontract-cli 1.2.7, Soda Core 4.27, GX Core 1.24 (Fivetran-stewarded; GX Cloud shut down), dbt v1.12 / v2.0, Databricks data quality monitoring (anomaly detection in Public Preview)

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/databricks-data-quality.md](concepts/databricks-data-quality.md) | SDP expectations, Delta/UC constraints, anomaly detection, data profiling — and when to use a library instead |
| [concepts/data-contracts.md](concepts/data-contracts.md) | ODCS v3.2.0 structure, ODPS, dbt and Soda contracts compared |
| [concepts/quality-dimensions.md](concepts/quality-dimensions.md) | Completeness, accuracy, consistency, timeliness, uniqueness, validity (Databricks SQL) |
| [concepts/observability.md](concepts/observability.md) | Freshness, volume, distribution drift; tool landscape |
| [concepts/soda-core.md](concepts/soda-core.md) | Soda Core v4 contract language, CLI, Core vs Cloud |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/databricks-native-dq.md](patterns/databricks-native-dq.md) | Expectations (Python/SQL), quarantine, event-log metrics, constraints, anomaly detection results, data profiling SDK |
| [patterns/pyspark-validation-libs.md](patterns/pyspark-validation-libs.md) | GX, pandera, Soda `soda-sparkdf`, PyDeequ on Spark — DBR compatibility matrix |
| [patterns/great-expectations.md](patterns/great-expectations.md) | GX Core 1.x suites, validation definitions, checkpoints, `batch_parameters` |
| [patterns/dbt-testing.md](patterns/dbt-testing.md) | `data_tests` + `arguments`, contracts, constraints, freshness, stored failures |
| [patterns/schema-validation.md](patterns/schema-validation.md) | JSON Schema, Pydantic, Avro compatibility; DataFrame/Delta enforcement pointers |
| [patterns/data-contract-authoring.md](patterns/data-contract-authoring.md) | Complete ODCS v3.2.0 contract validated against the official JSON Schema |
| [patterns/datacontract-cli-workflow.md](patterns/datacontract-cli-workflow.md) | import → lint → test → breaking → export with datacontract-cli on Databricks |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Native first on Databricks** | Expectations for pipelines, constraints for invariants, anomaly detection for freshness/completeness |
| **Contracts** | ODCS is the vendor-neutral format; dbt and Soda contracts enforce parts of it |
| **Rules vs baselines** | Tests assert known rules; observability learns normal behavior |

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| data-quality-analyst | All files | Quality rules, native Databricks quality, monitoring |
| data-contracts-engineer | concepts/data-contracts.md, patterns/data-contract-authoring.md, patterns/datacontract-cli-workflow.md | ODCS contracts, SLAs, breaking changes |
| code-reviewer | patterns/dbt-testing.md, concepts/databricks-data-quality.md | Data engineering review |
| test-generator | patterns/great-expectations.md, patterns/pyspark-validation-libs.md | GX / pandera test generation |

## Sources

- https://github.com/bitol-io/open-data-contract-standard/releases
- https://pypi.org/project/datacontract-cli/
- https://docs.soda.io/release-notes/soda-core
- https://docs.greatexpectations.io/docs/core/changelog/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/
