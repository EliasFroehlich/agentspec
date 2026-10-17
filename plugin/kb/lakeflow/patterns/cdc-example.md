# CDC Complete Example

> **Purpose**: End-to-end CDC pipeline (bronze feed, validated silver feed, SCD Type 1 and Type 2 gold) in Python and SQL
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Replicating an operational table into the lakehouse from a CDC feed landed as files
- Serving both "current state" and "full history" tables from one feed
- Handling out-of-order events and deletes without hand-written MERGE logic

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import col, expr

LANDING = spark.conf.get("landing_path")   # e.g. /Volumes/main/landing/files/users

# Bronze: raw CDC events (JSON files)
@dp.table(comment="Raw user CDC events")
def users_cdc_bronze():
    return (spark.readStream.format("cloudFiles")
            .option("cloudFiles.format", "json")
            .option("cloudFiles.inferColumnTypes", "true")
            .load(LANDING)
            .select("*", col("_metadata.file_path").alias("_source_file")))

# Silver: validated event stream
@dp.table(comment="Validated user CDC events")
@dp.expect_or_drop("valid_operation", "operation IN ('INSERT', 'UPDATE', 'DELETE')")
@dp.expect_or_drop("valid_sequence", "sequence_num IS NOT NULL")
@dp.expect_or_drop("valid_key", "user_id IS NOT NULL")
def users_cdc_silver():
    return spark.readStream.table("users_cdc_bronze")

# Gold: current state (SCD Type 1)
dp.create_streaming_table("users_current", cluster_by=["user_id"])
dp.create_auto_cdc_flow(
    name="users_current_cdc",
    target="users_current",
    source="users_cdc_silver",
    keys=["user_id"],
    sequence_by=col("sequence_num"),
    apply_as_deletes=expr("operation = 'DELETE'"),
    except_column_list=["operation", "sequence_num", "_source_file"],
    stored_as_scd_type=1,
)

# Gold: history (SCD Type 2)
dp.create_streaming_table("users_history")
dp.create_auto_cdc_flow(
    name="users_history_cdc",
    target="users_history",
    source="users_cdc_silver",
    keys=["user_id"],
    sequence_by=col("sequence_num"),
    apply_as_deletes=expr("operation = 'DELETE'"),
    except_column_list=["operation", "sequence_num", "_source_file"],
    stored_as_scd_type="2",
)
```

```sql
-- Same pipeline in SQL
CREATE OR REFRESH STREAMING TABLE users_cdc_bronze AS
SELECT *, _metadata.file_path AS _source_file
FROM STREAM read_files("${landing_path}", format => "json", inferColumnTypes => true);

CREATE OR REFRESH STREAMING TABLE users_cdc_silver (
  CONSTRAINT valid_operation EXPECT (operation IN ('INSERT','UPDATE','DELETE')) ON VIOLATION DROP ROW,
  CONSTRAINT valid_sequence  EXPECT (sequence_num IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT valid_key       EXPECT (user_id IS NOT NULL)      ON VIOLATION DROP ROW
) AS SELECT * FROM STREAM(users_cdc_bronze);

CREATE OR REFRESH STREAMING TABLE users_current;
CREATE FLOW users_current_cdc AS AUTO CDC INTO users_current
FROM STREAM(users_cdc_silver)
KEYS (user_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY sequence_num
COLUMNS * EXCEPT (operation, sequence_num, _source_file)
STORED AS SCD TYPE 1;

CREATE OR REFRESH STREAMING TABLE users_history;
CREATE FLOW users_history_cdc AS AUTO CDC INTO users_history
FROM STREAM(users_cdc_silver)
KEYS (user_id)
APPLY AS DELETE WHEN operation = 'DELETE'
SEQUENCE BY sequence_num
COLUMNS * EXCEPT (operation, sequence_num, _source_file)
STORED AS SCD TYPE 2;
```

## Configuration

| Pipeline setting | Value | Why |
|------------------|-------|-----|
| `configuration.landing_path` | `/Volumes/<catalog>/<schema>/<volume>/users` | Read by `spark.conf.get` / `${landing_path}` |
| `serverless` | `true` | Serverless includes the edition needed for CDC and expectations |
| Pipeline mode | triggered | Schedule with a Lakeflow Job |

## Example Usage

Events arriving in order 3, 1, 2 for the same key are applied by `sequence_num`; a late lower
sequence never overwrites a higher one. Expected outcome after the official tutorial data set:
SCD Type 1 holds one row per surviving key; SCD Type 2 holds one row per version, with the current
row having `__END_AT IS NULL`.

Rules to remember:

- `NULL` in the sequencing column is not supported.
- Keep one distinct update per key per sequence value; otherwise tie-break with `struct(...)`.
- Identity columns are not supported on AUTO CDC targets.
- Auto Loader does not guarantee file order; raise `pipelines.cdc.tombstoneGCThresholdInSeconds`
  above the maximum expected arrival delay when deletes may arrive late.

## See Also

- [CDC with AUTO CDC](cdc-apply-changes.md)
- [CDC fundamentals](../concepts/cdc-fundamentals.md)
- [Expectations](expectations.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/cdc
- https://docs.databricks.com/aws/en/ldp/tutorial-pipelines
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-apply-changes
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-streaming-table
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-apply-changes-into
- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/ldp/serverless
