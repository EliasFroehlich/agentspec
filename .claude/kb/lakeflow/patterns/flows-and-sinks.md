# Flows and Sinks

> **Purpose**: Explicit flows (append, once/backfill, multi-source), sinks to Delta/Kafka/Event Hubs/custom sources, and the ForEachBatch sink
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Several streaming sources must append into one streaming table without a `UNION`
- A streaming table needs a one-time historical backfill
- Pipeline output must go to Kafka, Event Hubs, an external Delta table or a custom destination
- Per-micro-batch logic (MERGE into another table, multiple writes) is required

## Implementation

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

# One target, many append flows
dp.create_streaming_table("events_all")

@dp.append_flow(target="events_all", name="events_web")
def events_web():
    return spark.readStream.table("main.raw.web_events")

@dp.append_flow(target="events_all", name="events_mobile")
def events_mobile():
    return spark.readStream.table("main.raw.mobile_events")

# One-time backfill: batch DataFrame, runs once (again only on full refresh)
@dp.append_flow(target="events_all", name="events_backfill_2024", once=True)
def events_backfill():
    return spark.read.table("main.archive.events_2024")

# Delta sink by table name (must be fully qualified)
dp.create_sink(
    name="events_export",
    format="delta",
    options={"tableName": "main.export.events"},
)

@dp.append_flow(target="events_export", name="to_events_export")
def to_events_export():
    return spark.readStream.table("events_all").select("event_id", "event_ts", "user_id")

# Kafka (or Event Hubs via its Kafka interface) sink
dp.create_sink(
    name="events_kafka",
    format="kafka",
    options={
        "kafka.bootstrap.servers": spark.conf.get("kafka_servers"),
        "topic": "events",
    },
)

@dp.append_flow(target="events_kafka", name="to_kafka")
def to_kafka():
    return (spark.readStream.table("events_all")
            .select(F.col("event_id").cast("string").alias("key"),
                    F.to_json(F.struct("*")).alias("value")))

# ForEachBatch sink: arbitrary per-batch logic
@dp.foreach_batch_sink(name="events_merge_sink")
def events_merge_sink(df, batch_id):
    df.createOrReplaceTempView("batch_events")
    df.sparkSession.sql("""
        MERGE INTO main.serving.events t
        USING batch_events s ON t.event_id = s.event_id
        WHEN MATCHED THEN UPDATE SET * WHEN NOT MATCHED THEN INSERT *
    """)

@dp.append_flow(target="events_merge_sink", name="to_merge_sink")
def to_merge_sink():
    return spark.readStream.table("events_all")
```

```sql
-- SQL flows (sinks are Python-only)
CREATE OR REFRESH STREAMING TABLE events_all;
CREATE FLOW events_web    AS INSERT INTO events_all BY NAME SELECT * FROM STREAM(main.raw.web_events);
CREATE FLOW events_hist   AS INSERT ONCE INTO events_all BY NAME SELECT * FROM main.archive.events_2024;
```

## Configuration

| Item | Rule |
|------|------|
| Default flow | Defining a streaming table/MV with a query creates a flow with the target's name |
| Append flows per target | Any number |
| `once=True` / `INSERT ONCE` | Batch query; reruns only on full refresh |
| `depends_on` on `append_flow` | Public Preview; orders flows (in continuous pipelines predecessors must be `once`) |
| `import_checkpoint` | Beta; migrate an existing Structured Streaming checkpoint into a new target |
| `dp.create_sink(name, format, options)` | `format` is `delta` or `kafka` (or a registered Python data source) |
| Sink flow types | Only `append_flow` and `update_flow` (Public Preview); no AUTO CDC into sinks |
| Sink limits | Python only, streaming only, no expectations, full refresh does not clean the sink |
| ForEachBatch | Pipeline resets the checkpoint on full refresh (`batch_id` restarts at 0); you clean the targets |
| Serverless egress control | Supports only the Kafka and Delta sink connectors |

## Example Usage

```python
# Wrong: legacy sink signature and keyword arguments that do not exist
dlt.create_sink(target="k", source="t", kafka_bootstrap_servers="b:9092")  # Wrong

# Correct
dp.create_sink("k", "kafka", {"kafka.bootstrap.servers": "b:9092", "topic": "t"})
```

For Event Hubs, authenticate with a Unity Catalog service credential:
`options={"databricks.serviceCredential": "<credential>", "kafka.bootstrap.servers":
"<namespace>.servicebus.windows.net:9093", "topic": "<hub>"}`.

## See Also

- [Python decorators](python-decorators.md)
- [SQL streaming](sql-streaming.md)
- [Core concepts](../concepts/core-concepts.md)
- [Limitations](../reference/limitations.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/concepts/flows
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-append-flow
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-sink
- https://docs.databricks.com/aws/en/ldp/ldp-sinks
- https://docs.databricks.com/aws/en/ldp/for-each-batch
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-flow
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
