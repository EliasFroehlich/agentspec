# Databricks Implementation Patterns

> **Purpose**: Production patterns for Lakeflow pipelines with expectations and AUTO CDC, Lakeflow Jobs on serverless (and classic compute with access modes), Unity Catalog setup with volumes and liquid clustering, and MLflow 3 on Unity Catalog
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Medallion pipelines with declarative data quality gates and SCD handling
- Multi-task orchestration where most tasks can run on serverless compute
- Bootstrapping catalogs, schemas, volumes and grants for a new domain
- Registering models with governance in Unity Catalog

## Implementation

### Lakeflow pipeline: expectations + AUTO CDC (Python)

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

@dp.table(comment="Raw customer CDC feed", table_properties={"quality": "bronze"})
def customers_cdc_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.inferColumnTypes", "true")
            .load("/Volumes/main/raw/landing/customers/"))

@dp.table(comment="Validated CDC events", cluster_by_auto=True)
@dp.expect_or_drop("valid_email", "email RLIKE '^[^@]+@[^@]+\\.[a-z]{2,}$'")
@dp.expect_or_fail("not_null_id", "customer_id IS NOT NULL")
@dp.expect("has_name", "first_name IS NOT NULL")
def customers_cdc_clean():
    return (spark.readStream.table("customers_cdc_bronze")
            .withColumn("processed_at", F.current_timestamp()))

# SCD Type 2 target maintained by an AUTO CDC flow (replaces apply_changes)
dp.create_streaming_table("customers_scd2", comment="Customer history")
dp.create_auto_cdc_flow(
    target="customers_scd2",
    source="customers_cdc_clean",
    keys=["customer_id"],
    sequence_by=F.col("event_ts"),
    apply_as_deletes=F.expr("op = 'DELETE'"),
    except_column_list=["op"],
    stored_as_scd_type="2",            # current row: __END_AT IS NULL
)

@dp.materialized_view(comment="Active customers per country")
def customers_by_country():
    return (spark.read.table("customers_scd2")
            .where(F.col("__END_AT").isNull())
            .groupBy("country").count())
```

### Lakeflow Jobs: serverless first, classic only where needed

For production, define jobs and pipelines in **Declarative Automation Bundles** YAML (same shape as the Jobs API);
see [lakeflow-jobs](../../lakeflow/reference/lakeflow-jobs.md) and [bundles-cli](../../lakeflow/reference/bundles-cli.md).

```python
# Jobs API create payload. Tasks without a cluster spec run on serverless compute.
job_config = {
    "name": "daily_orders",
    "performance_target": "STANDARD",          # or PERFORMANCE_OPTIMIZED (serverless tasks only)
    "environments": [{
        "environment_key": "default",
        "spec": {"environment_version": "6", "dependencies": ["requests==2.32.3"]},
    }],
    "tasks": [
        {"task_key": "ingest",
         "pipeline_task": {"pipeline_id": "<pipeline-id>"}},
        {"task_key": "transform", "depends_on": [{"task_key": "ingest"}],
         "spark_python_task": {"python_file": "/Workspace/Shared/etl/transform.py",
                               "parameters": ["--run-date", "{{job.start_time.iso_date}}"]},
         "environment_key": "default"},     # required for Python tasks on serverless
        {"task_key": "quality_checks", "depends_on": [{"task_key": "transform"}],
         "notebook_task": {"notebook_path": "/Workspace/Shared/etl/quality_checks"}},
        {"task_key": "legacy_jar", "depends_on": [{"task_key": "quality_checks"}],
         "spark_jar_task": {"main_class_name": "com.example.Export"},
         "libraries": [{"jar": "/Volumes/main/libs/jars/export.jar"}],
         "job_cluster_key": "classic"},
    ],
    "job_clusters": [{
        "job_cluster_key": "classic",
        "new_cluster": {
            "spark_version": "17.3.x-scala2.13",          # DBR 17.3 LTS (Spark 4.0); 18 LTS also available
            "data_security_mode": "DATA_SECURITY_MODE_DEDICATED",  # or ..._STANDARD; SINGLE_USER/USER_ISOLATION are legacy aliases
            "node_type_id": "i3.xlarge",
            "autoscale": {"min_workers": 1, "max_workers": 4},
        },
    }],
    "schedule": {"quartz_cron_expression": "0 0 6 * * ?", "timezone_id": "UTC"},
}
```

### Unity Catalog bootstrap

```sql
CREATE CATALOG IF NOT EXISTS main COMMENT 'Production catalog';
CREATE SCHEMA IF NOT EXISTS main.raw;
CREATE SCHEMA IF NOT EXISTS main.curated;
CREATE SCHEMA IF NOT EXISTS main.analytics;

-- Landing files live in a governed volume, not a DBFS mount
CREATE EXTERNAL LOCATION IF NOT EXISTS landing_s3
  URL 's3://my-bucket/landing/'
  WITH (STORAGE CREDENTIAL my_storage_credential);
CREATE EXTERNAL VOLUME IF NOT EXISTS main.raw.landing
  LOCATION 's3://my-bucket/landing/files/';   -- inside the location, not overlapping tables

-- New tables: liquid clustering, let predictive optimization pick keys
CREATE TABLE IF NOT EXISTS main.curated.orders (
  order_id STRING, customer_id STRING, order_ts TIMESTAMP, amount DECIMAL(12,2)
) CLUSTER BY AUTO;

GRANT USE CATALOG ON CATALOG main TO `data-engineers`;
GRANT USE SCHEMA, SELECT ON SCHEMA main.raw TO `data-engineers`;
GRANT READ VOLUME, WRITE VOLUME ON VOLUME main.raw.landing TO `data-engineers`;
GRANT USE SCHEMA, SELECT ON SCHEMA main.analytics TO `analysts`;
```

### MLflow 3 with the Unity Catalog registry

```python
import mlflow
from mlflow import MlflowClient
from sklearn.ensemble import RandomForestClassifier

# MLflow 3 defaults to the "databricks-uc" registry on Databricks.
with mlflow.start_run():
    clf = RandomForestClassifier(max_depth=7).fit(X_train, y_train)
    mlflow.sklearn.log_model(
        sk_model=clf,
        name="model",                                   # MLflow 3 (artifact_path in 2.x)
        input_example=X_train.iloc[[0]],                # signature inferred
        registered_model_name="main.ml.churn_model",    # three-level UC name
    )

MlflowClient().set_registered_model_alias("main.ml.churn_model", "champion", version=1)
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Pipeline compute | serverless (new pipelines) | Requires UC; classic pipelines still supported |
| Job task compute | serverless for supported task types | Notebook, Python script, wheel, dbt, JAR |
| `performance_target` | `STANDARD` start-up of 4–6 min | `PERFORMANCE_OPTIMIZED` starts faster, same SKU, more DBUs |
| `data_security_mode` | set explicitly (`..._AUTO` lets Databricks choose) | `..._DEDICATED` (single user/group) or `..._STANDARD` (shared, Spark Connect) |
| Serverless `environment_version` | `6` is current (2026-09-03) | Pins the client API (Spark Connect, Python 3.12) for serverless Python |

## Example Usage

### Wrong

```python
# Not serverless: legacy High-Concurrency profile, rejected together with access modes
new_cluster = {"spark_conf": {"spark.databricks.cluster.profile": "serverless"}}
# Legacy module and deprecated mounts
import dlt
@dlt.table
def t(): return spark.readStream.format("cloudFiles").load("/mnt/landing/")
```

### Correct

```python
# Serverless = omit new_cluster/job_cluster_key/existing_cluster_id on supported tasks
{"task_key": "t", "notebook_task": {"notebook_path": "/Workspace/Shared/etl/t"}}
```

## See Also

- [Databricks Lakeflow](../concepts/databricks-lakeflow.md) — platform map and current versions
- [Cost Optimization](cost-optimization.md) — DBU levers and billing system tables
- [Lakeflow KB](../../lakeflow/index.md) — pipelines in depth (authoritative for `pyspark.pipelines` and AUTO CDC)
- [Lakeflow Jobs reference](../../lakeflow/reference/lakeflow-jobs.md) and [Bundles CLI](../../lakeflow/reference/bundles-cli.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes
- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/jobs/run-serverless-jobs
- https://docs.databricks.com/api/workspace/jobs/create
- https://docs.databricks.com/aws/en/jobs/dynamic-value-references
- https://docs.databricks.com/aws/en/release-notes/serverless/environment-version/
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://docs.databricks.com/aws/en/dbfs/unity-catalog
- https://docs.databricks.com/aws/en/delta/clustering
- https://docs.databricks.com/aws/en/machine-learning/manage-model-lifecycle/
- https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/cluster
- https://kb.databricks.com/clusters/cannot-create-cluster-spark-conf-sparkdatabricksclusterprofile-is-not-allowed-when-choosing-an-access-mode
