# Python Streaming Patterns for Lakeflow Pipelines

> **Purpose**: Batch vs streaming reads, Auto Loader, Kafka, and a medallion example with `pyspark.pipelines`
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Ingesting files incrementally from Unity Catalog volumes or external locations
- Reading Kafka or another message bus into a streaming table
- Deciding whether a dataset is a streaming table or a materialized view

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

SOURCE = spark.conf.get("source_path")        # set in the pipeline `configuration`

# Bronze: Auto Loader into a streaming table (schema location is managed by the pipeline)
@dp.table(comment="Raw customers", cluster_by_auto=True)
def customers_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.inferColumnTypes", "true")
            .load(f"{SOURCE}/customers/")
            .select("*",
                    F.col("_metadata.file_path").alias("_source_file"),
                    F.current_timestamp().alias("_ingested_at")))

# Silver: stream from bronze, clean and validate
@dp.table(comment="Clean customers")
@dp.expect_or_drop("valid_id", "customer_id IS NOT NULL")
@dp.expect_or_drop("valid_email", "email LIKE '%@%.%'")
def customers_silver():
    return (spark.readStream.table("customers_bronze")
            .select("customer_id", "name", "email", "_ingested_at"))

# Gold: batch aggregate -> materialized view
@dp.materialized_view(comment="Customers per email domain")
def customer_domains():
    return (spark.read.table("customers_silver")
            .withColumn("domain", F.split("email", "@")[1])
            .groupBy("domain").count())

# Kafka source into a streaming table
@dp.table(comment="Raw click events")
def clicks_raw():
    return (spark.readStream.format("kafka")
            .option("kafka.bootstrap.servers", spark.conf.get("kafka_servers"))
            .option("subscribe", "clicks")
            .load()
            .select(F.col("value").cast("string").alias("payload"), "timestamp"))
```

## Configuration

| Option | Value | Notes |
|--------|-------|-------|
| `cloudFiles.format` | `json`, `csv`, `parquet`, `avro`, `text`, `binaryFile`, ... | Required |
| `cloudFiles.inferColumnTypes` | `true` | Otherwise JSON/CSV columns infer as strings |
| `cloudFiles.schemaHints` | `"amount DECIMAL(18,2)"` | Pin specific types |
| `cloudFiles.schemaLocation` | omit in pipelines | Pipelines manage schema location and checkpoints; set it in plain Structured Streaming jobs |
| Paths | `/Volumes/<catalog>/<schema>/<volume>/...` or cloud URIs | Docs recommend volumes or cloud URIs, not DBFS mounts |

## Example Usage

| Source | Dataset | Read |
|--------|---------|------|
| Files arriving over time | Streaming table | `spark.readStream.format("cloudFiles")` |
| Kafka / Event Hubs / Kinesis | Streaming table | `spark.readStream.format("kafka")` etc. |
| Delta table that only grows | Streaming table | `spark.readStream.table(...)` |
| Delta table with updates/deletes | Materialized view | `spark.read.table(...)` |
| One-time historical load into a streaming table | Append flow with `once=True` | `spark.read...` |

A streaming read of a source table that receives updates or deletes fails; for that case read with
`.option("skipChangeCommits", "true")` or use a materialized view.

```python
# CDC into a target: see cdc-apply-changes.md
dp.create_streaming_table("customers_current")
dp.create_auto_cdc_flow(
    target="customers_current", source="customers_silver",
    keys=["customer_id"], sequence_by=F.col("_ingested_at"), stored_as_scd_type=1,
)
```

## See Also

- [Python decorators](python-decorators.md)
- [Bronze ingestion](bronze-ingestion.md)
- [Flows and sinks](flows-and-sinks.md)
- [CDC with AUTO CDC](cdc-apply-changes.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-dev
- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/concepts
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/monitor-event-logs
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/ingestion/file-metadata-column
