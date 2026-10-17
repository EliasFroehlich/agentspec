# PySpark Validation Libraries on Databricks

> **Purpose**: Validate Spark DataFrames inside a Databricks job with GX Core, pandera, Soda Core (`soda-sparkdf`) or PyDeequ — runnable snippets plus a compatibility matrix per Databricks Runtime LTS
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- A plain job or notebook (not a Lakeflow pipeline) must validate a DataFrame before writing it
- Rules need aggregates or other tables, which SDP expectations cannot express
- Hand-rolled `validate_*` helpers should be replaced by a maintained library
- First check native options: [databricks-data-quality](../concepts/databricks-data-quality.md)

### Compatibility (verified 2026-10-17)

| Library (latest) | DBR 16.4 LTS (Spark 3.5) | DBR 17.3 LTS (Spark 4.0) | DBR 18 LTS (Spark 4.1) | Notes |
|------------------|--------------------------|--------------------------|------------------------|-------|
| GX Core 1.24.0 | Yes | Yes (GX ≥ 1.19) | Yes | `[spark]` extra: `pyspark<4.2`; Spark Connect DataFrames supported |
| pandera 0.34.1 | Yes | Yes (pandera ≥ 0.28) | Yes | `pandera[pyspark]`; Spark Connect DataFrames since 0.20.4 |
| Soda Core 4.27.0 (`soda-sparkdf`) | Yes | Yes | Not stated | Python ≥ 3.10; Databricks Connect 16.4.9+ / 17.3.1+; upgrade `pydantic` on the cluster |
| PyDeequ 1.7.0 | Yes (Deequ 2.0.21-spark-3.5) | **No** — PyDeequ declares `pyspark!=4.0.*`; only the Scala library exists for Spark 4.0 (`com.amazon.deequ:deequ:2.0.15-spark-4.0`) | Yes (Deequ 2.0.18-spark-4.1) | Py4J/JVM bridge → Spark Classic (dedicated) compute only; a Spark 3.5 JAR on a Spark 4.0 cluster does not work; PyDeequ 2.0 (Spark Connect) is beta, GitHub-only |
| DQX 0.16.0 (`databricks-labs-dqx`) | — | — | — | Databricks Labs project: "not formally supported by Databricks"; checks as code or YAML, quarantine, ODCS-derived rules |

## Implementation

### GX Core on a Spark DataFrame

```python
import great_expectations as gx

context = gx.get_context(mode="ephemeral")
batch_definition = (context.data_sources.add_spark(name="spark_in_memory")
                    .add_dataframe_asset(name="orders")
                    .add_batch_definition_whole_dataframe("orders_whole_df"))
suite = context.suites.add(gx.ExpectationSuite(name="orders_quality"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(column="amount", min_value=0))
vd = context.validation_definitions.add(
    gx.ValidationDefinition(name="orders_validation", data=batch_definition, suite=suite))
result = vd.run(batch_parameters={"dataframe": df})
if not result.success:
    raise ValueError(result.describe())
```

Full GX reference (checkpoints, actions, severity, `UnexpectedRowsExpectation`): [great-expectations](great-expectations.md).

### pandera (`pandera.pyspark`)

```python
import pandera.pyspark as pa
import pyspark.sql.types as T
from pandera.pyspark import DataFrameModel

class Orders(DataFrameModel):
    order_id: T.StringType() = pa.Field(nullable=False)
    customer_id: T.StringType() = pa.Field(nullable=False)
    status: T.StringType() = pa.Field(isin=["pending", "completed", "cancelled", "refunded"])
    amount: T.DoubleType() = pa.Field(ge=0)

    class Config:
        strict = True                    # reject unexpected columns

df_out = Orders.validate(check_obj=df)   # lazy by default: does NOT raise
errors = df_out.pandera.errors           # read immediately; later Spark ops may reset it
if errors:                               # {"SCHEMA": {...}, "DATA": {...}}
    raise ValueError(dict(errors))
```

Env vars: `PANDERA_VALIDATION_DEPTH=SCHEMA_ONLY|DATA_ONLY|SCHEMA_AND_DATA` (default), `PANDERA_VALIDATION_ENABLED=False`,
`PANDERA_CACHE_DATAFRAME` — leave caching off on serverless compute, which does not support `cache()`/`persist()`.
The opt-in narwhals backend (`PANDERA_USE_NARWHALS_BACKEND=True`, 0.32+) changes PySpark behavior: it raises
`SchemaErrors` instead of filling `df.pandera.errors`. Locally, PySpark 4.0.x needs `pandas<3` for
`pyspark.pandas`, which pandera imports.

### Soda Core v4 on a Spark session

```python
# %pip install soda-sparkdf  (then %pip install -U pydantic; dbutils.library.restartPython())
from soda_core.contracts import verify_contract_locally
from soda_sparkdf import SparkDataFrameDataSource

ds = SparkDataFrameDataSource.from_existing_session(session=spark, name="my_spark")
result = verify_contract_locally(
    data_sources=[ds],
    contract_file_path="contracts/orders.yml",   # dataset: my_spark/<catalog>/<schema>/<table>
    publish=False,                               # True + soda_cloud_file_path to send results
)
if not result.is_ok:                             # False on failed checks or execution errors
    raise RuntimeError(result.get_errors_str())
```

Contract grammar: [soda-core](../concepts/soda-core.md).

### PyDeequ (Spark 3.5 or 4.1 only)

```python
import os
os.environ["SPARK_VERSION"] = "3.5"              # required; "3.5" or "4.1", else RuntimeError
import pydeequ
from pydeequ.checks import Check, CheckLevel
from pydeequ.verification import VerificationSuite, VerificationResult
# The cluster needs the Maven library pydeequ.deequ_maven_coord
# (com.amazon.deequ:deequ:2.0.21-spark-3.5) installed on dedicated compute.

check = Check(spark, CheckLevel.Error, "orders")
result = (VerificationSuite(spark).onData(df)
          .addCheck(check.hasSize(lambda n: n >= 1)
                         .isComplete("order_id").isUnique("order_id")
                         .isContainedIn("status", ["pending", "completed", "cancelled", "refunded"])
                         .isNonNegative("amount"))
          .run())
VerificationResult.checkResultsAsDataFrame(spark, result).show()
```

## Configuration

| Concern | GX Core | pandera | Soda Core | PyDeequ |
|---------|---------|---------|-----------|---------|
| Rules defined in | Python (suite) | Python class / schema | YAML contract | Python (Scala API) |
| Failure handling | `result.success`, severity per expectation | Collects errors (lazy); raise yourself | `is_ok`, CLI exit codes | `checkResultsAsDataFrame` |
| Aggregates / uniqueness | Yes | `unique` via `Config` | Yes | Yes (analyzers) |
| Row-level failing rows | `result_format`, `get_unexpected_rows()` | Error report only | `failed_rows` check | Row-level results |
| Verified locally 2026-10-17 | 1.24.0 + PySpark 4.0.1 | 0.34.1 + PySpark 4.0.1 | 4.27.0 + PySpark 4.0.1 | 1.7.0 rejects `SPARK_VERSION=4.0` |

## See Also

- [great-expectations](great-expectations.md)
- [databricks-native-dq](databricks-native-dq.md)
- [schema-validation](schema-validation.md)

## Sources

- https://pypi.org/project/great-expectations/
- https://docs.greatexpectations.io/docs/core/changelog/
- https://docs.greatexpectations.io/docs/core/connect_to_data/dataframes/
- https://pandera.readthedocs.io/en/stable/pyspark_sql.html
- https://github.com/unionai-oss/pandera/releases
- https://docs.soda.io/reference/data-source-reference-for-soda-core/spark-dataframe
- https://pypi.org/project/soda-sparkdf/
- https://github.com/awslabs/python-deequ
- https://pypi.org/project/pydeequ/
- https://repo1.maven.org/maven2/com/amazon/deequ/deequ/maven-metadata.xml
- https://github.com/awslabs/python-deequ/releases/tag/v2.0.0b1
- https://github.com/databrickslabs/dqx
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://docs.databricks.com/aws/en/spark/connect-vs-classic
