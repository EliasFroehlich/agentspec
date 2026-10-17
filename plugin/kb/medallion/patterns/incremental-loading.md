# Incremental Loading

> **Purpose**: Process only new or changed records between layers — streaming reads with checkpoints, MERGE upserts, change data feed, Auto Loader, high-watermark fallback
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Moving new Bronze records into Silver without full rescans
- Propagating Silver changes (inserts, updates, deletes) into Gold
- Ingesting newly arrived files exactly once
- Cutting compute versus full reloads

| Technique | Use when | Notes |
|---|---|---|
| Streaming read of the Bronze table + checkpoint | Bronze is append-only (default recommendation) | `trigger(availableNow=True)` gives batch-like runs |
| Change data feed (CDF) | Source has updates/deletes (Silver → Gold) | Legacy CDF: `delta.enableChangeDataFeed`; automatic CDF: DBR 19+ with row tracking |
| `AUTO CDC` | CDC feeds into SCD1/SCD2 inside Lakeflow pipelines | See [silver-layer](../concepts/silver-layer.md) |
| High watermark (`MAX(_updated_at)`) | No streaming or CDF available | Misses late rows with older timestamps |
| Auto Loader | Files landing in a volume or external location | Tracks processed files in the checkpoint |

## Implementation

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, current_timestamp, row_number
from pyspark.sql.window import Window


def _dedup(df, keys: list[str], order_col: str):
    w = Window.partitionBy(*keys).orderBy(col(order_col).desc())
    return df.withColumn("_rn", row_number().over(w)).filter("_rn = 1").drop("_rn")


def bronze_to_silver_stream(bronze_table: str, silver_table: str, keys: list[str], checkpoint: str):
    """Recommended: stream the append-only Bronze table; MERGE each micro-batch into Silver.
    The Silver table must already exist (created by DDL)."""
    def upsert(batch_df, batch_id: int):
        latest = _dedup(batch_df, keys, "_ingested_at").withColumn("_updated_at", current_timestamp())
        cond = " AND ".join(f"t.{k} = s.{k}" for k in keys)
        (DeltaTable.forName(batch_df.sparkSession, silver_table).alias("t")
            .merge(latest.alias("s"), cond)
            .whenMatchedUpdateAll()
            .whenNotMatchedInsertAll()
            .execute())

    return (spark.readStream.table(bronze_table)
            .writeStream
            .foreachBatch(upsert)
            .option("checkpointLocation", checkpoint)
            .trigger(availableNow=True)
            .start())


def silver_changes_to_gold(silver_table: str, gold_table: str, key: str, start_version: int):
    """Apply Silver CDF to Gold. Several changes per key in one range are collapsed to the latest,
    otherwise MERGE fails with multiple source rows matching one target row."""
    changes = (spark.read.option("readChangeFeed", "true")
               .option("startingVersion", start_version)
               .table(silver_table)
               .filter(col("_change_type").isin("insert", "update_postimage", "delete")))
    latest = _dedup(changes, [key], "_commit_version")
    cdf_cols = ["_change_type", "_commit_version", "_commit_timestamp"]
    (DeltaTable.forName(spark, gold_table).alias("t")
        .merge(latest.alias("s"), f"t.{key} = s.{key}")
        .whenMatchedDelete(condition="s._change_type = 'delete'")
        .whenMatchedUpdate(set={c: f"s.{c}" for c in latest.columns if c not in cdf_cols})
        .whenNotMatchedInsert(condition="s._change_type <> 'delete'",
                              values={c: f"s.{c}" for c in latest.columns if c not in cdf_cols})
        .execute())
    # Persist max(_commit_version) of `changes` as the next start_version (control table).
```

## Configuration

| Setting | Default | Description |
|---|---|---|
| `checkpointLocation` | Required | One per stream; on a UC volume or external location, never `/mnt` |
| `trigger(availableNow=True)` | — | Processes everything available, then stops |
| `delta.enableChangeDataFeed` | `false` | Set in `CREATE TABLE` (property changes conflict with concurrent writes); not combinable with automatic CDF |
| `startingVersion` | Latest | Batch CDF reads; versions older than log/file retention are gone after VACUUM |
| `cloudFiles.schemaLocation` | Required for inference | Stores the inferred schema; new columns stop the stream once (`addNewColumns`) |

## SQL MERGE Pattern (watermark fallback)

```sql
MERGE INTO main.silver.orders AS t
USING (
  SELECT * FROM main.bronze.raw_orders
  WHERE _ingested_at > (SELECT COALESCE(MAX(_updated_at), TIMESTAMP'1900-01-01') FROM main.silver.orders)
  QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY _ingested_at DESC) = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET
  t.customer_id = s.customer_id,
  t.amount      = TRY_CAST(s.amount AS DECIMAL(10,2)),
  t._updated_at = s._ingested_at
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, amount, _updated_at)
  VALUES (s.order_id, s.customer_id, TRY_CAST(s.amount AS DECIMAL(10,2)), s._ingested_at);
```

## Auto Loader (Streaming Incremental)

```python
def auto_loader_to_bronze(landing_path: str, bronze_table: str, checkpoint: str):
    """landing_path: '/Volumes/<catalog>/<schema>/<volume>/<dir>' or an external-location URI."""
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.schemaLocation", f"{checkpoint}/schema")
            .option("cloudFiles.inferColumnTypes", "true")
            .load(landing_path)
            .select("*", col("_metadata.file_path").alias("_source_file"),
                    current_timestamp().alias("_ingested_at"))
            .writeStream
            .option("checkpointLocation", checkpoint)
            .option("mergeSchema", "true")
            .trigger(availableNow=True)
            .toTable(bronze_table))
```

## Example Usage

```python
bronze_to_silver_stream("main.bronze.raw_orders", "main.silver.orders", ["order_id"],
                        checkpoint="/Volumes/main/ops/checkpoints/silver_orders")
```

## See Also

- [Layer Transitions](layer-transitions.md)
- [Bronze Layer](../concepts/bronze-layer.md)
- [Silver Layer](../concepts/silver-layer.md)
- [Schema Evolution](schema-evolution.md)
- [Delta operations](../../lakehouse/patterns/delta-operations.md)

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion
- https://docs.databricks.com/aws/en/structured-streaming/delta-lake
- https://docs.databricks.com/aws/en/tables/features/change-data-feed
- https://docs.databricks.com/aws/en/tables/table-properties
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/ingestion/file-metadata-column
- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/delta/merge
