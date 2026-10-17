# Feature Engineering in Unity Catalog (Databricks)

> **Purpose**: Databricks Feature Store in Unity Catalog — feature tables with `FeatureEngineeringClient`, time-series point-in-time lookups, training sets, model logging with feature lineage, and batch scoring
> **MCP Validated**: 2026-10-17

## When to Use

- Features live in a Databricks lakehouse and models are registered in Unity Catalog
- You need point-in-time correct training data and the same features at inference (`score_batch`)
- You compute features yourself (Lakeflow / Spark pipelines) — the GA authoring path. For declarative, platform-managed
  pipelines Databricks recommends **Feature Views** for most new cases; both publish to the Online Feature Store (Lakebase)
- Outside Databricks, see the Feast-based [feature-engineering](feature-engineering.md) pattern

## Implementation

```python
# %pip install databricks-feature-engineering   (0.18.2 on PyPI as of 2026-10-17; Feature Views need >= 0.16.0)
from databricks.feature_engineering import FeatureEngineeringClient, FeatureLookup
import mlflow
from sklearn.ensemble import RandomForestClassifier

fe = FeatureEngineeringClient()

# --- 1. Feature tables: a UC Delta table with a primary key -------------------------------
customer_features = compute_customer_features(spark.table("main.sales.orders"))   # your Spark logic

fe.create_table(
    name="main.ml.customer_features",
    primary_keys=["customer_id", "as_of_ts"],
    timeseries_columns="as_of_ts",          # enables point-in-time (as-of) joins
    df=customer_features,
    description="Customer purchase aggregates, daily snapshot",
)

# Incremental refresh: upsert by primary key
fe.write_table(name="main.ml.customer_features", df=todays_features, mode="merge")

# --- 2. Training set with point-in-time lookups ---------------------------------------------
labels = spark.table("main.ml.churn_labels")        # customer_id, label_ts, churned
training_set = fe.create_training_set(
    df=labels,
    feature_lookups=[
        FeatureLookup(
            table_name="main.ml.customer_features",
            feature_names=["orders_30d", "avg_order_value_90d", "days_since_last_order"],
            lookup_key="customer_id",
            timestamp_lookup_key="label_ts",        # feature values as of each label time
        ),
    ],
    label="churned",
    exclude_columns=["customer_id", "label_ts"],
)
train_pdf = training_set.load_df().toPandas()

# --- 3. Train and log the model WITH its feature metadata -----------------------------------
with mlflow.start_run():
    X, y = train_pdf.drop(columns=["churned"]), train_pdf["churned"]
    model = RandomForestClassifier(n_estimators=200).fit(X, y)
    fe.log_model(
        model=model,
        artifact_path="churn_model",
        flavor=mlflow.sklearn,
        training_set=training_set,                 # stores lookups for inference-time retrieval
        registered_model_name="main.ml.churn_model",
    )

# --- 4. Batch inference: features are looked up automatically -------------------------------
scoring_df = spark.table("main.ml.customers_to_score")    # customer_id, label_ts
predictions = fe.score_batch(model_uri="models:/main.ml.churn_model/1", df=scoring_df)
```

## Configuration

| Item | Value | Notes |
|------|-------|-------|
| Package | `databricks-feature-engineering` | Workspace Feature Store (non-UC) is deprecated |
| Primary key | Required on every feature table | Composite keys allowed |
| `timeseries_columns` | Timestamp column in the primary key | Without it, joins match only exact timestamps |
| `timestamp_lookup_key` | Timestamp column in the label DataFrame | Required on lookups to time-series tables |
| `write_table(mode="merge")` | Upsert | Rows absent from the DataFrame stay unchanged |
| Online serving | Online Feature Store (Lakebase) | Publish feature tables or Feature Views for real-time lookups |

## Example Usage

```python
# score_batch returns the input columns + looked-up feature values + a "prediction" column
predictions.select("customer_id", "orders_30d", "prediction").show(5)
```

## See Also

- [Feature Stores Concept](../concepts/feature-stores.md)
- [Feature Engineering (Feast)](feature-engineering.md)
- [Training Data Pipelines](training-data-pipelines.md)

## Sources

- https://docs.databricks.com/aws/en/machine-learning/feature-store/
- https://docs.databricks.com/aws/en/machine-learning/feature-store/uc/feature-tables-uc
- https://docs.databricks.com/aws/en/machine-learning/feature-store/time-series
- https://docs.databricks.com/aws/en/machine-learning/feature-store/train-models-with-feature-store
- https://docs.databricks.com/aws/en/machine-learning/feature-store/feature-views
- https://pypi.org/project/databricks-feature-engineering/
