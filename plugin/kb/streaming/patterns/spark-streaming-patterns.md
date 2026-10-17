# Spark Streaming Patterns

> **Purpose**: Structured Streaming core patterns: Kafka source, watermarks, windowed aggregation, idempotent foreachBatch, stream-stream joins, triggers
> **Version Coverage**: Apache Spark 4.2.0 (2026-07-14); Databricks DBR 17.3 LTS (Spark 4.0.0), 18 LTS (Spark 4.1.0), 19 (Spark 4.2.0)
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Incremental ingestion from Kafka or Delta into Delta/Unity Catalog tables
- Event-time aggregations that must tolerate late data
- Enriching one stream with another inside a bounded time range
- Choosing a trigger for cost vs latency (`availableNow`, `processingTime`, real-time mode)

API details live in the spark domain: [transform-with-state](../../spark/patterns/transform-with-state.md) and
[real-time-mode](../../spark/concepts/real-time-mode.md). This file keeps the comparison and minimum versions.

## Implementation

```python
from pyspark.sql import DataFrame, functions as F
from pyspark.sql.types import StructType, StructField, StringType, DecimalType, TimestampType

orders_schema = StructType([
    StructField("order_id", StringType()), StructField("customer_id", StringType()),
    StructField("amount", DecimalType(12, 2)), StructField("order_ts", TimestampType()),
])
clicks_schema = StructType([
    StructField("customer_id", StringType()), StructField("page", StringType()),
    StructField("click_ts", TimestampType()),
])

def kafka_stream(topic: str):
    return (spark.readStream.format("kafka")
            .option("kafka.bootstrap.servers", "broker1:9092")   # required
            .option("subscribe", topic)
            .option("startingOffsets", "earliest")
            .load())

# 1. Kafka -> typed columns + watermark
orders = (kafka_stream("orders")
          .select(F.from_json(F.col("value").cast("string"), orders_schema).alias("o"))
          .select("o.*")
          .withWatermark("order_ts", "10 minutes"))

# 2. Event-time windowed aggregation
revenue_5m = (orders
              .groupBy(F.window("order_ts", "5 minutes"))
              .agg(F.count("*").alias("order_count"), F.sum("amount").alias("revenue")))

# 3. foreachBatch with idempotent Delta writes (retries of the same batch_id are skipped)
def write_orders(batch_df: DataFrame, batch_id: int) -> None:
    (batch_df.dropDuplicates(["order_id"])
        .write.format("delta").mode("append")
        .option("txnAppId", "orders-silver-writer")   # stable per query
        .option("txnVersion", batch_id)
        .saveAsTable("main.sales.orders_silver"))

orders_query = (orders.writeStream
    .foreachBatch(write_orders)
    .option("checkpointLocation", "/Volumes/main/sales/chk/orders_silver")  # one per query
    .trigger(availableNow=True)                                              # drain, then stop
    .start())

# 4. Stream-stream left outer join: watermarks on both sides + time-range condition
clicks = (kafka_stream("clicks")
          .select(F.from_json(F.col("value").cast("string"), clicks_schema).alias("c"))
          .select("c.*")
          .withWatermark("click_ts", "15 minutes"))

enriched = orders.alias("o").join(
    clicks.alias("c"),
    F.expr("""
        o.customer_id = c.customer_id AND
        c.click_ts BETWEEN o.order_ts - INTERVAL 30 MINUTES AND o.order_ts
    """),
    "leftOuter",
)

(enriched.writeStream
    .outputMode("append")                        # outer stream-stream joins emit in append mode
    .option("checkpointLocation", "/Volumes/main/sales/chk/orders_enriched")
    .trigger(processingTime="1 minute")
    .toTable("main.sales.orders_enriched"))
```

## Configuration

| Trigger | Syntax (PySpark) | Use case |
|---------|------------------|----------|
| Default micro-batch | none | Continuous micro-batches as data arrives |
| Processing time | `.trigger(processingTime="30 seconds")` | Near-real-time with lower per-batch overhead |
| Available now | `.trigger(availableNow=True)` | Scheduled incremental loads; replaces deprecated `once=True` |
| Real-time mode | `.trigger(realTime="5 minutes")` | Sub-second operational workloads; Spark 4.2+ PySpark, DBR 16.4 LTS+ ([details](../../spark/concepts/real-time-mode.md)) |
| Continuous (legacy) | `continuous` trigger argument | Experimental, at-least-once; Spark docs: "use Real-time Mode instead"; not supported on Databricks |

| Setting | Default | Description |
|---------|---------|-------------|
| `checkpointLocation` | none | Durable path (object storage or `/Volumes/...`); never shared between queries |
| `spark.sql.adaptive.streaming.stateless.enabled` | `true` (4.1+) | AQE for stateless streaming queries; set `false` if a regression appears |
| `spark.sql.streaming.checkpoint.verifyMetadataExists.enabled` | `true` (4.2+) | Fail with `STREAMING_CHECKPOINT_MISSING_METADATA_FILE` instead of silently starting a new query ID |
| `spark.sql.streaming.stateStore.providerClass` | RocksDB on DBR 17.3+; HDFS-backed in OSS | DBR ≤ 17.2: `com.databricks.sql.streaming.state.RocksDBStateStoreProvider`; OSS: `org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider` |
| `spark.sql.streaming.stateStore.rocksdb.changelogCheckpointing.enabled` | `true` on DBR 17.3+; `false` in OSS and DBR < 17.3 | Upload only changed state at commit |

## Example Usage

| Spark | Databricks | Streaming changes |
|-------|------------|-------------------|
| 4.0.0 (2025-05-23) | DBR 17.3 LTS | `transformWithState` / `transformWithStateInPandas`; `availableNow` falls back to one batch if a source lacks support |
| 4.1.0 (2025-12-16) | DBR 18 LTS | Real-time mode (Scala, stateless); row-based Python `transformWithState`; AQE for stateless queries; Spark Declarative Pipelines |
| 4.2.0 (2026-07-14) | DBR 19 | RTM trigger in PySpark; stream-stream non-outer join in Update mode; stable source/sink names (`DataStreamReader.name()`); strict checkpoint metadata check |

## Stateful and Low-Latency: Engine Comparison

| Need | Spark Structured Streaming | Flink | Kafka Streams | RisingWave |
|------|----------------------------|-------|---------------|------------|
| Custom per-key state + timers | `transformWithState` (Spark 4.0+; DBR 16.2+) | Keyed state + timers; async State V2 (2.0+) | Processor API + state stores | SQL MVs only (UDFs for logic) |
| State backend | HDFS-backed or RocksDB (default on DBR 17.3+) | HashMap, RocksDB, ForSt | RocksDB | Hummock on object storage |
| Sub-second latency | Real-time mode: Spark 4.1 (Scala, stateless) / 4.2 (PySpark); DBR 16.4 LTS+, Photon off, update mode, Kafka-style sources/sinks only (no Delta) | Native (record-at-a-time) | Native | Sub-second MV refresh |

| Minimum versions | Apache Spark | Databricks |
|------------------|--------------|------------|
| `transformWithState` / `transformWithStateInPandas` | 4.0 (row-based Python 4.1) | DBR 16.2+ (standard access mode: Python 16.3+, Scala 17.3+) |
| `transformWithState` in real-time mode | — | DBR 17.3 LTS+ (`transformWithStateInPandas` not supported in RTM) |
| Stream-stream joins in real-time mode with other streams on the cluster | — | DBR 18 LTS+ |
| `TwsTester` (unit-test a `StatefulProcessor`) | 4.2 docs | — |

## Common Mistakes

### Wrong

```python
orders.join(clicks, expr=F.expr("..."), how="left")   # TypeError: join() has no 'expr' argument
df.writeStream.trigger(continuous="1 second")          # legacy; unsupported on Databricks
df.writeStream.option("checkpointLocation", "/tmp/chk")  # local path: lost on restart
```

### Correct

```python
orders.alias("o").join(clicks.alias("c"), F.expr("o.customer_id = c.customer_id AND ..."), "leftOuter")
df.writeStream.option("checkpointLocation", "/Volumes/main/sales/chk/orders").trigger(availableNow=True)
```

## See Also

- [transform-with-state](../../spark/patterns/transform-with-state.md) · [real-time-mode](../../spark/concepts/real-time-mode.md)
- [kafka-producer-consumer](kafka-producer-consumer.md)
- [stream-processing-fundamentals](../concepts/stream-processing-fundamentals.md)
- [cdc-patterns](cdc-patterns.md)
- [performance-tuning](../../spark/patterns/performance-tuning.md)

## Sources

- https://spark.apache.org/news/ · https://spark.apache.org/releases/spark-release-4.1.0.html · https://spark.apache.org/releases/spark-release-4-2-0.html
- https://spark.apache.org/docs/latest/streaming/ss-migration-guide.html
- https://spark.apache.org/docs/latest/streaming/apis-on-dataframes-and-datasets.html
- https://apache.github.io/spark/streaming/real-time-mode.html
- https://docs.databricks.com/aws/en/structured-streaming/triggers
- https://docs.databricks.com/aws/en/structured-streaming/checkpoints
- https://docs.databricks.com/aws/en/structured-streaming/foreach
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://docs.databricks.com/aws/en/stateful-applications/ · https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store
- https://docs.databricks.com/aws/en/structured-streaming/real-time/setup · https://docs.databricks.com/aws/en/structured-streaming/real-time/reference
- https://spark.apache.org/docs/4.2.0/streaming/structured-streaming-transform-with-state.html
