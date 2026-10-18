# Kafka Fundamentals

> **Purpose**: Topics, partitions, consumer and share groups, exactly-once, KRaft, tiered storage, Kafka 4.x changes, Redpanda
> **Version Coverage**: Apache Kafka 4.3.1 (2026-06-25); 4.0+ is KRaft-only; 3.9.x is the last ZooKeeper (bridge) release
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Apache Kafka is a distributed event streaming platform built on a partitioned, replicated commit log. **Topics** group events, **partitions** give parallelism and per-key ordering, **consumer groups** assign each partition to one member, and **share groups** (KIP-932, production-ready in 4.2) let many members cooperatively consume the same partitions with per-record acknowledgement. Exactly-once comes from idempotent producers plus transactions. Since 4.0 the metadata layer is KRaft only.

## The Concept

```python
# Topic "orders" (3 partitions, RF=3): P0 leader B1 | P1 leader B2 | P2 leader B3
# Consumer group: one member per partition, offsets committed per partition
# Share group: members share partitions; broker tracks each record (ACCEPT / RELEASE / REJECT)
from confluent_kafka import Producer
import json

producer = Producer({
    "bootstrap.servers": "broker1:9092",
    "enable.idempotence": True,        # no duplicates on retry
    "acks": "all",                     # wait for all in-sync replicas
    "compression.type": "zstd",
})
order = {"order_id": "o-1", "customer_id": 42}
producer.produce("orders", key=str(order["customer_id"]).encode(),  # same key -> same partition
                 value=json.dumps(order).encode())
producer.flush()
```

## Quick Reference

| Concept | Description | Default / note |
|---------|-------------|----------------|
| Partition | Ordered, immutable log inside a topic | `num.partitions=1` |
| Consumer group | One member per partition; offsets committed per partition | `group.protocol=classic` unless set to `consumer` |
| Share group | Members share partitions; per-record ack and delivery count | Production-ready in 4.2 (KIP-932) |
| ISR | Replicas caught up with the leader | `acks=all` waits for the ISR |
| Compacted topic | Keeps the latest value per key | `cleanup.policy=compact` |
| Tiered storage | Old segments offloaded to remote storage | Production-ready since 3.9 (KIP-405) |

| Kafka vs Redpanda | Apache Kafka 4.x | Redpanda |
|-------------------|------------------|----------|
| Runtime | JVM (brokers need Java 17) | C++ (no JVM) |
| Metadata / consensus | KRaft (ZooKeeper removed in 4.0) | Raft, built in |
| API | Native | Kafka-compatible |
| Tiered storage | Native; you supply a `RemoteStorageManager` plugin | Built in; requires an Enterprise license |
| Schema Registry | External (Confluent, Apicurio) | Built into the broker binary |

Latency comparisons between the two depend on hardware and workload; benchmark your own case.

## Common Mistakes

### Wrong

```python
# Auto-commit: offsets may be committed before processing finishes
{"bootstrap.servers": "broker1:9092", "group.id": "g1", "enable.auto.commit": True}
```

### Correct

```python
from confluent_kafka import Consumer
consumer = Consumer({
    "bootstrap.servers": "broker1:9092",
    "group.id": "g1",
    "enable.auto.commit": False,          # commit only after processing
    "auto.offset.reset": "earliest",
    "isolation.level": "read_committed",  # skip aborted transactional records
})
```

## Kafka 4.x Timeline

| Release | Date | Headline |
|---------|------|----------|
| 4.0.0 | 2025-03-18 | ZooKeeper removed; KIP-848 consumer protocol GA (clients opt in with `group.protocol=consumer`); share groups early access; brokers/Connect/tools need Java 17, clients/Streams Java 11; KIP-896 removes protocol versions older than 2.1 |
| 4.1.0 | 2025-09-04 | Share groups preview (enable with `share.version=1`) |
| 4.2.0 | 2026-02-17 | Share groups production-ready (RENEW ack, lag metrics); Streams rebalance protocol KIP-1071 GA with a limited feature set |
| 4.3.0 | 2026-05-22 | KIP-1066 broker/log-dir cordoning; KIP-1240 extra share group configs; KIP-1023 follower fetch from tiered offset |

## KRaft: Dynamic Controller Quorum (3.9+)

```properties
# controller.properties: dynamic quorum (KIP-853). Static quorums use controller.quorum.voters
process.roles=controller
node.id=1
listeners=CONTROLLER://controller1.example.com:9093
controller.listener.names=CONTROLLER
controller.quorum.bootstrap.servers=controller1.example.com:9093,controller2.example.com:9093,controller3.example.com:9093
```

```bash
bin/kafka-storage.sh format --cluster-id <CLUSTER_ID> --standalone --config config/controller.properties
# start the other controllers (formatted with --no-initial-controllers), let them catch up, then:
bin/kafka-metadata-quorum.sh --command-config config/controller.properties \
  --bootstrap-controller controller1.example.com:9093 add-controller
```

A static quorum cannot be converted to a dynamic one; choose at format time.

## Migrating ZooKeeper Clusters (must happen on 3.9)

| Step | Action (Kafka 3.9 docs) |
|------|-------------------------|
| 1 | Upgrade brokers to 3.9.x (docs: software 3.9.1) with `inter.broker.protocol.version=3.9` |
| 2 | Provision KRaft controllers with `zookeeper.metadata.migration.enable=true` and `zookeeper.connect`; format with `kafka-storage.sh format --standalone --cluster-id=<ZK cluster id>` |
| 3 | Rolling-restart brokers with the migration flag plus `controller.quorum.bootstrap.servers`; metadata copies automatically |
| 4 | Restart brokers in KRaft mode (`process.roles=broker`, `node.id`), drop ZooKeeper configs |
| 5 | Finalize: remove the migration flag and ZooKeeper settings from controllers; no rollback after this |
| 6 | Upgrade the KRaft cluster to 4.x |

There is no `kafka-metadata.sh` migration step; `kafka-metadata-shell`/`kafka-metadata-quorum` are inspection tools.

## Tiered Storage (production-ready since 3.9)

```properties
# broker
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=<your RemoteStorageManager, e.g. an S3 plugin>
remote.log.storage.manager.class.path=/opt/kafka/plugins/rsm/*
remote.log.metadata.manager.listener.name=PLAINTEXT
# topic: remote.storage.enable=true, local.retention.ms=3600000, retention.ms=604800000
# Kafka ships no production RemoteStorageManager; compacted topics are not supported
```

## Share Groups (4.2+)

Check the feature level with `bin/kafka-features.sh --bootstrap-server broker1:9092 describe`; if `share.version`
is 0, enable with `... upgrade --feature share.version=1` (Confluent Platform 8.3 enables them by default).
Clusters with fewer than 3 brokers must first set `share.coordinator.state.topic.min.isr=1` (and replication factor 1).
Group configs (`kafka-configs.sh --entity-type groups`): `share.auto.offset.reset`, `share.record.lock.duration.ms`,
`share.isolation.level`, `share.session.timeout.ms`; 4.3 adds `share.delivery.count.limit`,
`share.partition.max.record.locks`, `share.renew.acknowledge.enable`. There is no `group.type` setting.

## Related

- [kafka-producer-consumer](../patterns/kafka-producer-consumer.md) · [cdc-patterns](../patterns/cdc-patterns.md) · [debezium-operations](../patterns/debezium-operations.md)

## Sources

- https://kafka.apache.org/community/downloads/ · https://docs.confluent.io/platform/current/config-manage/kafka-queues.html
- https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/ · https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/
- https://kafka.apache.org/blog/2026/05/22/apache-kafka-4.3.0-release-announcement/ · https://kafka.apache.org/blog/2024/11/06/apache-kafka-3.9.0-release-announcement/
- https://kafka.apache.org/39/operations/kraft/ · https://kafka.apache.org/40/operations/kraft/ · https://kafka.apache.org/43/operations/tiered-storage/
- https://kafka.apache.org/42/getting-started/upgrade/ · https://kafka.apache.org/42/configuration/group-configs/ · https://kafka.apache.org/43/configuration/group-configs/
- https://docs.redpanda.com/current/manage/tiered-storage/ · https://docs.redpanda.com/current/manage/schema-reg/schema-reg-overview/
