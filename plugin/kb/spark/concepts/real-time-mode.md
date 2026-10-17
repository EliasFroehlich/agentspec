# Real-Time Mode

> **Purpose**: Structured Streaming trigger modes — micro-batch, AvailableNow, and real-time mode (Databricks DBR 16.4 LTS+, OSS Spark 4.1+) with its real prerequisites; watermarks and output modes
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Structured Streaming uses the DataFrame API over unbounded inputs. The **trigger** decides how data
is processed: micro-batches (default), `availableNow` (process what exists, then stop) or
**real-time mode**, a trigger type for sub-second operational workloads (Databricks documents
end-to-end latency "as low as five milliseconds"). Real-time mode is not automatic: it is an explicit
trigger with strict compute and query requirements. Watermarks bound state for event-time
aggregations in every mode.

| Engine | Real-time mode availability |
|--------|-----------------------------|
| Databricks Structured Streaming | DBR 16.4 LTS+ (so 17.3 LTS, 18 LTS, 19) on classic compute; serverless compute not supported; stream-stream joins with other streams on the same cluster need DBR 18 LTS+ |
| Databricks Lakeflow pipelines (classic or serverless) | Not via Structured Streaming triggers, but via pipeline configuration: continuous mode + `spark.databricks.streaming.realTimeMode.enabled` in the pipeline `spark_conf` + an update flow with `pipelines.trigger = "RealTime"` (Public Preview, DBR 18.1.3 preview channel) |
| Apache Spark | 4.1.0 (Scala, stateless queries); PySpark `trigger(realTime=...)` in 4.2.0 |

## The Concept

```python
# Databricks real-time mode — compute prerequisites (setup page), set on the CLUSTER, not in the notebook:
#   classic compute (dedicated, or standard for Python only; serverless compute not supported)
#   autoscaling OFF, Photon OFF, spot instances OFF
#   cluster Spark config: spark.databricks.streaming.realTimeMode.enabled true

query = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", broker)
    .option("subscribe", "orders")
    .load()
    .selectExpr("CAST(key AS STRING) AS key", "value")
    .writeStream
    .format("kafka")
    .option("kafka.bootstrap.servers", broker)
    .option("topic", "orders_enriched")
    .option("checkpointLocation", checkpoint_path)
    .outputMode("update")                 # real-time mode supports update mode only
    .trigger(realTime="5 minutes")        # PySpark requires the interval (micro-batch/checkpoint length)
    .start())
```

```python
# Micro-batch with watermark (any mode) — bounded state for windowed aggregation
revenue_per_minute = (orders_stream
    .withWatermark("event_time", "10 minutes")
    .groupBy(F.window("event_time", "1 minute"))
    .agg(F.sum("amount").alias("total_revenue")))
```

## Quick Reference

| Trigger | Syntax (Python) | Use case |
|---------|-----------------|----------|
| Default (unspecified) | — | `processingTime` 0 ms; continuous micro-batches (~seconds latency); watch cloud-storage API costs |
| Processing time | `.trigger(processingTime="10 seconds")` | Balance cost and latency |
| Available now | `.trigger(availableNow=True)` | Scheduled incremental batch, backfills |
| Real-time mode | `.trigger(realTime="5 minutes")` | Sub-second operational workloads (fraud, personalization) |
| Continuous (OSS, experimental) | `.trigger(continuous="1 second")` | **Not supported on Databricks** — use real-time mode |

| Real-time mode constraint (Databricks) | Detail |
|----------------------------------------|--------|
| Output mode | `update` only (no `append`/`complete`) |
| Sources | Kafka, Event Hubs (Kafka connector), Kinesis (EFO recommended), AWS MSK |
| Sinks | Kafka, Event Hubs, `foreachWriter`; **Delta is not a source or sink** |
| Windows | Tumbling and sliding supported; session windows not supported |
| Arbitrary state | Row-based `transformWithState` only (`transformWithStateInPandas` not supported); DBR 17.3 LTS+ |
| Sizing | Task slots ≥ total tasks across all stages (e.g. 8 Kafka partitions + 20 shuffle partitions = 28 slots) |

| Output mode | Behavior | Use with |
|-------------|----------|----------|
| `append` | Only finalized rows (after watermark) | Windowed aggs, non-agg queries |
| `update` | Changed rows only | Aggregations, stateful ops, real-time mode |
| `complete` | Entire result table | Small aggregations only |

## Common Mistakes

### Wrong

```python
# Expecting real-time latency from a regular job cluster with Photon + autoscaling,
# Delta as sink, and no trigger
(stream.writeStream.format("delta").outputMode("append")
    .option("checkpointLocation", cp).toTable("main.ops.events"))
```

### Correct

```python
# Real-time mode: explicit trigger, update mode, supported sink, Photon/autoscaling off on the cluster
(stream.writeStream.format("kafka").option("kafka.bootstrap.servers", broker)
    .option("topic", "events_out").option("checkpointLocation", cp)
    .outputMode("update").trigger(realTime="5 minutes").start())
# For Delta targets and analytical latency (seconds-minutes), use micro-batch or availableNow.
```

## Related

- [dataframe-api](../concepts/dataframe-api.md)
- [photon](../concepts/photon.md)
- [transform-with-state](../patterns/transform-with-state.md)
- [spark-streaming-patterns](../../streaming/patterns/spark-streaming-patterns.md)

## Sources

- https://docs.databricks.com/aws/en/structured-streaming/real-time/
- https://docs.databricks.com/aws/en/structured-streaming/real-time/setup
- https://docs.databricks.com/aws/en/structured-streaming/real-time/reference
- https://docs.databricks.com/aws/en/structured-streaming/triggers
- https://docs.databricks.com/aws/en/ldp/real-time
- https://docs.databricks.com/aws/en/stateful-applications/
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/releases/spark-release-4-2-0.html
