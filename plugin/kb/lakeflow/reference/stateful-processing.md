# Stateful Processing in Lakeflow Pipelines

> **Purpose**: Watermarks, stream-stream joins, windowed aggregations, deduplication and state store settings in pipelines
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Watermarks

A watermark is a time threshold for stateful operations (aggregations, joins, deduplication). Data
older than the threshold may be dropped and the corresponding state is released. Without a
watermark, streaming aggregations are not processed incrementally and state can grow without bound.

| Interface | Syntax |
|-----------|--------|
| Python | `df.withWatermark("event_ts", "10 minutes")` |
| SQL | `FROM STREAM(t) WATERMARK event_ts DELAY OF INTERVAL 10 MINUTES` |

Trade-off: a short threshold emits results sooner but drops more late data; a long threshold keeps
more late data but holds more state and delays output. Test and monitor to pick it.

## 2. Windowed aggregations

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import expr, window

@dp.table
def profit_by_hour():
    return (spark.readStream.table("sales")
            .withWatermark("ts", "1 hour")
            .groupBy(window("ts", "1 hour").alias("time"))       # tumbling
            .agg(expr("sum(profit) AS profit")))

@dp.table
def clicks_sliding():
    return (spark.readStream.table("clicks")
            .withWatermark("click_ts", "10 minutes")
            .groupBy(window("click_ts", "1 hour", "30 minutes"))  # sliding
            .count())
```

```sql
CREATE OR REFRESH STREAMING TABLE ad_seconds_5m AS
SELECT ad_id, window(click_ts, '5 minutes') AS w, SUM(seconds) AS total_seconds
FROM STREAM(ad_clicks)
WATERMARK click_ts DELAY OF INTERVAL 3 MINUTES
GROUP BY ad_id, window(click_ts, '5 minutes');
```

The window must use the same timestamp column as the watermark. Results for a window are emitted
when data passes the window end plus the watermark delay; then its state is dropped.

## 3. Stream-stream joins

Both sides need a watermark and the join needs a time-interval condition on the watermarked columns.
The engine keeps one global watermark based on the slowest stream.

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import expr

dp.create_streaming_table("ad_impression_clicks")

@dp.append_flow(target="ad_impression_clicks")
def join_clicks_and_impressions():
    clicks = spark.readStream.table("raw_clicks").withWatermark("click_ts", "3 minutes")
    imps = spark.readStream.table("raw_impressions").withWatermark("imp_ts", "3 minutes")
    return (imps.alias("i").join(
                clicks.alias("c"),
                expr("""i.user_id = c.user_id AND c.ad_id = i.ad_id AND
                        c.click_ts >= i.imp_ts AND c.click_ts <= i.imp_ts + interval 3 minutes"""),
                "inner")
            .select("i.user_id", "i.ad_id", "c.click_ts"))
```

## 4. Deduplication

`dropDuplicatesWithinWatermark()` removes duplicates on chosen keys even when other fields differ;
it requires a watermark. To process an initial snapshot in event-time order, set the
`withEventTimeOrder` **read option** (Python only) or the pipeline Spark conf
`spark.databricks.delta.withEventTimeOrder.enabled = true`.

```python
from pyspark import pipelines as dp

@dp.table
def clicks_dedup():
    return (spark.readStream
            .option("withEventTimeOrder", "true")      # read option, not a function argument
            .table("raw_clicks")
            .withWatermark("click_ts", "5 seconds")
            .dropDuplicatesWithinWatermark(["user_id", "ad_id"]))
```

`dropDuplicates()` on a stream without a watermark keeps every key in state forever.

## 5. State store

- Classic compute: Databricks recommends the RocksDB state store for large state:

```json
{
  "configuration": {
    "spark.sql.streaming.stateStore.providerClass": "com.databricks.sql.streaming.state.RocksDBStateStoreProvider"
  }
}
```

- Serverless pipelines manage state store configuration automatically.
- Pipelines manage schema and checkpoint locations automatically, per flow; a full refresh resets
  the checkpoints.
- Millisecond-latency stateful workloads: see real-time mode for Lakeflow pipelines.
- Custom arbitrary stateful operators are outside the pipelines stateful-processing guide; check
  the Structured Streaming documentation for the Spark version your channel runs before using them.

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Windowed table never emits | Watermark not advancing (idle source, null or future timestamps) | Filter nulls, cap future timestamps, check every input stream |
| State keeps growing | Missing watermark, very long delay, high-cardinality keys | Add/shorten watermark, RocksDB on classic |
| Late events missing | Threshold too short | Increase delay, monitor late-data rate |
| Aggregation recomputed every update | Streaming aggregation without watermark | Add `withWatermark` / `WATERMARK` |

```python
from pyspark.sql import functions as F

clean = (spark.readStream.table("events")
         .where(F.col("event_ts").isNotNull())
         .withColumn("event_ts", F.least("event_ts", F.current_timestamp())))
```

## Sources

- https://docs.databricks.com/aws/en/ldp/stateful-processing
- https://docs.databricks.com/aws/en/ldp/serverless
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-append-flow
- https://docs.databricks.com/aws/en/ldp/for-each-batch
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema
- https://learn.microsoft.com/en-us/azure/databricks/ldp/stateful-processing
