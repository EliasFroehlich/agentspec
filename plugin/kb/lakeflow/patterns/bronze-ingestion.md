# Bronze Ingestion Patterns

> **Purpose**: Incremental raw ingestion into streaming tables with Auto Loader / `read_files`, file metadata via `_metadata`, and warn-level quality
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Files land in a Unity Catalog volume or external location (S3, ADLS, GCS) and must be loaded once
- Several file families (for example `orders/`, `returns/`) share one ingestion convention
- You need lineage columns (source file, ingestion time) on every bronze row

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

LANDING = spark.conf.get("landing_path")      # e.g. /Volumes/main/landing/files

def bronze_reader(subdir: str, fmt: str = "parquet"):
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", fmt)
            .option("cloudFiles.inferColumnTypes", "true")
            .option("cloudFiles.schemaHints", "amount DECIMAL(18,2)")
            .load(f"{LANDING}/{subdir}/")
            .select(
                "*",
                F.col("_metadata.file_path").alias("_source_file"),
                F.col("_metadata.file_modification_time").alias("_source_file_ts"),
                F.current_timestamp().alias("_ingested_at"),
            ))

@dp.table(
    name="orders_bronze",
    comment="Raw orders as delivered",
    table_properties={"quality": "bronze"},
    cluster_by_auto=True,
)
@dp.expect("has_order_id", "order_id IS NOT NULL")          # warn only in bronze
@dp.expect("no_rescued_data", "_rescued_data IS NULL")
def orders_bronze():
    return bronze_reader("orders")

# Several file families with one function each (default arg binds the loop value)
for family in ["returns", "shipments"]:
    @dp.table(name=f"{family}_bronze", table_properties={"quality": "bronze"})
    def family_bronze(subdir=family):
        return bronze_reader(subdir)
```

```sql
-- SQL equivalent with read_files (option names without the cloudFiles. prefix)
CREATE OR REFRESH STREAMING TABLE orders_bronze (
  CONSTRAINT has_order_id EXPECT (order_id IS NOT NULL)
)
CLUSTER BY AUTO
TBLPROPERTIES ("quality" = "bronze")
AS SELECT *,
  _metadata.file_path AS _source_file,
  _metadata.file_modification_time AS _source_file_ts,
  current_timestamp() AS _ingested_at
FROM STREAM read_files("${landing_path}/orders/", format => "parquet", inferColumnTypes => true);
```

## Configuration

| Option | Value | Notes |
|--------|-------|-------|
| `cloudFiles.format` | `parquet`, `json`, `csv`, ... | Required |
| `cloudFiles.inferColumnTypes` | `true` | JSON/CSV/XML otherwise infer every column as string |
| `cloudFiles.schemaHints` | `"amount DECIMAL(18,2)"` | Pin types that inference gets wrong |
| `cloudFiles.schemaLocation` | omit | Lakeflow pipelines manage schema location and checkpoints |
| `pipelines.autoOptimize.managed` | default `true` | No need to set it |
| Source path | `/Volumes/...` or `s3://`, `abfss://`, `gs://` | Avoid DBFS mounts |

| `_metadata` field | Type | Min DBR |
|-------------------|------|---------|
| `file_path` | STRING | 10.5 |
| `file_name` | STRING | 10.5 |
| `file_size` | LONG | 10.5 |
| `file_modification_time` | TIMESTAMP | 10.5 |
| `file_block_start`, `file_block_length` | LONG | 13.0 |

`input_file_name()` is deprecated since DBR 13.3 LTS, not available on Unity Catalog, and no longer
supported in Databricks SQL and DBR 17.3 LTS+; use `_metadata.file_path` / `_metadata.file_name`.

## Example Usage

```python
# Wrong: deprecated function, mount path, and an unformatted "${env}" in a Python string
.load("s3://my-bucket-${env}/orders/").withColumn("_source_file", F.input_file_name())  # Wrong

# Correct
env = spark.conf.get("env")
spark.readStream.format("cloudFiles").option("cloudFiles.format", "parquet") \
    .load(f"s3://my-bucket-{env}/orders/") \
    .select("*", F.col("_metadata.file_path").alias("_source_file"))
```

If the source data already has a `_metadata` column, rename it (for example `source_metadata`) or the
file metadata column is shadowed. Keep bronze rows as delivered; do typing and business rules in
silver.

## See Also

- [Silver cleansing](silver-cleansing.md)
- [Python streaming](python-streaming.md)
- [Lakeflow Connect](../concepts/lakeflow-connect.md)
- [Expectations](expectations.md)

## Sources

- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://docs.databricks.com/aws/en/ingestion/file-metadata-column
- https://docs.databricks.com/aws/en/sql/language-manual/functions/input_file_name
- https://docs.databricks.com/aws/en/sql/language-manual/functions/read_files
- https://docs.databricks.com/aws/en/ldp/developer/python-dev
- https://docs.databricks.com/aws/en/ldp/properties
