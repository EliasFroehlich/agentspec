# Silver Layer

> **Purpose**: Cleansed, validated, deduplicated, non-aggregated records — schema enforcement, late/out-of-order data, joins and the start of modeling
> **Confidence**: 0.93
> **MCP Validated**: 2026-10-17

## Overview

Silver turns Bronze into validated, cleaned and enriched data. Databricks guidance: build Silver **from Bronze (or other Silver) tables, never directly from ingestion**; read append-only Bronze as a **stream** (batch only for small tables); keep at least one **validated, non-aggregated** representation of each record; and handle schema enforcement, nulls, deduplication, late or out-of-order data, quality checks, type casting and joins here. Modeling often starts in Silver (VARIANT, structs, flattening/normalizing). Dimensional modeling and aggregation are the Gold layer's job, although SCD history for conformed entities is often kept in Silver.

## The Pattern

```python
from pyspark.sql.functions import col, row_number, trim, upper, to_timestamp
from pyspark.sql.window import Window

def bronze_to_silver(bronze_df, business_keys: list[str]):
    """Cleanse, cast and deduplicate a Bronze batch (latest ingestion wins)."""
    cleansed = (
        bronze_df
        .withColumn("order_id", trim(col("order_id")))
        .withColumn("customer_id", upper(trim(col("customer_id"))))
        .withColumn("order_ts", to_timestamp(col("order_ts")))
        .withColumn("amount", col("amount").cast("decimal(10,2)"))   # ANSI mode (DBR 17.0+): bad casts fail; use try_cast in SQL
        .filter(col("order_id").isNotNull())
    )
    w = Window.partitionBy(*business_keys).orderBy(col("_ingested_at").desc())
    return cleansed.withColumn("_rn", row_number().over(w)).filter("_rn = 1").drop("_rn")
```

## MERGE INTO Pattern (SQL)

```sql
MERGE INTO main.silver.orders AS t
USING (
    SELECT * FROM main.bronze.raw_orders
    WHERE _ingested_at > :last_processed_ts                  -- or stream + foreachBatch
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY _ingested_at DESC) = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED AND s._ingested_at > t._updated_at THEN UPDATE SET
    t.customer_id = s.customer_id,
    t.amount      = TRY_CAST(s.amount AS DECIMAL(10,2)),
    t.order_ts    = TRY_CAST(s.order_ts AS TIMESTAMP),
    t._updated_at = s._ingested_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, amount, order_ts, _created_at, _updated_at)
VALUES (s.order_id, s.customer_id, TRY_CAST(s.amount AS DECIMAL(10,2)),
        TRY_CAST(s.order_ts AS TIMESTAMP), s._ingested_at, s._ingested_at);
```

## SCD Type 2

Prefer the declarative API when the table is built in a Lakeflow pipeline (serverless, Pro or Advanced edition) — it handles out-of-order events:

```sql
CREATE OR REFRESH STREAMING TABLE main.silver.customers_history;
CREATE FLOW customers_scd2 AS AUTO CDC INTO main.silver.customers_history
FROM stream(main.bronze.customers_cdc)
KEYS (customer_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY change_seq
COLUMNS * EXCEPT (operation, change_seq)
STORED AS SCD TYPE 2;          -- adds __START_AT / __END_AT; current rows have __END_AT IS NULL
```

Outside pipelines, use one MERGE with a staged source (Delta Lake docs pattern). `values`/`set` entries are **SQL expression strings**, so `lit(...)` cannot be used inside them:

```python
from delta.tables import DeltaTable

def apply_scd2(spark, target: str, updates_df, key: str):
    """updates_df: key, tracked columns, hash_value, valid_from (source change time)."""
    tgt = DeltaTable.forName(spark, target)
    changed = (updates_df.alias("u")
               .join(tgt.toDF().alias("t"), key)
               .where("t.is_current = true AND u.hash_value <> t.hash_value")
               .selectExpr("NULL AS merge_key", "u.*"))                 # inserts the new version
    staged = changed.unionByName(updates_df.selectExpr(f"{key} AS merge_key", "*"))
    insert_values = {c: f"s.{c}" for c in updates_df.columns}
    insert_values.update({"is_current": "true", "valid_to": "NULL"})
    (tgt.alias("t")
        .merge(staged.alias("s"), f"t.{key} = s.merge_key")
        .whenMatchedUpdate(condition="t.is_current = true AND t.hash_value <> s.hash_value",
                           set={"is_current": "false", "valid_to": "s.valid_from"})
        .whenNotMatchedInsert(values=insert_values)
        .execute())
```

## Quick Reference

| Property | Value | Notes |
|---|---|---|
| Source | Bronze/Silver tables, streaming reads | Not directly from ingestion |
| Write mode | MERGE (upsert) or `AUTO CDC` | Dedup on business keys |
| Schema | Enforced; NOT NULL/CHECK constraints are enforced on Delta | Evolve via DDL — see [schema-evolution](../patterns/schema-evolution.md) |
| Layout | `CLUSTER BY AUTO` or `CLUSTER BY (business_key, event_date)` | Not `PARTITIONED BY` business date on tables under 1 TB |
| Late data | Watermarks (`withWatermark`) or `SEQUENCE BY` in AUTO CDC | Out-of-order safe |
| Quality | Expectations / constraints / quarantine | [data-quality-gates](../patterns/data-quality-gates.md) |

## Common Mistakes

### Wrong

```python
bronze_df.write.format("delta").mode("append").saveAsTable("main.silver.orders")  # no dedup
"valid_to": "lit('9999-12-31')"     # inside whenNotMatchedInsert values: not valid SQL
```

### Correct

```python
merge_into_silver(bronze_to_silver(bronze_df, ["order_id"]), target="main.silver.orders")
"valid_to": "NULL"                  # or "CAST('9999-12-31' AS TIMESTAMP)" if you use a sentinel
```

## Related

- [Bronze Layer](bronze-layer.md)
- [Gold Layer](gold-layer.md)
- [Data Quality Gates](../patterns/data-quality-gates.md)
- [SCD types](../../data-modeling/concepts/scd-types.md)
- [Lakeflow AUTO CDC pattern](../../lakeflow/patterns/cdc-apply-changes.md) (Python: `from pyspark import pipelines as dp`, `dp.create_auto_cdc_flow`)
- [ANSI mode and TRY_* functions](../../spark/concepts/ansi-mode.md)

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion
- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.delta.io/latest/delta-update.html
- https://docs.databricks.com/aws/en/delta/merge
- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-ansi-compliance
