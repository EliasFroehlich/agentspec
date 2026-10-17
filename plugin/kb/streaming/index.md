# Streaming Knowledge Base

> **Purpose**: Stream processing — Kafka 4.x (KRaft, share groups), Flink 2.x, Spark Structured Streaming 4.x (transformWithState, real-time mode), Debezium 3.x and Flink CDC, RisingWave and Materialize
> **Version Coverage**: Kafka 4.3.1, Flink 2.3.0, Spark 4.2.0 (DBR 17.3 LTS / 18 LTS / 19), Debezium 3.7.0.Final, Flink CDC 3.7.0, RisingWave 3.1.0
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/stream-processing-fundamentals.md](concepts/stream-processing-fundamentals.md) | Event time, watermarks, windowing |
| [concepts/flink-architecture.md](concepts/flink-architecture.md) | Checkpointing, state backends (HashMap/RocksDB/ForSt), Flink 2.x config and SQL additions |
| [concepts/kafka-fundamentals.md](concepts/kafka-fundamentals.md) | Partitions, consumer and share groups, KRaft, ZooKeeper migration, tiered storage, Redpanda |
| [concepts/streaming-databases.md](concepts/streaming-databases.md) | RisingWave 3.x and Materialize (Cloud and Self-Managed) |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/flink-sql-patterns.md](patterns/flink-sql-patterns.md) | Kafka tables, windowed aggregation, temporal joins |
| [patterns/kafka-producer-consumer.md](patterns/kafka-producer-consumer.md) | Idempotent producer, KIP-848 consumer, DLQ, share consumers |
| [patterns/spark-streaming-patterns.md](patterns/spark-streaming-patterns.md) | Kafka source, watermarks, idempotent foreachBatch, stream-stream joins, triggers; stateful/low-latency comparison and minimum versions (APIs in `kb/spark`) |
| [patterns/cdc-patterns.md](patterns/cdc-patterns.md) | Debezium connector config, signal-driven snapshots, Delta CDF, Iceberg changelog |
| [patterns/debezium-operations.md](patterns/debezium-operations.md) | Debezium version matrix, embedded engine, Operator/Server, signal channels, 3.7 features |
| [patterns/flink-cdc-pipelines.md](patterns/flink-cdc-pipelines.md) | Flink CDC 3.x YAML pipelines (route, transform, schema evolution) and `mysql-cdc` SQL |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

## Key Version Changes

| Technology | Current | Headline changes |
|-----------|---------|------------------|
| **Apache Kafka** | 4.3.1 (2026-06-25) | 4.0: ZooKeeper removed, KIP-848 GA; 4.2: share groups production-ready; tiered storage production-ready since 3.9 |
| **Apache Flink** | 2.3.0 (2026-06-25) | 2.0: ForSt, async state, DataSet/Scala APIs removed, `config.yaml` only; 2.1 `ML_PREDICT`; 2.2 `VECTOR_SEARCH`; 2.3 `FROM_CHANGELOG`/`TO_CHANGELOG` |
| **Spark Structured Streaming** | 4.2.0 (2026-07-14) | 4.0 `transformWithState`; 4.1 real-time mode (Scala); 4.2 real-time trigger in PySpark |
| **Databricks** | DBR 17.3 LTS+ | RocksDB state store and changelog checkpointing on by default; real-time mode DBR 16.4 LTS+ |
| **Debezium** | 3.7.0.Final (2026-09-29) | Kafka 4.0 from 3.2; async engine default since 3.2 (introduced 2.6); Operator since 2.3; 3.7 CLI and Platform alerting |
| **RisingWave** | 3.1.0 (2026-09-21) | 3.0: Iceberg V3, pgvector ingestion, WebSocket source, HTTP sink, exactly-once; 3.1: `VARIANT`, refreshed MVs |

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| streaming-engineer | All files | Stream pipeline design |
| spark-streaming-architect | patterns/spark-streaming-patterns.md, ../spark/patterns/transform-with-state.md, ../spark/concepts/real-time-mode.md | Structured Streaming on Spark/Databricks |
| pipeline-architect | patterns/cdc-patterns.md, patterns/flink-cdc-pipelines.md | CDC pipeline orchestration |

## Sources

- https://kafka.apache.org/community/downloads/ · https://flink.apache.org/downloads/ · https://spark.apache.org/news/
- https://docs.databricks.com/aws/en/release-notes/runtime/ · https://debezium.io/releases/
- https://docs.risingwave.com/changelog/release-notes
