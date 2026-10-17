# Streaming Quick Reference

> **Purpose**: Fast lookup tables for Kafka, Flink, Spark Structured Streaming, Debezium and streaming databases
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Current Versions

| Technology | Current | Notes |
|-----------|---------|-------|
| Apache Kafka | 4.3.1 (2026-06-25) | KRaft only since 4.0; 3.9.x was the last ZooKeeper (bridge) release |
| Apache Flink | 2.3.0 (2026-06-25) | `config.yaml` only; Flink CDC 3.7.0 |
| Apache Spark | 4.2.0 (2026-07-14) | DBR 17.3 LTS = Spark 4.0.0, DBR 18 LTS = 4.1.0, DBR 19 = 4.2.0 |
| Debezium | 3.7.0.Final (2026-09-29) | Built against Kafka Connect 4.3.1 |
| RisingWave | 3.1.0 (2026-09-21) | 3.0.5 patch (2026-10-12) |
| Materialize | v26.x | Cloud or Self-Managed (license key; free Community tier) |

## Framework Comparison

| Feature | Flink 2.3 | Spark Structured Streaming 4.2 | RisingWave 3.1 | Kafka Streams |
|---------|-----------|--------------------------------|----------------|---------------|
| Latency | ms | Micro-batch seconds; real-time mode sub-second | Sub-second | ms |
| Language | Java, SQL, Python | Python, Scala, Java, SQL | SQL (Postgres) | Java |
| State | HashMap, RocksDB, ForSt | HDFS-backed or RocksDB (RocksDB default on DBR 17.3+) | Hummock on object storage | RocksDB |
| Exactly-once | Checkpoints + transactional sinks | Checkpoint/WAL + idempotent sinks | Yes (exactly-once sinks in 3.0) | Transactions |
| Best for | Complex event processing | Lakehouse batch + stream | Streaming SQL apps | Kafka-native services |

## Kafka 4.x

| Change | Detail |
|--------|--------|
| ZooKeeper | Removed in 4.0; migrate on 3.9 (`zookeeper.metadata.migration.enable`) before upgrading |
| Controller quorum | Dynamic: `controller.quorum.bootstrap.servers` + `kafka-storage.sh format --standalone` |
| Consumer protocol | KIP-848 GA in 4.0; clients opt in with `group.protocol=consumer` |
| Share groups (KIP-932) | Early access 4.0, preview 4.1, **production-ready 4.2**; `share.version=1`; Java `KafkaShareConsumer` |
| Tiered storage | Production-ready since 3.9 (KIP-405); needs a `RemoteStorageManager` plugin |
| Java | Brokers/Connect/tools Java 17; clients/Streams Java 11 |

## Flink 2.x

| Change | Detail |
|--------|--------|
| Config | `execution.checkpointing.dir`, `execution.checkpointing.savepoint-dir`, `execution.checkpointing.incremental` |
| State backend | `state.backend.type: hashmap \| rocksdb \| forst` (ForSt = disaggregated, experimental) |
| Removed in 2.0 | DataSet API, Scala API, `SourceFunction`/`SinkFunction`, `flink-conf.yaml` |
| SQL additions | Materialized tables 1.20; `ML_PREDICT` 2.1; `VECTOR_SEARCH` 2.2; `FROM_CHANGELOG`/`TO_CHANGELOG` 2.3 |

## Spark Structured Streaming

| Feature | Apache Spark | Databricks |
|---------|--------------|------------|
| `transformWithState` | 4.0+ (row-based Python 4.1+) | DBR 16.2+ |
| Real-time mode | 4.1 Scala stateless; 4.2 PySpark trigger | DBR 16.4 LTS+, Photon off, `outputMode("update")` |
| RocksDB changelog checkpointing | `spark.sql.streaming.stateStore.rocksdb.changelogCheckpointing.enabled` | Default on DBR 17.3+ |
| Continuous trigger | Experimental, legacy | Not supported |

## Debezium 3.x

| Feature | Detail |
|---------|--------|
| Kafka 4.0 | Built and tested from 3.2 (3.1 was built on Kafka Connect 3.9.0) |
| Engine | Async engine since 2.6; default (old engine removed) since 3.2 |
| Operator | Since 2.3 (preview); Helm chart `debezium/debezium-operator` |
| Unwrap SMT | `delete.tombstone.handling.mode` (`drop`, `tombstone`, `rewrite`, ...) |
| Incremental snapshot | Signal-driven (`signal.data.collection`, `execute-snapshot`) |
| 3.7 | Debezium CLI, Platform alerting, incubating TiDB/Milvus/SQLite |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Process-time windows when event time exists | Event time + watermarks |
| Unbounded state | State TTL, watermarks, temporal filters |
| Local or shared checkpoint paths | One durable path per query (object storage or `/Volumes/...`) |
| Continuous-processing trigger | `.trigger(realTime="5 minutes")` (Spark 4.2+/DBR 16.4 LTS+) or `processingTime` |
| Legacy `state.*` checkpoint/savepoint keys in Flink 2.x | `execution.checkpointing.dir` / `.savepoint-dir` / `.incremental` |
| Old unwrap-SMT delete options | `delete.tombstone.handling.mode` |
| A "group type" client property for queues | Enable `share.version` and use a share consumer |
| Session/heartbeat timeouts on a `consumer`-protocol client | Set `group.consumer.*` on the broker/group |

## Related Documentation

| Topic | Path |
|-------|------|
| Fundamentals | `concepts/stream-processing-fundamentals.md` |
| Kafka | `concepts/kafka-fundamentals.md`, `patterns/kafka-producer-consumer.md` |
| Spark | `patterns/spark-streaming-patterns.md`; APIs: `../spark/patterns/transform-with-state.md`, `../spark/concepts/real-time-mode.md` |
| CDC | `patterns/cdc-patterns.md`, `patterns/debezium-operations.md`, `patterns/flink-cdc-pipelines.md` |
| Full Index | `index.md` |

## Sources

- https://kafka.apache.org/community/downloads/ · https://kafka.apache.org/42/getting-started/upgrade/
- https://flink.apache.org/downloads/ · https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/config/
- https://spark.apache.org/news/ · https://docs.databricks.com/aws/en/structured-streaming/real-time/setup
- https://docs.databricks.com/aws/en/stateful-applications/ · https://docs.databricks.com/aws/en/structured-streaming/rocksdb-state-store
- https://debezium.io/releases/ · https://docs.risingwave.com/changelog/release-notes · https://materialize.com/docs/self-managed-deployments/
