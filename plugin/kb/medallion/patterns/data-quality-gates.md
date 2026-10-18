# Data Quality Gates

> **Purpose**: Quality checks per Medallion layer — portable PySpark gates with quarantine, plus where Databricks-native expectations, constraints and monitoring fit
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Validating records on the Bronze → Silver and Silver → Gold transitions
- Quarantining bad records for investigation instead of silently dropping them
- Recording pass rates per rule so data contracts have measurable SLAs

## Quality by Layer

| Layer | What to check | Databricks-native mechanism | Portable mechanism |
|---|---|---|---|
| Bronze | Structural only: parseable, required keys present, batch not empty | Auto Loader rescued-data column; `CONSTRAINT ... EXPECT` (warn) on the streaming table | Gate below, warn-only |
| Silver | Business rules, types, uniqueness on keys, referential checks; quarantine | Expectations with `ON VIOLATION DROP ROW` / `FAIL UPDATE`; enforced `NOT NULL` / `CHECK` constraints | Gate below + quarantine table |
| Gold | Aggregate sanity and reconciliation against Silver | Expectations on materialized views; data quality monitoring (anomaly detection, data profiling) | Reconciliation queries + audit table |

Databricks-native details (expectations, enforced constraints, anomaly detection, data profiling — formerly Lakehouse Monitoring — and their system tables) live in [databricks-data-quality](../../data-quality/concepts/databricks-data-quality.md); this file does not duplicate them. Tool choice (Great Expectations, Soda, dbt tests) is in [../../data-quality/index.md](../../data-quality/index.md).

## Implementation

```python
from dataclasses import dataclass
from functools import reduce
from typing import Optional
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, count, current_timestamp, isnan, lit, when
from pyspark.sql.functions import sum as sum_
from pyspark.sql.types import DoubleType, FloatType


@dataclass(frozen=True)
class QualityRule:
    name: str
    column: str
    check: str               # "not_null" | "positive" | "range" | "regex" | "not_empty"
    threshold: float         # minimum pass rate, 0.0-1.0
    params: Optional[dict] = None


def _condition(df: DataFrame, rule: QualityRule):
    c = col(rule.column)
    if rule.check == "not_null":
        is_float = isinstance(df.schema[rule.column].dataType, (DoubleType, FloatType))
        return c.isNotNull() & (~isnan(c) if is_float else lit(True))
    if rule.check == "positive":
        return c > 0
    if rule.check == "range":
        return c.between(rule.params["min"], rule.params["max"])
    if rule.check == "regex":
        return c.rlike(rule.params["pattern"])
    if rule.check == "not_empty":
        return c.isNotNull() & (c != "")
    raise ValueError(f"Unknown check: {rule.check}")


def evaluate_quality_gate(df: DataFrame, rules: list[QualityRule]):
    """Return (clean_df, quarantine_df, metrics). NULL rule results count as failures."""
    flagged = df
    for r in rules:
        flagged = flagged.withColumn(f"_qc_{r.name}", when(_condition(df, r), True).otherwise(False))
    qc_cols = [f"_qc_{r.name}" for r in rules]
    all_ok = reduce(lambda a, b: a & b, [col(c) for c in qc_cols])

    # One aggregation job for all rules instead of one count() per rule
    row = flagged.agg(count(lit(1)).alias("_total"),
                      *[sum_(col(c).cast("int")).alias(c) for c in qc_cols]).first()
    total, metrics = row["_total"], []
    for r in rules:
        passed = row[f"_qc_{r.name}"] or 0
        rate = passed / total if total else 1.0
        metrics.append({"rule": r.name, "total": total, "passed": passed,
                        "pass_rate": rate, "threshold": r.threshold, "gate_passed": rate >= r.threshold})

    clean = flagged.filter(all_ok).drop(*qc_cols)
    quarantine = flagged.filter(~all_ok).withColumn("_quarantined_at", current_timestamp())
    return clean, quarantine, metrics
```

## Configuration

| Setting | Default | Description |
|---|---|---|
| `threshold` | `0.99` | Minimum pass rate for a rule; below it the gate fails |
| `quarantine_table` | `<catalog>.quarantine.<table>` | Failed records plus `_quarantined_at` |
| `halt_on_failure` | `false` | Raise when any `gate_passed` is false |
| `audit_table` | `<catalog>.quality.audit_log` | One row per rule per run |

## Example Usage

```python
order_rules = [
    QualityRule("order_id_not_null", "order_id", "not_null", threshold=1.0),
    QualityRule("amount_positive", "amount", "positive", threshold=0.99),
    QualityRule("amount_range", "amount", "range", 0.98, {"min": 0.01, "max": 999_999.99}),
]
clean, quarantine, metrics = evaluate_quality_gate(spark.table("main.bronze.raw_orders"), order_rules)
if not all(m["gate_passed"] for m in metrics):
    raise RuntimeError(f"Quality gate failed: {metrics}")

quarantine.write.mode("append").saveAsTable("main.quarantine.raw_orders")
merge_into_silver(clean, target="main.silver.orders", keys=["order_id"])   # MERGE, not append
```

```sql
-- Databricks-native: expectations on a streaming table (Lakeflow pipeline or Databricks SQL)
CREATE OR REFRESH STREAMING TABLE main.silver.orders_validated (
  CONSTRAINT valid_order_id EXPECT (order_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT positive_amount EXPECT (amount > 0)                      -- default: warn
) AS SELECT * FROM STREAM main.bronze.raw_orders;

-- Enforced constraints on any Delta table: violating writes fail the transaction
ALTER TABLE main.silver.orders ALTER COLUMN order_id SET NOT NULL;
ALTER TABLE main.silver.orders ADD CONSTRAINT amount_positive CHECK (amount > 0);

-- Audit table for gate metrics
CREATE TABLE IF NOT EXISTS main.quality.audit_log (
  run_id STRING, layer STRING, table_name STRING, rule_name STRING,
  total_records BIGINT, passed_records BIGINT, pass_rate DOUBLE,
  threshold DOUBLE, gate_passed BOOLEAN, checked_at TIMESTAMP
) CLUSTER BY (table_name, checked_at);
```

### Wrong

```sql
-- Neither statement exists in Databricks SQL
ALTER TABLE main.silver.orders SET TBLPROPERTIES ('quality.monitor.enabled' = 'true');
CREATE OR REPLACE EXPECTATION orders_quality ON TABLE main.silver.orders
EXPECT (order_id IS NOT NULL) WITH (violation_action = 'WARN');
```

Use `CONSTRAINT ... EXPECT` inside a streaming table or materialized view definition, `CHECK`/`NOT NULL` constraints on Delta tables, and enable data quality monitoring through Catalog Explorer or its API.

## See Also

- [Silver Layer](../concepts/silver-layer.md)
- [Layer Transitions](layer-transitions.md)
- [Schema Evolution](schema-evolution.md)
- [databricks-data-quality](../../data-quality/concepts/databricks-data-quality.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/lakehouse/medallion
