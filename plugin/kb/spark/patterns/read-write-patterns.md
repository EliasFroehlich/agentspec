# Read/Write Patterns

> **Purpose**: Production I/O for Unity Catalog / Delta tables, plain Parquet files and Iceberg — table-first reads/writes, clustering on write, selective overwrite, schema evolution, file metadata, VARIANT ingestion
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- Reading/writing governed tables (Unity Catalog three-level names) or raw files in cloud storage
- Selectively overwriting a slice of a table (one day, one country)
- Schema evolution is expected, or semi-structured payloads arrive as JSON

## Implementation

```python
from pyspark.sql import DataFrame
from pyspark.sql import functions as F

# --- TABLES (Unity Catalog / metastore): address by name, not by path -------------------
orders = spark.read.table("main.sales.orders")
orders_v42 = spark.read.option("versionAsOf", 42).table("main.sales.orders")      # Delta time travel

# Create with liquid clustering (DataFrameWriterV2.clusterBy / DataFrameWriter.clusterBy: Spark 4.0+)
(df.writeTo("main.sales.orders").using("delta").clusterBy("order_date").createOrReplace())

# Append / selective overwrite of one slice (Delta replaceWhere)
df.write.mode("append").saveAsTable("main.sales.orders")
(df.write.mode("overwrite")
    .option("replaceWhere", "order_date = '2026-10-16'")
    .saveAsTable("main.sales.orders"))

# --- RAW FILES ------------------------------------------------------------------------
raw = (spark.read
    .option("mergeSchema", "true")            # Parquet/ORC reader option (union of file schemas)
    .parquet("s3://bucket/raw/events/"))

# Source-file lineage: use the _metadata column (input_file_name() is not supported on
# DBR 17.3 LTS+ and is unavailable on Unity Catalog)
raw = raw.select("*", F.col("_metadata.file_path").alias("source_file"))

# Plain Parquet exports: Hive-style partition folders are still the right tool
(raw.write.mode("overwrite")
    .partitionBy("event_date")
    .option("maxRecordsPerFile", 1_000_000)
    .parquet("s3://bucket/exports/events/"))

# --- ICEBERG (catalog-managed) ---------------------------------------------------------
from pyspark.sql.functions import partitioning     # Spark 4.0+: DSv2 transforms moved here

(df.writeTo("iceberg_cat.db.events")
    .using("iceberg")
    .partitionedBy(partitioning.days("event_ts"))
    .createOrReplace())
events_snap = spark.read.option("snapshot-id", 123456789).table("iceberg_cat.db.events")
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `mergeSchema` (read) | `false` | Parquet/ORC readers: merge schemas of all files (expensive on many files) |
| `mergeSchema` (Delta write) | `false` | Adds new source columns to the target — only if your schema-change policy allows it |
| `replaceWhere` | — | Delta: atomically overwrite only rows matching the predicate |
| `partitionOverwriteMode` | `static` | `dynamic` = overwrite only partitions present in the output (partitioned tables/files) |
| `maxRecordsPerFile` | `0` (unlimited) | Cap rows per output file |
| `spark.sql.extensions` / `spark.sql.catalog.spark_catalog` | — | Needed **only on OSS Spark** for Delta (`io.delta.sql.DeltaSparkSessionExtension`, `org.apache.spark.sql.delta.catalog.DeltaCatalog`); built in on Databricks |

## Example Usage

```python
def load_day(date: str) -> None:
    """Raw files -> curated Delta table, idempotent per day."""
    raw = spark.read.parquet(f"s3://bucket/raw/orders/dt={date}")
    curated: DataFrame = transform_orders(raw)       # see concepts/dataframe-api.md
    (curated.write.mode("overwrite")
        .option("replaceWhere", f"order_date = '{date}'")
        .saveAsTable("main.sales.orders"))
```

### Semi-structured data with VARIANT (Spark 4.0+, Databricks DBR 15.4+)

```python
events = spark.sql("""
    SELECT id, parse_json(raw_json) AS payload
    FROM main.bronze.raw_events
""")
events.select(
    "id",
    F.variant_get("payload", "$.user.name", "string").alias("user_name"),
    F.variant_get("payload", "$.items[0].price", "decimal(10,2)").alias("first_item_price"),
)
spark.sql("CREATE TABLE main.bronze.events (event_id BIGINT, event_data VARIANT)")
```

`F.col("payload:user.name")` does not work: `F.col` resolves a column name. The `:` path syntax is
SQL (Databricks DBR 15.4+, OSS Spark 4.1+): `SELECT payload:user.name::string FROM ...`.

## See Also

- [delta-integration](../patterns/delta-integration.md)
- [partitioning](../concepts/partitioning.md)
- [ansi-mode](../concepts/ansi-mode.md)

## Sources

- https://docs.databricks.com/aws/en/tables/clustering
- https://spark.apache.org/releases/spark-release-4-0-0.html
- https://spark.apache.org/docs/latest/sql-data-sources-parquet.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.partitioning.days.html
- https://docs.databricks.com/aws/en/sql/language-manual/functions/input_file_name
- https://docs.databricks.com/aws/en/ingestion/file-metadata-column
- https://docs.databricks.com/aws/en/semi-structured/variant
- https://spark.apache.org/docs/4.0.0/api/python/reference/pyspark.sql/api/pyspark.sql.functions.variant_get.html
- https://docs.delta.io/latest/delta-batch.html
