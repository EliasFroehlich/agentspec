# Getting Started with Lakeflow Pipelines

> **Purpose**: First pipeline end to end: Lakeflow Pipelines Editor, folder layout, bronze/silver/gold with CDC, scheduling
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Overview

New pipelines are created from the Lakeflow Pipelines Editor with Unity Catalog, the `current`
channel, and serverless compute as defaults. Source code lives in workspace files (`.py`/`.sql`)
under a `transformations/` folder in the pipeline root folder. The official tutorial builds a CDC
pipeline: Auto Loader ingest, cleaned stream with expectations, then `AUTO CDC` into SCD tables.

## The Concept

### Folder layout (created by the editor)

| Folder | Holds | Evaluated by the pipeline? |
|--------|-------|----------------------------|
| `transformations/` | Pipeline source (`.py`, `.sql`) | Yes |
| `explorations/` | Ad hoc notebooks and queries | No |
| `utilities/` | Python modules imported by source files | Imported only |

Create from: New > ETL pipeline, a workspace or Git folder (Create > ETL pipeline), or "Create a
source-controlled project" (Declarative Automation Bundles project).

### Minimal CDC pipeline (Python)

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, expr

# Bronze: incremental file ingestion with Auto Loader
@dp.table(comment="Raw customer CDC events")
def customers_cdc_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load("/Volumes/main/landing/files/customers/"))

# Silver: drop malformed events
@dp.table
@dp.expect_or_drop("valid_id", "id IS NOT NULL")
@dp.expect_or_drop("valid_op", "operation IN ('APPEND', 'UPDATE', 'DELETE')")
def customers_cdc_clean():
    return spark.readStream.table("customers_cdc_bronze")

# Gold: current state (SCD Type 1)
dp.create_streaming_table("customers")
dp.create_auto_cdc_flow(
    target="customers",
    source="customers_cdc_clean",
    keys=["id"],
    sequence_by=col("operation_date"),
    apply_as_deletes=expr("operation = 'DELETE'"),
    except_column_list=["operation", "operation_date"],
    stored_as_scd_type=1,
)

# Gold: full history (SCD Type 2 -> __START_AT / __END_AT)
dp.create_streaming_table("customers_history")
dp.create_auto_cdc_flow(
    name="customers_history_cdc",
    target="customers_history",
    source="customers_cdc_clean",
    keys=["id"],
    sequence_by=col("operation_date"),
    apply_as_deletes=expr("operation = 'DELETE'"),
    except_column_list=["operation", "operation_date"],
    stored_as_scd_type="2",
)
```

### Same gold step in SQL

```sql
CREATE OR REFRESH STREAMING TABLE customers;

CREATE FLOW customers_cdc AS AUTO CDC INTO customers
FROM STREAM(customers_cdc_clean)
KEYS (id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY operation_date
COLUMNS * EXCEPT (operation, operation_date)
STORED AS SCD TYPE 1;
```

## Quick Reference

| Step | Where |
|------|-------|
| Run one file / one table | Editor: Run file, or refresh a single table |
| Validate without writing | Dry run (`databricks bundle run --validate-only <pipeline>`) |
| Schedule | Editor Schedule button (creates a job) or a pipeline task in a Lakeflow Job |
| Run continuously in production | Continuous job wrapping a triggered pipeline |
| Put under Git and CI/CD | Git folder + Declarative Automation Bundles |

## Common Mistakes

### Wrong

```python
# Auto Loader schema/checkpoint paths on DBFS mounts inside a pipeline
.option("cloudFiles.schemaLocation", "/mnt/schema/customers")
```

### Correct

```python
# Inside a pipeline the schema location is managed for you; read from a UC volume
spark.readStream.format("cloudFiles").option("cloudFiles.format", "json") \
    .load("/Volumes/main/landing/files/customers/")
```

## Related

- [Core concepts](core-concepts.md)
- [CDC fundamentals](cdc-fundamentals.md)
- [Deployment model](deployment-model.md)
- [Lakeflow Jobs reference](../reference/lakeflow-jobs.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/tutorial-pipelines
- https://docs.databricks.com/aws/en/ldp/multi-file-editor
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/developer/python-dev
- https://docs.databricks.com/aws/en/ldp/monitor-event-logs
- https://docs.databricks.com/aws/en/jobs/tasks/pipeline
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
