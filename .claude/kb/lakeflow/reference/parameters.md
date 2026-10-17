# Pipeline Parameters

> **Purpose**: Parameterize pipeline source code with pipeline parameters (SQL named parameters, Beta) and the `configuration` map (Python `spark.conf.get`, SQL `${key}`), including job pushdown and precedence
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Two mechanisms

| | Pipeline parameters (`parameters`, Beta) | Configuration (`configuration`) |
|---|---|---|
| Reference | SQL only: `:key` (wrap in `IDENTIFIER()` in identifier positions) | Python `spark.conf.get("key")`; SQL `${key}` |
| Override per update | Yes: Run with different settings, Start update API, pipeline task, job parameters | No (edit settings) |
| Pushed down from jobs | Yes, all job parameters | No |
| Also used for | - | Spark conf (`pipelines.*`, `spark.*`) |
| Missing value | Update fails | `spark.conf.get` raises unless a default is given |

Keys: alphanumeric, `_`, `-`, `.`. Values are always strings. Avoid keys that collide with reserved
pipeline or Spark configuration names.

## 2. Pipeline parameters (SQL)

```yaml
# bundle resource
resources:
  pipelines:
    sales_pipeline:
      name: sales_pipeline
      catalog: ${var.catalog}
      schema: sales
      parameters:
        source_catalog: dev_raw
        start_date: "2026-01-01"
```

```sql
CREATE OR REFRESH MATERIALIZED VIEW transaction_summary AS
SELECT account_id, COUNT(txn_id) AS txn_count, SUM(txn_amount) AS revenue
FROM :source_catalog.sales.transactions
WHERE txn_date >= :start_date
GROUP BY account_id;

USE CATALOG IDENTIFIER(:source_catalog);
```

Precedence (highest first): job run parameters > job parameters > pipeline task parameters >
pipeline parameters. In a job, values can use dynamic value references such as
`{{job.trigger.time.iso_date}}` or `{{tasks.<task>.values.<name>}}`.

```yaml
tasks:
  - task_key: refresh
    pipeline_task:
      pipeline_id: ${resources.pipelines.sales_pipeline.id}
      parameters:
        start_date: "{{job.parameters.start_date}}"
```

Caveats:

- A pipeline runs one update at a time; a job with `max_concurrent_runs > 1` or a pipeline task in a
  For each is capped to concurrency 1.
- Filtering a materialized view on both ends of a date range (`>= :start AND < :end`) forces a full
  refresh on every update; a one-sided filter stays incremental.

## 3. Configuration map (Python and SQL)

```yaml
resources:
  pipelines:
    sales_pipeline:
      configuration:
        mypipeline.landing_path: /Volumes/main/landing/files
        mypipeline.start_date: "2026-01-01"
        mypipeline.enable_pii_masking: "true"
```

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

LANDING = spark.conf.get("mypipeline.landing_path")
START = spark.conf.get("mypipeline.start_date", "2020-01-01")       # default avoids failures
MASK_PII = spark.conf.get("mypipeline.enable_pii_masking", "false").lower() == "true"

@dp.table
def orders_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .load(f"{LANDING}/orders/"))

@dp.materialized_view
def customers():
    df = spark.read.table("main.crm.customers").where(F.col("updated_at") >= START)
    return df.withColumn("email", F.sha2("email", 256)) if MASK_PII else df
```

```sql
CREATE OR REFRESH MATERIALIZED VIEW customer_events AS
SELECT * FROM source_table WHERE date > '${mypipeline.start_date}';
```

## 4. Conditional expectations from a flag

Decorators must wrap a function; build the dataset in a helper to switch rules on or off:

```python
ENABLE_QC = spark.conf.get("mypipeline.enable_quality_checks", "true") == "true"

def orders_silver_df():
    return spark.readStream.table("orders_bronze")

if ENABLE_QC:
    @dp.table(name="orders_silver")
    @dp.expect_or_drop("valid_order", "order_id IS NOT NULL")
    def orders_silver():
        return orders_silver_df()
else:
    @dp.table(name="orders_silver")
    def orders_silver():
        return orders_silver_df()
```

## 5. Practices

| Do | Don't |
|----|-------|
| Prefix keys (`mypipeline.*`) | Reuse `spark.*` or `pipelines.*` names for your own values |
| Give `spark.conf.get` a default | Assume a key exists |
| Convert types in code (`int(...)`, `== "true"`) | Expect non-string values |
| Reference secrets as `{{secrets/<scope>/<key>}}` in Spark conf or with SQL `secret('<scope>', '<key>')` | Put secret values in plain `configuration`/`parameters` |
| Set per-environment values through bundle targets/variables | Hard-code catalogs or paths |
| Use pipeline parameters for values that change per run | Use bundle variables for per-run values (they resolve at deploy time) |

## Sources

- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/load
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/jobs/tasks/pipeline
- https://docs.databricks.com/aws/en/jobs/dynamic-value-references
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/dev-tools/bundles/variables
- https://github.com/databricks/databricks-sdk-py/blob/main/databricks/sdk/service/jobs.py
