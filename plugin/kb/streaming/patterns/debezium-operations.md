# Debezium Operations

> **Purpose**: Run Debezium 3.x beyond a single connector: version/Kafka matrix, embedded engine, Debezium Server on Kubernetes, signals
> **Version Coverage**: Debezium 3.7.0.Final (2026-09-29); 3.6.3.Final (2026-09-18)
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Choosing a Debezium release for a given Kafka / Kafka Connect version
- Embedding CDC in a JVM application without a Kafka Connect cluster (Debezium Engine)
- Running Debezium Server (Kafka or non-Kafka sinks) on Kubernetes through the Debezium Operator
- Triggering incremental or blocking snapshots on a running connector

## Implementation

```java
// Debezium Engine (embedded). Since 3.2.0 the default implementation is AsyncEmbeddedEngine
// (introduced in 2.6.0); the old EmbeddedEngine is no longer available.
final Properties props = new Properties();
props.setProperty("name", "orders-engine");
props.setProperty("connector.class", "io.debezium.connector.mysql.MySqlConnector");
props.setProperty("offset.storage", "org.apache.kafka.connect.storage.FileOffsetBackingStore");
props.setProperty("offset.storage.file.filename", "/var/lib/cdc/offsets.dat");
props.setProperty("offset.flush.interval.ms", "60000");
props.setProperty("database.hostname", "mysql-primary");
props.setProperty("database.port", "3306");
props.setProperty("database.user", "debezium");
props.setProperty("database.password", System.getenv("MYSQL_PASSWORD"));
props.setProperty("database.server.id", "85744");
props.setProperty("topic.prefix", "shop");
props.setProperty("schema.history.internal", "io.debezium.storage.file.history.FileSchemaHistory");
props.setProperty("schema.history.internal.file.filename", "/var/lib/cdc/schemahistory.dat");
props.setProperty("record.processing.threads", "4");        // async engine thread pool
props.setProperty("record.processing.order", "ORDERED");    // or UNORDERED

try (DebeziumEngine<ChangeEvent<String, String>> engine = DebeziumEngine.create(Json.class)
        .using(props)
        .notifying(record -> handleChange(record.key(), record.value()))
        .build()) {
    ExecutorService executor = Executors.newSingleThreadExecutor();
    executor.execute(engine);
    awaitShutdownSignal();   // engine stops when the try block exits
}
```

```bash
# Debezium Operator (Helm) -> manages Debezium Server instances from DebeziumServer resources
kubectl create namespace debezium
helm repo add debezium https://charts.debezium.io
helm install debezium-operator debezium/debezium-operator --version 3.7.0 -n debezium
```

```yaml
apiVersion: debezium.io/v1alpha1
kind: DebeziumServer
metadata:
  name: mysql-orders
  namespace: debezium
spec:
  # spec.version defaults to the operator version (format "3.7.0"); spec.image overrides it
  source:
    class: io.debezium.connector.mysql.MySqlConnector
    config:
      database.hostname: mysql-primary
      database.port: 3306
      database.user: debezium
      database.password: <from-k8s-secret>   # expose a Secret via spec.runtime.environment
      database.server.id: 184055
      topic.prefix: shop
      table.include.list: shop.orders
    offset:
      type: kafka
      config: {bootstrap.servers: "kafka:9092", topic: debezium-offsets}
    schemaHistory:
      type: kafka
      config: {bootstrap.servers: "kafka:9092", topic: debezium-schema-history}
  sink:
    type: kafka                     # other Debezium Server sinks use their own type
    config:
      bootstrap.servers: kafka:9092
  format:
    value: {type: json, config: {schemas.enable: false}}
```

```properties
# Kafka signal channel in addition to the default source channel
signal.enabled.channels=source,kafka
signal.kafka.topic=cdc-signals              # default: <topic.prefix>-signal
signal.kafka.bootstrap.servers=kafka:9092
# message key must equal topic.prefix; value:
# {"type":"execute-snapshot","data":{"data-collections":["shop.orders"],"type":"INCREMENTAL"}}
```

## Configuration

| Debezium | Date | Kafka Connect base | Notes |
|----------|------|--------------------|-------|
| 2.3.0.Final | 2023-06-21 | — | Debezium Operator introduced as a preview |
| 2.6.0.Final | 2024-04 | — | AsyncEmbeddedEngine introduced; Debezium Server switched to it |
| 3.0.x | 2024-10-02 (3.0.0) | 3.9.0 (3.0.8) | Connectors need Java 17; Server/Operator/Outbox need Java 21 |
| 3.1.x | 2025 | 3.9.0 | Last line built on Kafka 3.x |
| 3.2.0.Final | 2025-07-09 | Kafka 4.0 | Built and tested with Kafka 4.0; async engine becomes the default |
| 3.7.0.Final | 2026-09-29 | 4.3.1 | Tested with Kafka Connect 3.1+; Java 17+ (connectors), 21+ (Server, Operator) |

| Engine property | Purpose |
|-----------------|---------|
| `record.processing.threads` | Thread pool size for record processing (default sized from CPU cores) |
| `record.processing.order` | `ORDERED` or `UNORDERED` delivery to the consumer |
| `record.processing.shutdown.timeout.ms` | Wait for in-flight records at shutdown |

## Example Usage

What arrived in 3.7 (2026-09-29):

| Area | Change |
|------|--------|
| Connectors | Incubating TiDB, Milvus and SQLite connectors; CockroachDB no longer incubating |
| Tooling | First official Debezium CLI (pipeline lifecycle from one command line) |
| Debezium Platform | Alerting; can deploy pipelines to hosts outside Kubernetes |
| Debezium Server | Native distribution module; Databricks Zerobus Ingest sink |
| JDBC sink | Dead letter queue option; validate-only schema evolution mode |
| AI | PyDebeziumAI: keeps vector stores in sync with relational sources |

## Common Mistakes

### Wrong

```text
"Debezium 3.x introduced the async engine and the Kubernetes operator"
"Debezium 3.1 runs on Kafka Connect 4.0"
```

### Correct

```text
Async engine: 2.6 (default since 3.2). Operator: 2.3 preview. Kafka 4.0 build: 3.2+.
```

## See Also

- [cdc-patterns](cdc-patterns.md)
- [flink-cdc-pipelines](flink-cdc-pipelines.md)
- [kafka-fundamentals](../concepts/kafka-fundamentals.md)

## Sources

- https://debezium.io/releases/ · https://debezium.io/releases/3.7/ · https://debezium.io/releases/3.7/release-notes
- https://debezium.io/blog/2026/09/29/debezium-3-7-final-released/
- https://debezium.io/blog/2025/07/09/debezium-3-2-final-released/
- https://debezium.io/releases/3.1/release-notes · https://debezium.io/releases/3.0/release-notes
- https://debezium.io/blog/2024/07/08/async-embedded-engine/
- https://debezium.io/documentation/reference/stable/development/engine.html
- https://debezium.io/blog/2023/06/21/debezium-2-3-final-released/
- https://debezium.io/documentation/reference/stable/operations/debezium-operator.html · https://charts.debezium.io/index.yaml
- https://debezium.io/documentation/reference/stable/configuration/signalling.html
