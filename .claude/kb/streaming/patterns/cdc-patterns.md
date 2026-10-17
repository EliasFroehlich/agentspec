# CDC Patterns

> **Purpose**: Change Data Capture with Debezium 3.x (Kafka Connect), Delta Lake Change Data Feed and Iceberg changelog views
> **Version Coverage**: Debezium 3.7.0.Final (2026-09-29, built against Kafka Connect 4.3.1); Flink CDC 3.7.0
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Several consumers need the same row-level changes from an OLTP database: Debezium -> Kafka
- One database must land in a lakehouse without Kafka: Flink CDC pipelines ([flink-cdc-pipelines](flink-cdc-pipelines.md))
- A lakehouse table must feed downstream incrementally: Delta CDF or Iceberg changelog views
- Engine, operator, signals and version matrix: [debezium-operations](debezium-operations.md)

## Implementation

```json
{
  "name": "mysql-cdc-orders",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-primary",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "${file:/secrets/mysql.properties:password}",
    "database.server.id": "184054",
    "topic.prefix": "cdc.shop",
    "database.include.list": "shop",
    "table.include.list": "shop.orders,shop.customers",

    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.shop",

    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.delete.tombstone.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,source.ts_ms",

    "snapshot.mode": "initial",
    "signal.enabled.channels": "source",
    "signal.data.collection": "shop.debezium_signal",
    "incremental.snapshot.chunk.size": "1024"
  }
}
```

```sql
-- Signal table (source channel) and an ad hoc incremental snapshot request.
-- Incremental snapshots are triggered by signals; there is no "enable" flag.
CREATE TABLE shop.debezium_signal (id VARCHAR(42) PRIMARY KEY, type VARCHAR(32) NOT NULL, data VARCHAR(2048) NULL);

INSERT INTO shop.debezium_signal (id, type, data) VALUES (
  'snap-orders-001', 'execute-snapshot',
  '{"data-collections": ["shop.orders"], "type": "incremental"}');
```

```sql
-- Delta Lake Change Data Feed. Legacy CDF (OSS Delta, any DBR): set the table property.
-- Databricks DBR 19+: "automatic CDF" on UC tables with row tracking needs no property
-- (and cannot be combined with the legacy property). Same table_changes()/readChangeFeed APIs.
ALTER TABLE main.sales.orders SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

SELECT * FROM table_changes('main.sales.orders', 5)
WHERE _change_type IN ('insert', 'update_postimage')   -- also: update_preimage, delete
ORDER BY _commit_version;
```

```python
# Delta CDF as a streaming source (checkpoint on object storage or a UC volume, never local disk)
(spark.readStream
    .option("readChangeFeed", "true")
    .option("startingVersion", 5)
    .table("main.sales.orders")
    .filter("_change_type IN ('insert', 'update_postimage')")
    .writeStream
    .option("checkpointLocation", "/Volumes/main/sales/chk/orders_cdf")
    .trigger(availableNow=True)
    .toTable("main.sales.orders_downstream"))
```

```sql
-- Iceberg (Spark procedure): changelog view between two snapshots
CALL lake.system.create_changelog_view(
  table              => 'shop.orders',
  options            => map('start-snapshot-id', '1234567890', 'end-snapshot-id', '1234567999'),
  changelog_view     => 'orders_changelog',
  identifier_columns => array('order_id')         -- enables UPDATE_BEFORE / UPDATE_AFTER rows
);
SELECT * FROM orders_changelog WHERE _change_type IN ('INSERT', 'UPDATE_AFTER') ORDER BY _change_ordinal;
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `snapshot.mode` | `initial` | Snapshot once, then stream the binlog |
| `signal.enabled.channels` | `source` enabled by default | Channels: `source`, `kafka`, `file`, `jmx` |
| `signal.data.collection` | none | `<database>.<table>` of the signal table (MySQL) |
| `incremental.snapshot.chunk.size` | `1024` | Rows per incremental snapshot chunk |
| `transforms.unwrap.delete.tombstone.handling.mode` | `tombstone` | `drop`, `tombstone`, `rewrite`, `rewrite-with-tombstone`, `delete-to-tombstone` |
| `transforms.unwrap.add.fields` | none | Metadata copied as `__op`, `__source_ts_ms`, ... |

## Example Usage

| CDC engine | Source | Transport | Snapshot | Output |
|-----------|--------|-----------|----------|--------|
| Debezium | MySQL, Postgres, MongoDB, Oracle, SQL Server, Db2, more | Kafka Connect, Debezium Server, Engine | Initial + signal-driven incremental/blocking | Avro, JSON, Protobuf |
| Flink CDC 3.x | MySQL, Postgres, Oracle, SQL Server, more | Flink job (no Kafka needed) | Parallel incremental snapshot | Pipeline sinks or Flink rows |
| Delta CDF | Delta tables | Spark batch/stream | Version or timestamp range | `_change_type`, `_commit_version` |
| Iceberg changelog | Iceberg tables | Spark procedure | Snapshot or timestamp range | `_change_type`, `_change_ordinal` |

| Pattern | Complexity | Best for |
|---------|------------|----------|
| Debezium -> Kafka -> Flink/Spark | High (3 systems) | Many consumers of the same changes |
| Flink CDC pipeline -> Iceberg/Paimon | Medium | Direct database-to-lake sync with schema evolution |
| Delta CDF streaming | Low | Lake-to-lake incremental pipelines |
| Databricks AUTO CDC | Low | Applying CDC (SCD1/2) inside Lakeflow pipelines; see [cdc-apply-changes](../../lakeflow/patterns/cdc-apply-changes.md) |

## Common Mistakes

### Wrong

```json
{
  "transforms.unwrap.drop.tombstones": "false",
  "transforms.unwrap.delete.handling.mode": "rewrite",
  "incremental.snapshot.enabled": "true"
}
```

`delete.handling.mode` / `drop.tombstones` are not in the current SMT reference (replaced by
`delete.tombstone.handling.mode`), and `incremental.snapshot.enabled` is not a MySQL connector property.
The signal table does not need to be in `table.include.list` (it does need its columns if you use `column.include.list`).

### Correct

```json
{
  "transforms.unwrap.delete.tombstone.handling.mode": "rewrite",
  "signal.data.collection": "shop.debezium_signal"
}
```

## See Also

- [debezium-operations](debezium-operations.md)
- [flink-cdc-pipelines](flink-cdc-pipelines.md)
- [kafka-fundamentals](../concepts/kafka-fundamentals.md)
- [spark-streaming-patterns](spark-streaming-patterns.md)
- [streaming-databases](../concepts/streaming-databases.md)

## Sources

- https://debezium.io/releases/ · https://debezium.io/releases/3.7/release-notes
- https://debezium.io/documentation/reference/stable/connectors/mysql.html
- https://debezium.io/documentation/reference/stable/configuration/signalling.html
- https://debezium.io/documentation/reference/stable/transformations/event-flattening.html
- https://docs.databricks.com/aws/en/structured-streaming/checkpoints · https://docs.databricks.com/aws/en/delta/delta-change-data-feed
- https://iceberg.apache.org/docs/latest/spark-procedures/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/core-concept/data-pipeline/
