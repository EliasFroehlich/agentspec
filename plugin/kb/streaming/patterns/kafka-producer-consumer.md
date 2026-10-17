# Kafka Producer & Consumer

> **Purpose**: Idempotent producer, consumer groups (KIP-848 protocol), DLQ, share consumers (Kafka 4.2+)
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Production Kafka patterns for Python using confluent-kafka: idempotent producers (prevent duplicates), consumer groups with manual commit (at-least-once) on the KIP-848 `consumer` protocol, dead letter queues for poison messages, and share consumers for queue-style work distribution.

## The Pattern

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json, logging

logger = logging.getLogger(__name__)

# ============================================================
# Idempotent Producer (prevents duplicates on retry)
# ============================================================
producer_config = {
    "bootstrap.servers": "kafka:9092",
    "enable.idempotence": True,        # exactly-once per partition
    "acks": "all",                     # wait for all replicas
    "retries": 5,
    "max.in.flight.requests.per.connection": 5,
    "compression.type": "zstd",
    "linger.ms": 10,                   # batch for throughput
    "batch.size": 65536,
}

producer = Producer(producer_config)

def produce_event(topic: str, key: str, value: dict) -> None:
    """Produce a message with delivery callback."""
    def on_delivery(err, msg):
        if err:
            logger.error("Delivery failed: %s", err)
        else:
            logger.debug("Delivered to %s [%d] @ %d", msg.topic(), msg.partition(), msg.offset())

    producer.produce(
        topic=topic,
        key=key.encode("utf-8"),
        value=json.dumps(value).encode("utf-8"),
        on_delivery=on_delivery,
    )
    producer.poll(0)  # trigger callbacks

producer.flush()  # wait for all messages to be delivered

# ============================================================
# Consumer with manual commit + DLQ
# ============================================================
consumer_config = {
    "bootstrap.servers": "kafka:9092",
    "group.id": "order-processor",
    "group.protocol": "consumer",      # KIP-848 (broker 4.0+, librdkafka 2.12+ GA); default is "classic"
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,       # manual commit for at-least-once
    "max.poll.interval.ms": 300000,
    # with group.protocol=consumer do NOT set session.timeout.ms, heartbeat.interval.ms or
    # partition.assignment.strategy: they are broker/group configs (group.consumer.*) now
}

consumer = Consumer(consumer_config)
consumer.subscribe(["orders"])

dlq_producer = Producer({"bootstrap.servers": "kafka:9092"})

def process_messages(max_messages: int = 1000) -> None:
    """Consume with DLQ for poison messages."""
    processed = 0
    while processed < max_messages:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            logger.error("Consumer error: %s", msg.error())
            break

        try:
            event = json.loads(msg.value().decode("utf-8"))
            handle_order(event)  # business logic
            consumer.commit(message=msg, asynchronous=False)
            processed += 1
        except Exception as e:
            logger.error("Failed to process message: %s", e)
            # Route to Dead Letter Queue
            dlq_producer.produce(
                topic="orders-dlq",
                key=msg.key(),
                value=msg.value(),
                headers={"error": str(e).encode(), "original_topic": b"orders"},
            )
            consumer.commit(message=msg, asynchronous=False)

    consumer.close()
```

## Quick Reference

| Config | Value | Purpose |
|--------|-------|---------|
| `enable.idempotence` | `True` | Prevent duplicate produces on retry |
| `acks` | `all` | Durability: wait for all ISR replicas |
| `enable.auto.commit` | `False` | Manual commit for at-least-once |
| `auto.offset.reset` | `earliest` | Start from beginning for new groups |
| `compression.type` | `zstd` | Best ratio/speed for most workloads |
| `linger.ms` | `10-50` | Batch messages for throughput |
| `group.protocol` | `consumer` | KIP-848 incremental rebalances; GA in Kafka 4.0 brokers, opt-in on clients |
| `share.acknowledgement.mode` | `implicit` / `explicit` | Share consumers only (Java GA in 4.2; Python preview) |

## Share Consumer (Kafka 4.2+)

```java
// Java client: share groups are production-ready in Apache Kafka 4.2
Properties props = new Properties();
props.setProperty("bootstrap.servers", "kafka:9092");
props.setProperty("group.id", "order-workers");               // a share group
props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.setProperty("share.acknowledgement.mode", "explicit");

try (KafkaShareConsumer<String, String> consumer = new KafkaShareConsumer<>(props)) {
    consumer.subscribe(List.of("orders"));
    while (running) {
        for (ConsumerRecord<String, String> record : consumer.poll(Duration.ofMillis(1000))) {
            try {
                handleOrder(record.value());
                consumer.acknowledge(record, AcknowledgeType.ACCEPT);
            } catch (RetryableBusinessException e) {                     // your own exception type
                consumer.acknowledge(record, AcknowledgeType.RELEASE);   // redeliver later
            } catch (Exception e) {
                consumer.acknowledge(record, AcknowledgeType.REJECT);    // poison: stop redelivery
            }
        }
        consumer.commitSync();
    }
}
```

Python: `confluent_kafka.ShareConsumer` arrived in confluent-kafka-python 2.15.0 as a **Preview** (not for
production; no RENEW yet). Lock duration, delivery-count limit and offset reset are group configs set on
the broker (`kafka-configs.sh --entity-type groups`), not client properties.

## Related

- [kafka-fundamentals concept](../concepts/kafka-fundamentals.md)
- [cdc-patterns](cdc-patterns.md)
- [flink-sql-patterns](flink-sql-patterns.md)

## Sources

- https://kafka.apache.org/40/operations/consumer-rebalance-protocol/
- https://kafka.apache.org/43/generated/consumer_config.html
- https://kafka.apache.org/42/javadoc/org/apache/kafka/clients/consumer/KafkaShareConsumer.html
- https://kafka.apache.org/42/getting-started/upgrade/
- https://github.com/confluentinc/librdkafka/releases/tag/v2.12.0
- https://github.com/confluentinc/confluent-kafka-python/releases/tag/v2.15.0
- https://docs.confluent.io/kafka-clients/python/current/overview.html
