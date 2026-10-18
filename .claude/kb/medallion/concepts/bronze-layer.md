# Bronze Layer

> **Purpose**: Raw ingestion layer — incremental append, source fidelity (string/VARIANT/binary), file metadata via `_metadata`, minimal validation
> **Confidence**: 0.93
> **MCP Validated**: 2026-10-17

## Overview

Bronze keeps the raw state of each source, appended incrementally, as the replayable source of truth for audit and reprocessing. Databricks guidance: perform **minimal** validation here, store most fields as **string, VARIANT or binary** to survive unexpected schema changes, and add provenance columns such as `_metadata.file_name`. Bronze is read by pipelines that build Silver — not by analysts — and those reads should usually be **streaming** reads. Sources: cloud object storage or Unity Catalog volumes (Auto Loader / `read_files`), message buses (Kafka source), or federated systems.

## The Pattern

```python
from pyspark.sql.functions import col, current_timestamp, lit

def stream_files_to_bronze(source_path: str, bronze_table: str, checkpoint: str, source_system: str):
    """Auto Loader -> Bronze. source_path: '/Volumes/<catalog>/<schema>/<volume>/<dir>' or a UC
    external-location URI. Requires Databricks (Auto Loader); _metadata needs DBR 10.5+."""
    return (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schemaLocation", f"{checkpoint}/schema")  # inference + _rescued_data
        .load(source_path)
        .select(
            "*",
            col("_metadata.file_path").alias("_source_file"),
            col("_metadata.file_modification_time").alias("_source_file_mtime"),
            current_timestamp().alias("_ingested_at"),
            lit(source_system).alias("_source_system"),
        )
        .writeStream
        .option("checkpointLocation", checkpoint)
        .option("mergeSchema", "true")          # writer option: lets new source columns land
        .trigger(availableNow=True)
        .toTable(bronze_table)
    )
```

JSON fields are inferred as strings by default; new columns stop the stream once (`addNewColumns` mode) and the schema location is updated — run it under a Lakeflow Job with automatic retries ([lakeflow-jobs](../../lakeflow/reference/lakeflow-jobs.md)). To keep each record as one VARIANT column (DBR 15.3+ for JSON), use `.option("singleVariantColumn", "raw")`; rescued-data columns and schema evolution then do not apply.

## Quick Reference

| Property | Value | Notes |
|---|---|---|
| Write mode | Append (streaming or incremental batch) | No MERGE/overwrite in Bronze |
| Field types | String, VARIANT or binary | VARIANT: DBR 15.4+ type; ingest as VARIANT DBR 15.3+ (JSON), 16.4+ (CSV/XML) |
| Provenance | `_metadata.file_path`, `file_name`, `file_modification_time` | Select `_metadata` explicitly in the read |
| Ingest time | `_ingested_at = current_timestamp()` | Set in the query, not a column DEFAULT |
| Layout | `CLUSTER BY AUTO` (UC managed) or `CLUSTER BY (_ingested_at)` | Not `PARTITIONED BY` date on tables under 1 TB |
| Paths | `/Volumes/...` or external-location URIs | DBFS mounts (`/mnt`) are deprecated and incompatible with serverless |
| Downstream reads | `spark.readStream.table(bronze)` | Batch reads only for small dimension-like tables |

## Common Mistakes

### Wrong

```python
from pyspark.sql.functions import input_file_name   # not available on Unity Catalog;
df = (spark.read.option("mergeSchema", "true")      # deprecated in DBR 13.3 LTS, no longer
      .json("/mnt/landing/orders/")                  # supported in DBR 17.3 LTS and above
      .withColumn("_source_file", input_file_name()))
# mergeSchema is a Parquet/ORC reader option (ignored by the JSON reader); /mnt is deprecated.
# Auto Loader ("cloudFiles") reads files only — it is not a Kafka consumer.
```

### Correct

```python
from pyspark.sql.functions import col, parse_json
df = (spark.read.format("json").load("/Volumes/main/landing/raw/orders/")
      .select("*", col("_metadata.file_path").alias("_source_file")))

kafka_bronze = (                                     # Kafka: use the Kafka source
    spark.readStream.format("kafka")
    .option("kafka.bootstrap.servers", "<host:port>")
    .option("subscribe", "orders")
    .option("startingOffsets", "earliest")
    .load()
    .select(col("key").cast("string"), parse_json(col("value").cast("string")).alias("payload"),
            col("topic"), col("partition"), col("offset"), col("timestamp")))
```

## SQL Alternatives

```sql
-- Lakeflow pipeline or Databricks SQL streaming table (read_files options take no cloudFiles. prefix)
CREATE OR REFRESH STREAMING TABLE main.bronze.raw_orders
CLUSTER BY AUTO
AS SELECT *, _metadata.file_path AS _source_file, current_timestamp() AS _ingested_at
FROM STREAM read_files('/Volumes/main/landing/raw/orders/', format => 'json');

-- Plain Delta table with a raw VARIANT payload
CREATE TABLE IF NOT EXISTS main.bronze.raw_events (
  raw            VARIANT,
  _source_file   STRING,
  _ingested_at   TIMESTAMP,
  _source_system STRING
) CLUSTER BY (_ingested_at);

-- Iceberg (Spark + Iceberg runtime; format v3 needed for the variant type)
CREATE TABLE IF NOT EXISTS catalog.bronze.raw_orders (
  raw_json STRING, _source_file STRING, _ingested_at TIMESTAMP, _source_system STRING
) USING iceberg PARTITIONED BY (day(_ingested_at))
TBLPROPERTIES ('format-version' = '3');
```

## Shift-Left Checks at Bronze

Keep checks structural and cheap (file parseable, required keys present, batch not empty); route bad records to the rescued-data column or a quarantine table instead of dropping them. Business rules belong in Silver — see [data-quality-gates](../patterns/data-quality-gates.md).

## Related

- [Silver Layer](silver-layer.md)
- [Layer Transitions](../patterns/layer-transitions.md)
- [Incremental Loading](../patterns/incremental-loading.md)
- [Liquid clustering](../../lakehouse/concepts/liquid-clustering.md)
- [Lakeflow bronze ingestion](../../lakeflow/patterns/bronze-ingestion.md) and [Python module migration](../../lakeflow/concepts/python-module-migration.md) (`@dp.table` instead of `import dlt`)

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion
- https://docs.databricks.com/aws/en/sql/language-manual/functions/input_file_name
- https://docs.databricks.com/aws/en/ingestion/file-metadata-column
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/ingestion/variant
- https://docs.databricks.com/aws/en/sql/language-manual/functions/read_files
- https://docs.databricks.com/aws/en/dbfs/mounts
- https://spark.apache.org/docs/latest/sql-data-sources-json.html
- https://iceberg.apache.org/docs/latest/spark-ddl/
