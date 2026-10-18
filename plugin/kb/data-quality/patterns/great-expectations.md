# Great Expectations (GX Core 1.x)

> **Purpose**: Exact GX Core 1.x API for validating Spark and pandas DataFrames — data source → asset → batch definition → suite → validation definition → checkpoint, with runtime `batch_parameters`
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Validating a DataFrame inside a Python job (Spark or pandas) before it is written
- Reusable expectation suites with per-expectation `severity` and HTML Data Docs
- SQL-shaped rules on a Spark batch (`UnexpectedRowsExpectation`)
- Teams already on GX; for new Databricks pipelines compare with native expectations first
  ([databricks-data-quality](../concepts/databricks-data-quality.md))

**Status (2026-10):** GX Core **1.24.0** (2026-10-07), Python 3.10–3.14, Apache 2.0. Spark 4 (and ANSI
mode) supported since **1.19.0** — the `[spark]` extra allows `pyspark>=2.3.2,<4.2`, so DBR 17.3 LTS
(Spark 4.0) needs GX ≥ 1.19. Fivetran became steward of GX Core on 2026-05-13 (repository now
`github.com/fivetran/great_expectations`). **GX Cloud was acquired by FICO and shut down on 2026-06-01**:
`gx.get_context(mode="cloud")` raises since 1.18.0, and ExpectAI (a GX Cloud feature) is gone.

## Implementation

```python
# pip install 'great_expectations[spark]>=1.19'   (on Databricks, PySpark comes from the runtime)
import great_expectations as gx
from great_expectations.checkpoint import UpdateDataDocsAction

df = spark.read.table("main.sales.orders")          # any Spark DataFrame (pandas: add_pandas)

context = gx.get_context(mode="ephemeral")          # "file" persists config under gx/; no "cloud"
data_source = context.data_sources.add_spark(name="spark_in_memory")
data_asset = data_source.add_dataframe_asset(name="orders")
batch_definition = data_asset.add_batch_definition_whole_dataframe("orders_whole_df")

suite = context.suites.add(gx.ExpectationSuite(name="orders_quality"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToBeUnique(column="order_id"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(
    column="customer_id", mostly=0.99, severity="warning"))          # severity: critical|warning|info
suite.add_expectation(gx.expectations.ExpectColumnValuesToBeInSet(
    column="status", value_set=["pending", "completed", "cancelled", "refunded"]))
suite.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(
    column="amount", min_value=0, max_value=1_000_000))
suite.add_expectation(gx.expectations.ExpectTableRowCountToBeBetween(
    min_value=1, max_value=10_000_000))
suite.add_expectation(gx.expectations.UnexpectedRowsExpectation(           # passes when 0 rows return
    unexpected_rows_query="SELECT * FROM {batch} WHERE amount < 0 AND status = 'completed'",
    description="Completed orders cannot have a negative amount"))

validation_definition = context.validation_definitions.add(
    gx.ValidationDefinition(name="orders_validation", data=batch_definition, suite=suite))

checkpoint = context.checkpoints.add(gx.Checkpoint(
    name="orders_checkpoint",
    validation_definitions=[validation_definition],
    actions=[UpdateDataDocsAction(name="update_data_docs")],
    result_format={"result_format": "SUMMARY"},
))

# The DataFrame is passed at run time — a dataframe batch definition has no data of its own
result = checkpoint.run(batch_parameters={"dataframe": df})

validation_result = result.run_results[next(iter(result.run_results))]
if not result.success and validation_result.get_max_severity_failure() == "critical":
    raise ValueError(result.describe())               # describe() returns a JSON string
```

Verified on 2026-10-17 with GX 1.24.0 + PySpark 4.0.1 (local): the run returns `success=False` for the
bad rows, `get_max_severity_failure()` returns `FailureSeverity.CRITICAL`, and
`checkpoint.run()` without `batch_parameters` raises `BuildBatchRequestError`.

## Configuration

| Component | API (1.x) | Notes |
|-----------|-----------|-------|
| Context | `gx.get_context(mode="ephemeral" \| "file")` | No `mode`: nearest File context, else Ephemeral |
| Data source | `context.data_sources.add_spark(name=)`, `add_pandas(name=)`, `add_databricks_sql(name=, connection_string=)` | Databricks SQL string: `databricks://token:<TOKEN>@<HOST>:<PORT>?http_path=<HTTP_PATH>&catalog=<CATALOG>&schema=<SCHEMA>` (can be `"${ENV_VAR}"`) — needs the `[databricks]` extra (SQLAlchemy) |
| Asset / batch | `add_dataframe_asset(name=)` → `add_batch_definition_whole_dataframe(name)` | Data arrives via `batch_parameters={"dataframe": df}` |
| Suite | `context.suites.add(gx.ExpectationSuite(name=))` → `suite.add_expectation(...)` | |
| Validation definition | `context.validation_definitions.add(gx.ValidationDefinition(name=, data=, suite=))` | `.run(batch_parameters=...)`; `get_unexpected_rows()` (1.15+) for all failing rows of `UnexpectedRowsExpectation` |
| Checkpoint | `context.checkpoints.add(gx.Checkpoint(name=, validation_definitions=, actions=, result_format=))` | `.run(batch_parameters=None, expectation_parameters=None, run_id=None)` |
| Actions | `great_expectations.checkpoint`: `UpdateDataDocsAction`, `SlackNotificationAction`, `EmailAction`, `MicrosoftTeamsNotificationAction`, `PagerdutyAlertAction`, `OpsgenieAlertAction`, `SNSNotificationAction` | `notify_on`: `all`, `success`, `failure`, `critical`, `warning`, `info` |
| Common params | `mostly` (0–1), `severity` (1.5.9+, default `critical`), `row_condition` | `row_condition=(Column("status") == "completed")` with `from great_expectations.expectations.row_conditions import Column` (1.9+) |
| Extras | `[spark]`, `[spark-connect]`, `[databricks]` | `[databricks]` is for Databricks SQL, not Spark |

## Example Usage

```python
# Interactive check (results are not persisted)
batch = batch_definition.get_batch(batch_parameters={"dataframe": df})
print(batch.validate(gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id")).success)

# Data Docs for a File context
context.build_data_docs()
```

| Don't | Do |
|-------|----|
| `great_expectations suite new` / `checkpoint run` / YAML checkpoints | Removed in 1.0: everything is Python ("all configurations except credentials are now in Python"); no console script is installed |
| `checkpoint.run()` on a dataframe asset | `checkpoint.run(batch_parameters={"dataframe": df})` |
| String `row_condition` + `condition_parser` | Deprecated since 1.9, removed in 2.0 — use `Column(...)` objects |
| `mode="cloud"` / ExpectAI | GX Cloud is shut down; use GX Core locally |

## See Also

- [pyspark-validation-libs](pyspark-validation-libs.md) — GX vs pandera vs Soda vs PyDeequ on Databricks runtimes
- [databricks-native-dq](databricks-native-dq.md)
- [dbt-testing](dbt-testing.md)

## Sources

- https://docs.greatexpectations.io/docs/core/changelog/
- https://pypi.org/project/great-expectations/
- https://docs.greatexpectations.io/docs/core/connect_to_data/dataframes/
- https://docs.greatexpectations.io/docs/core/connect_to_data/sql_data/
- https://docs.greatexpectations.io/docs/core/run_validations/create_a_validation_definition
- https://docs.greatexpectations.io/docs/core/trigger_actions_based_on_results/create_a_checkpoint_with_actions
- https://docs.greatexpectations.io/docs/core/customize_expectations/use_sql_to_define_a_custom_expectation
- https://docs.greatexpectations.io/docs/core/customize_expectations/row_conditions/
- https://greatexpectations.io/blog/changes-to-know-for-gx-core-1-0/
- https://greatexpectations.io/blog/an-update-from-great-expectations/
- https://www.fivetran.com/press/fivetran-to-become-steward-of-the-great-expectations-open-source-community-and-gx-core-project
