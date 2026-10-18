# Databricks Native Data Quality

> **Purpose**: Runnable code for Databricks-native quality: SDP expectations (Python + SQL), quarantine, expectation metrics from the event log, Delta/UC constraints, anomaly detection results, data profiling via SDK
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Rows written by a Lakeflow pipeline must be warned on, dropped, quarantined or must stop the update
- An invariant must hold for every writer of a Delta table (`NOT NULL`, `CHECK`)
- You want freshness/completeness alerts for a whole schema without writing rules (anomaly detection)
- You need per-table statistics and drift over time (data profiling)

Concepts and the native-vs-library decision: [databricks-data-quality](../concepts/databricks-data-quality.md).

## Implementation

### 1. Pipeline expectations (Python, `pyspark.pipelines`)

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import expr

RULES = {
    "has_order_id": "order_id IS NOT NULL",
    "valid_status": "status IN ('pending','completed','cancelled','refunded')",
    "non_negative_amount": "amount >= 0",
}

@dp.table(name="main.sales.orders_bronze_checked")
@dp.expect("recent_order", "order_date >= '2020-01-01'")        # warn: row kept, metric recorded
@dp.expect_or_drop("has_customer", "customer_id IS NOT NULL")    # drop: row removed, count recorded
@dp.expect_or_fail("has_updated_at", "updated_at IS NOT NULL")   # fail: update stops, atomic rollback
def orders_bronze_checked():
    return spark.readStream.table("main.sales.orders_raw")

# Quarantine (official pattern): flag instead of dropping, then split
QUARANTINE = "NOT({0})".format(" AND ".join(RULES.values()))

@dp.table(private=True)                                          # pipeline-scoped, not published
@dp.expect_all(RULES)                                            # dict name -> expression, warn
def orders_quarantine_flagged():
    return (spark.readStream.table("main.sales.orders_bronze_checked")
            .withColumn("is_quarantined", expr(QUARANTINE)))

@dp.table(name="main.sales.orders_valid")
def orders_valid():
    return spark.readStream.table("orders_quarantine_flagged").filter("NOT is_quarantined")

@dp.table(name="main.sales.orders_invalid")
def orders_invalid():
    return spark.readStream.table("orders_quarantine_flagged").filter("is_quarantined")
```

`expect_all`, `expect_all_or_drop`, `expect_all_or_fail` take a dict. Grouping several rules under one action
is Python-only. Module naming and the mapping from the legacy module: [python-module-migration](../../lakeflow/concepts/python-module-migration.md);
expectation semantics in depth: [expectations-model](../../lakeflow/concepts/expectations-model.md).

### 2. Pipeline expectations (SQL) — also valid in DBSQL standalone `CREATE STREAMING TABLE` / `CREATE MATERIALIZED VIEW`

```sql
CREATE OR REFRESH STREAMING TABLE main.sales.orders_silver (
  CONSTRAINT valid_status   EXPECT (status IN ('pending','completed','cancelled','refunded')),
  CONSTRAINT has_customer   EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT has_updated_at EXPECT (updated_at IS NOT NULL)  ON VIOLATION FAIL UPDATE
) AS SELECT * FROM STREAM(main.sales.orders_raw);

-- Aggregate rule (expectations cannot aggregate): validate an MV built for the check
CREATE OR REFRESH MATERIALIZED VIEW main.sales.orders_pk_check (
  CONSTRAINT unique_order_id EXPECT (n = 1) ON VIOLATION FAIL UPDATE
) AS SELECT order_id, COUNT(*) AS n FROM main.sales.orders_silver GROUP BY order_id;
```

A failing validation MV does not block sibling tables in the same update; to gate downstream tables, put the
check in its own pipeline and chain the pipelines in a job.

### 3. Expectation metrics from the event log (official query)

```sql
CREATE OR REPLACE TEMPORARY VIEW event_log_raw AS SELECT * FROM event_log('<pipeline-id>');

WITH latest_update AS (
  SELECT origin.update_id AS latest_update_id
  FROM event_log_raw
  WHERE event_type = 'create_update'
  ORDER BY timestamp DESC
  LIMIT 1
)
SELECT row_expectations.dataset AS dataset,
       row_expectations.name    AS expectation,
       SUM(row_expectations.passed_records) AS passing_records,
       SUM(row_expectations.failed_records) AS failing_records
FROM (
  SELECT explode(from_json(details:flow_progress:data_quality:expectations,
           "array<struct<name: string, dataset: string, passed_records: int, failed_records: int>>"))
         AS row_expectations
  FROM event_log_raw, latest_update
  WHERE event_type = 'flow_progress' AND origin.update_id = latest_update.latest_update_id
)
GROUP BY row_expectations.dataset, row_expectations.name;
```

### 4. Table constraints (any writer)

```sql
ALTER TABLE main.sales.orders ALTER COLUMN order_id SET NOT NULL;           -- existing rows validated first
ALTER TABLE main.sales.orders ADD CONSTRAINT amount_non_negative CHECK (amount >= 0);
ALTER TABLE main.sales.orders ADD CONSTRAINT orders_pk PRIMARY KEY (order_id) RELY;  -- informational (UC)
ALTER TABLE main.sales.orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES main.sales.customers NOT ENFORCED RELY;      -- FK RELY: DBR 15.4+
DESCRIBE DETAIL main.sales.orders;   -- CHECK constraints appear as table properties
ALTER TABLE main.sales.orders DROP CONSTRAINT amount_non_negative;
```

A violating write fails with `CHECK_CONSTRAINT_VIOLATION` / `DELTA_VIOLATE_CONSTRAINT_WITH_VALUES`
(SQLSTATE 23001). `RELY` lets Photon remove redundant `DISTINCT` and unneeded `LEFT JOIN`s — only declare
it when the data truly satisfies the key. `UNIQUE` constraints need DBR 18.2+ (Public Preview).

### 5. Anomaly detection (Public Preview)

Enable in Catalog Explorer: schema → **Details** → **Data Quality Monitoring** → **Enable** → turn on
**Anomaly detection** (needs `MANAGE` on the schema or catalog and serverless in the workspace). Exclude
tables with `excluded_table_full_names` in the data-quality Create/Update Monitor API. Disabling deletes
the job and all results.

```sql
-- Latest health per table (system table is account-admin only by default)
SELECT catalog_name, schema_name, table_name, event_time, status,
       freshness.status    AS freshness_status,
       completeness.status AS completeness_status
FROM system.data_quality_monitoring.table_results
WHERE catalog_name = 'main' AND schema_name = 'sales'
QUALIFY ROW_NUMBER() OVER (PARTITION BY table_id ORDER BY event_time DESC) = 1;
```

### 6. Data profiling (formerly Lakehouse Monitoring) via SDK

```python
# %pip install "databricks-sdk>=0.68.0"   # the older quality_monitors API is deprecated
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.dataquality import (
    AggregationGranularity, DataProfilingConfig, Monitor, TimeSeriesConfig)

w = WorkspaceClient()
table = w.tables.get("main.sales.orders")
output_schema = w.schemas.get("main.sales_monitoring")

config = DataProfilingConfig(
    output_schema_id=output_schema.schema_id,
    assets_dir="/Workspace/Shared/data_quality/orders",
    time_series=TimeSeriesConfig(
        timestamp_column="updated_at",
        granularities=[AggregationGranularity.AGGREGATION_GRANULARITY_1_DAY]),
)
w.data_quality.create_monitor(
    monitor=Monitor(object_type="table", object_id=table.table_id, data_profiling_config=config))
# Writes main.sales_monitoring.orders_profile_metrics and orders_drift_metrics + a dashboard
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Expectation action | warn | `DROP ROW` / `expect_or_drop`, `FAIL UPDATE` / `expect_or_fail` |
| Expectation scope | — | Streaming tables, MVs, temporary views; not sinks, not `AUTO CDC FROM SNAPSHOT` |
| Profile type | — | `Snapshot` (≤ 4 TB), `TimeSeries` (timestamp + granularities), `InferenceLog` (model columns) |
| Anomaly detection privileges | — | `MANAGE` to enable; `SELECT` to see a table's quality details |

## See Also

- [databricks-data-quality](../concepts/databricks-data-quality.md)
- [pyspark-validation-libs](pyspark-validation-libs.md)
- [Lakeflow advanced expectations](../../lakeflow/patterns/expectations-advanced.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/expectation-patterns
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-expectations
- https://docs.databricks.com/aws/en/ldp/monitor-event-logs
- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/sql/user/queries/query-optimization-constraints
- https://docs.databricks.com/aws/en/error-messages/error-classes
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/results
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/create-monitor-api
