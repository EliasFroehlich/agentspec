# transformWithState (Arbitrary Stateful Processing)

> **Purpose**: Correct PySpark `StatefulProcessor` + `transformWithStateInPandas` pattern (Spark 4.0+, Databricks DBR 16.2+) replacing `applyInPandasWithState` / `mapGroupsWithState`, with timers and TTL
> **Confidence**: 0.88
> **MCP Validated**: 2026-10-17

## When to Use

- Per-key custom state that built-in aggregations, `dropDuplicates` or windows cannot express
- Session tracking, downtime/heartbeat detection, per-entity state machines
- You need timers (processing or event time) or TTL-based state eviction

## Implementation

```python
from typing import Iterator
import pandas as pd
from pyspark.sql.streaming import StatefulProcessor, StatefulProcessorHandle
from pyspark.sql.types import StructType, StructField, StringType, LongType

STATE_SCHEMA = StructType([StructField("event_count", LongType(), True)])
OUTPUT_SCHEMA = StructType([
    StructField("user_id", StringType(), True),
    StructField("event_count", LongType(), True),
])

class SessionCounter(StatefulProcessor):
    """Counts events per user; emits and clears state after 30 min of inactivity."""

    def init(self, handle: StatefulProcessorHandle) -> None:
        self.handle = handle
        # getValueState(name, schema, ttlDurationMs=None); get() returns a tuple or None
        self.count = handle.getValueState("count", STATE_SCHEMA)

    def handleInputRows(self, key, rows, timerValues) -> Iterator[pd.DataFrame]:
        n = sum(len(pdf) for pdf in rows)
        current = self.count.get()[0] if self.count.exists() else 0
        self.count.update((current + n,))
        # Reset the inactivity timer (timers cannot be registered in init())
        for t in self.handle.listTimers():
            self.handle.deleteTimer(t)
        self.handle.registerTimer(timerValues.getCurrentProcessingTimeInMs() + 30 * 60 * 1000)
        yield pd.DataFrame()   # nothing to emit yet

    def handleExpiredTimer(self, key, timerValues, expiredTimerInfo) -> Iterator[pd.DataFrame]:
        total = self.count.get()[0] if self.count.exists() else 0
        self.count.clear()
        yield pd.DataFrame({"user_id": key, "event_count": [total]})   # key = grouping-key tuple

    def close(self) -> None:
        pass

# State store: RocksDB is the default provider on Databricks DBR 17.3 LTS+ (nothing to set).
# Databricks DBR < 17.3: set the provider before starting the query; the RocksDB page uses
#   com.databricks.sql.streaming.state.RocksDBStateStoreProvider
# (the transformWithState page shows the OSS class name below, which is the one for OSS Spark).
spark.conf.set(
    "spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider",   # OSS Spark
)

sessions = (events_stream
    .groupBy("user_id")
    .transformWithStateInPandas(
        statefulProcessor=SessionCounter(),
        outputStructType=OUTPUT_SCHEMA,
        outputMode="Append",          # each emitted row is a finished session
        timeMode="ProcessingTime",
    ))
```

## Configuration

| Item | Value / Rule |
|------|--------------|
| Python operators | `transformWithStateInPandas` (OSS Spark 4.0+) and row-based `transformWithState` (OSS Spark 4.1+). Databricks documents both for DBR 16.2+, so both work on DBR 17.3 LTS; real-time mode supports only the row-based `transformWithState` |
| Scala/Java operator | `transformWithState` (Spark 4.0+) |
| State variables | `getValueState`, `getListState`, `getMapState` on the handle, declared in `init` |
| Timers | `handle.registerTimer(ms)`, `listTimers()`, `deleteTimer(ms)` — not allowed in `init` |
| `timeMode` | `"None"`, `"ProcessingTime"`, `"EventTime"` (event time needs a watermark) |
| State schema evolution | Value side only; requires `spark.sql.streaming.stateStore.encodingFormat = avro` |
| Databricks | DBR 16.2+; standard access mode: Python DBR 16.3+, Scala DBR 17.3+; real-time mode: DBR 17.3 LTS+ |
| Photon | Stateful streaming is not accelerated by Photon |

## Example Usage

```python
# Finalized rows (Append): stream straight into Delta — the Delta sink is documented for append/complete
(sessions.writeStream
    .outputMode("append")
    .option("checkpointLocation", "/Volumes/main/ops/checkpoints/sessions")
    .toTable("main.ops.user_sessions"))

# Running per-key values (processor run with outputMode="Update"): upsert with foreachBatch + MERGE
from delta.tables import DeltaTable

def upsert_counts(batch_df, batch_id):
    (DeltaTable.forName(batch_df.sparkSession, "main.ops.user_counts").alias("t")
        .merge(batch_df.alias("s"), "t.user_id = s.user_id")
        .whenMatchedUpdateAll()
        .whenNotMatchedInsertAll()
        .execute())

(running_counts.writeStream
    .outputMode("update")
    .foreachBatch(upsert_counts)
    .option("checkpointLocation", "/Volumes/main/ops/checkpoints/user_counts")
    .start())
```

### Wrong

```python
from pyspark.sql.streaming import TransformWithState   # no such class
class X(TransformWithState): ...
timer_values.register_event_time_timer(ms)               # not an API; use handle.registerTimer
```

## See Also

- [real-time-mode](../concepts/real-time-mode.md)
- [photon](../concepts/photon.md)
- [spark-streaming-patterns](../../streaming/patterns/spark-streaming-patterns.md)

## Sources

- https://spark.apache.org/docs/4.0.0/streaming/structured-streaming-transform-with-state.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.GroupedData.transformWithStateInPandas.html
- https://spark.apache.org/docs/latest/api/python/_modules/pyspark/sql/streaming/stateful_processor.html
- https://docs.databricks.com/aws/en/stateful-applications/
- https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store
- https://docs.databricks.com/aws/en/structured-streaming/delta-lake
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://docs.databricks.com/aws/en/compute/photon
