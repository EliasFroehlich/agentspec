# SCD Types

> **Purpose**: Slowly Changing Dimension Types 1–6 — trade-offs, a correct single-MERGE SCD2, and declarative SCD1/SCD2 with `AUTO CDC` on Databricks
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Slowly Changing Dimensions (SCDs) handle changes to dimension attributes over time. Type 1 overwrites, Type 2 adds a row per version with effective dates, Type 3 keeps the previous value in a column. Choose per attribute, based on whether users need historical context. On Databricks, Lakeflow pipelines implement SCD1 and SCD2 declaratively with `AUTO CDC` (the successor of `APPLY CHANGES`), which also handles out-of-order events.

## The Concept

```sql
-- SCD Type 2 dimension. Grain: one row = one version of a customer.
-- Current version: is_current = TRUE and effective_to IS NULL.
CREATE TABLE dim_customer_scd2 (
    customer_sk     BIGINT GENERATED ALWAYS AS IDENTITY,   -- Delta: disables concurrent writes
    customer_id     STRING    NOT NULL,                    -- natural key (not unique here)
    customer_name   STRING    NOT NULL,
    email           STRING,
    segment         STRING,
    region          STRING,
    effective_from  TIMESTAMP NOT NULL,
    effective_to    TIMESTAMP,
    is_current      BOOLEAN   NOT NULL
);

-- One MERGE, one commit. Changed keys appear twice in the source:
--   merge_key = customer_id -> matches and closes the current row
--   merge_key = NULL        -> never matches, inserts the new version
-- Brand-new keys appear once (merge_key = customer_id, no match) and are inserted.
-- staging_customers: customer_id, customer_name, email, segment, region, change_ts.
-- change_ts comes from the source, so the closed row and the new row share one timestamp.
MERGE INTO dim_customer_scd2 AS t
USING (
    SELECT s.customer_id AS merge_key, s.* FROM staging_customers s
    UNION ALL
    SELECT NULL AS merge_key, s.*
    FROM staging_customers s
    JOIN dim_customer_scd2 d ON d.customer_id = s.customer_id AND d.is_current
    WHERE d.customer_name IS DISTINCT FROM s.customer_name
       OR d.segment       IS DISTINCT FROM s.segment
       OR d.region        IS DISTINCT FROM s.region
) AS src
ON t.customer_id = src.merge_key AND t.is_current
WHEN MATCHED AND (t.customer_name IS DISTINCT FROM src.customer_name
               OR t.segment       IS DISTINCT FROM src.segment
               OR t.region        IS DISTINCT FROM src.region) THEN
    UPDATE SET effective_to = src.change_ts, is_current = FALSE
WHEN NOT MATCHED THEN
    INSERT (customer_id, customer_name, email, segment, region, effective_from, effective_to, is_current)
    VALUES (src.customer_id, src.customer_name, src.email, src.segment, src.region,
            src.change_ts, NULL, TRUE);
```

```sql
-- Databricks Lakeflow pipelines (serverless, Pro or Advanced): declarative SCD2
CREATE OR REFRESH STREAMING TABLE dim_customer_history;
CREATE FLOW customer_scd2 AS AUTO CDC INTO dim_customer_history
FROM stream(main.bronze.customers_cdc)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY change_ts
COLUMNS * EXCEPT (operation)
STORED AS SCD TYPE 2;     -- generates __START_AT / __END_AT; current rows: __END_AT IS NULL
-- STORED AS SCD TYPE 1 for overwrite; AUTO CDC FROM SNAPSHOT for full-snapshot sources;
-- BITEMPORAL is Beta. Identity columns are not supported on AUTO CDC targets.
```

## Quick Reference

| Type | Strategy | Storage | Query Complexity | Use When |
|------|----------|---------|-----------------|----------|
| Type 1 | Overwrite | Low | Simple | History not needed (typo fixes) |
| Type 2 | New row + dates | High | Medium (filter current) | Full history required |
| Type 3 | Previous column | Low | Simple | Only one prior value needed |
| Type 4 | Separate history table | Medium | JOIN required | Current + history separated |
| Type 6 | Hybrid (1+2+3) | High | Medium | Current + history + previous |

| Implementation | Where | Notes |
|---|---|---|
| Single MERGE with staged source | Any Delta/Iceberg engine with MERGE | Shown above; also the Delta Lake docs pattern |
| `AUTO CDC ... STORED AS SCD TYPE 2` (Python: `dp.create_auto_cdc_flow`) | Lakeflow pipelines (Databricks) | Out-of-order safe via `SEQUENCE BY`; Apache Spark 4.2 OSS Declarative Pipelines only has Auto CDC for SCD Type 1 |
| dbt snapshots | dbt | [snapshot-scd2](../../dbt/patterns/snapshot-scd2.md) |

## Common Mistakes

### Wrong

```sql
-- Two statements keyed on CURRENT_TIMESTAMP: each statement gets its own timestamp, so the
-- second INSERT finds no rows "closed at CURRENT_TIMESTAMP" and new versions are lost.
MERGE ... UPDATE SET effective_to = CURRENT_TIMESTAMP, is_current = FALSE ...;
INSERT ... WHERE t.is_current = FALSE AND t.effective_to = CURRENT_TIMESTAMP;
-- '<>' comparisons skip rows where either side is NULL; use IS DISTINCT FROM
```

### Correct

```sql
SELECT * FROM dim_customer_scd2 WHERE is_current;                 -- current rows
SELECT * FROM dim_customer_scd2                                    -- point in time
WHERE effective_from <= :report_date
  AND (effective_to > :report_date OR effective_to IS NULL);       -- :param = Databricks SQL marker
```

## Related

- [dimensional-modeling](dimensional-modeling.md)
- [schema-evolution](schema-evolution.md)
- [snapshot-scd2 pattern](../../dbt/patterns/snapshot-scd2.md)
- [Silver layer (Python SCD2)](../../medallion/concepts/silver-layer.md)
- [Lakeflow AUTO CDC pattern](../../lakeflow/patterns/cdc-apply-changes.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/limitations
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://docs.delta.io/latest/delta-update.html
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.databricks.com/aws/en/tables/features/generated-columns
