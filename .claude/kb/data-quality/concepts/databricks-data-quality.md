# Databricks Data Quality (Native Features)

> **Purpose**: The four native Databricks quality layers — pipeline expectations, Delta/Unity Catalog constraints, anomaly detection, data profiling — and when a library (GX, pandera, Soda, DQX) is the better tool
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Databricks covers data quality at four points without any external library: **expectations** in Lakeflow
Spark Declarative Pipelines (SDP, formerly Delta Live Tables) validate rows as a pipeline writes them, **table constraints** reject bad
writes from any writer, **anomaly detection** watches freshness and completeness across a whole schema, and
**data profiling** (formerly Lakehouse Monitoring) tracks per-table statistics and drift. Together they are
Unity Catalog's "data quality monitoring" plus write-time enforcement. Libraries remain the answer for rules
the native features cannot express (aggregates, cross-table checks in plain jobs, portable contracts).

## The Concept

```sql
-- 1. Write-time invariants: enforced for every writer (jobs, notebooks, SDP, external Delta clients)
CREATE TABLE main.sales.orders (
  order_id    STRING NOT NULL,
  customer_id STRING NOT NULL,
  amount      DECIMAL(12,2),
  status      STRING,
  CONSTRAINT orders_pk PRIMARY KEY (order_id) RELY      -- informational, NOT enforced
);
ALTER TABLE main.sales.orders ADD CONSTRAINT amount_non_negative CHECK (amount >= 0);

-- 2. Pipeline expectations: per-row rules with warn / drop / fail actions
CREATE OR REFRESH STREAMING TABLE main.sales.orders_clean (
  CONSTRAINT valid_status EXPECT (status IN ('pending','completed','cancelled','refunded')),
  CONSTRAINT has_customer EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT valid_amount EXPECT (amount >= 0) ON VIOLATION FAIL UPDATE
) AS SELECT * FROM STREAM(main.sales.orders_raw);
-- 3./4. Anomaly detection and data profiling are enabled on the schema / table (UI or API),
--       not in DDL. See ../patterns/databricks-native-dq.md.
```

## Quick Reference

| Layer | What it checks | Enforcement | Status / minimum |
|-------|----------------|-------------|------------------|
| SDP expectations (`@dp.expect*`, `CONSTRAINT … EXPECT`) | Row-level boolean SQL expression | `warn` (default, row kept + metric), `DROP ROW`, `FAIL UPDATE` (atomic rollback) | No preview label (GA); streaming tables, materialized views, temporary views; also DBSQL `CREATE STREAMING TABLE` / `CREATE MATERIALIZED VIEW` |
| `NOT NULL`, `CHECK` | Row-level, deterministic expression | Transaction fails on violation | Delta tables; adding one validates existing rows; upgrades writer protocol to 3 |
| `PRIMARY KEY` / `FOREIGN KEY` | Documentation + optimizer hints | **Not enforced** | Unity Catalog only; GA DBR 15.2+; `RELY` (PK 14.2+, FK 15.4+) lets Photon rewrite queries |
| `UNIQUE` | Documentation + optimizer hint | **Not enforced** | Public Preview, **DBR 18.2+** (not on DBR 17.3 LTS) |
| Anomaly detection | Table freshness (commit history) and completeness (rows in last 24 h) per schema/catalog | Alerts only (alerts in Beta) | Public Preview; serverless; `MANAGE` on schema; results in `system.data_quality_monitoring.table_results`; billed as `DATA_QUALITY_MONITORING` |
| Data profiling | Per-table profile + drift metrics (`Snapshot`, `TimeSeries`, `InferenceLog`) | Dashboards / alerts only | Formerly Lakehouse Monitoring; output tables `<table>_profile_metrics`, `<table>_drift_metrics`; SDK `w.data_quality` (`databricks-sdk>=0.68.0`) |

### Native or library?

| Need | Use |
|------|-----|
| Drop or quarantine bad rows inside a Lakeflow pipeline | SDP expectations (`expect_or_drop`, quarantine pattern) |
| Invariant that must hold for every writer, forever | `NOT NULL` / `CHECK` constraint |
| "Is any table in this schema stale or short of rows?" without writing rules | Anomaly detection |
| Distribution drift, null-rate trends, model inference logs | Data profiling |
| Uniqueness, row-count ranges, referential checks (aggregates / other tables) | Library or SQL assertion — expectations cannot use aggregates or subqueries to other tables; PK/FK/UNIQUE are not enforced |
| Validate a DataFrame in a plain job (no SDP) before writing | GX (`add_spark`), pandera (`pandera.pyspark`), Soda (`soda-sparkdf`), DQX — see [pyspark-validation-libs](../patterns/pyspark-validation-libs.md) |
| Agreement shared with other teams and tools | ODCS contract + `datacontract test` — see [data-contracts](data-contracts.md) |

Expectations limits (official): no custom Python functions, external calls or subqueries referencing other
tables; no aggregate, window or generator functions in the expression; not supported on sinks or with
`AUTO CDC FROM SNAPSHOT`; `fail` records no metrics. `@dp.expect` is a Databricks feature, not part of
Apache Spark's open-source Declarative Pipelines.

## Common Mistakes

### Wrong

```sql
-- PK is informational: duplicates are accepted silently
ALTER TABLE main.sales.orders ADD CONSTRAINT orders_pk PRIMARY KEY (order_id);
-- Aggregate/subquery inside an expectation is rejected
CONSTRAINT unique_id EXPECT ((SELECT COUNT(*) FROM main.sales.orders o2 WHERE o2.order_id = order_id) = 1)
```

### Correct

```sql
-- Enforce row-level rules natively; assert uniqueness with a separate aggregate check
ALTER TABLE main.sales.orders ADD CONSTRAINT amount_non_negative CHECK (amount >= 0);
SELECT order_id, COUNT(*) AS n FROM main.sales.orders GROUP BY order_id HAVING COUNT(*) > 1;
-- Inside SDP: an MV over GROUP BY + EXPECT (n = 1) ON VIOLATION FAIL UPDATE (official pattern)
```

## Related

- [databricks-native-dq](../patterns/databricks-native-dq.md) — runnable code for all four layers
- [pyspark-validation-libs](../patterns/pyspark-validation-libs.md) — GX, pandera, Soda, DQX, PyDeequ on DBR
- [observability](observability.md) — where anomaly detection fits among observability tools
- [Lakeflow expectations patterns](../../lakeflow/patterns/expectations.md) and [expectations model](../../lakeflow/concepts/expectations-model.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/expectation-patterns
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-streaming-table
- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-alter-table-add-constraint
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/
- https://spark.apache.org/docs/latest/declarative-pipelines-programming-guide.html
