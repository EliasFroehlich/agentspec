# Flink Architecture

> **Purpose**: JobManager/TaskManager, checkpointing, state backends (HashMap, RocksDB, ForSt), Flink SQL, Flink 2.x changes
> **Version Coverage**: Apache Flink 2.3.0 (2026-06-25); 2.2.1, 2.1.3, 2.0.2 maintained
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Apache Flink is a distributed stream processing engine with a master-worker architecture. The **JobManager** coordinates scheduling, checkpointing and recovery; **TaskManagers** run the operators. Exactly-once state consistency comes from distributed snapshots (Chandy-Lamport style checkpoint barriers), with state kept in a configurable state backend.

## The Concept

```yaml
# JobManager: Dispatcher (accepts jobs) | ResourceManager (slots) | JobMaster (one per job)
# TaskManager x N: task slots run operator subtasks; keyed state lives in the state backend
# Checkpoint: barriers flow with the data -> each operator snapshots state -> durable storage
#
# conf/config.yaml (Flink 2.0+ reads only config.yaml; flink-conf.yaml is no longer supported)
execution.checkpointing.interval: 60s
execution.checkpointing.mode: EXACTLY_ONCE
execution.checkpointing.min-pause: 30s
execution.checkpointing.incremental: true
execution.checkpointing.dir: s3://my-bucket/flink/chk
execution.checkpointing.savepoint-dir: s3://my-bucket/flink/savepoints
state.backend.type: rocksdb          # hashmap (default) | rocksdb | forst
state.backend.rocksdb.memory.managed: true
```

## Quick Reference

| Component | Role | Failure impact |
|-----------|------|----------------|
| JobManager | Coordination, scheduling, checkpoints | Job restarts from the last checkpoint |
| TaskManager | Data processing, state access | Lost slots rescheduled on other TMs |
| Checkpoint | Automatic consistent snapshot | Recovery point |
| Savepoint | User-triggered, portable snapshot | Upgrades, rescaling, migration |

| `state.backend.type` | Best for | Notes |
|----------------------|----------|-------|
| `hashmap` (default) | Small state, lowest latency | State on the JVM heap |
| `rocksdb` | Large state (TB) on local disk | Incremental checkpoints |
| `forst` (2.0+) | Disaggregated state on remote DFS | Async state access; docs mark it experimental |

## Common Mistakes

### Wrong

```sql
-- No checkpointing and no state TTL: state lost on failure and grows forever
SELECT user_id, COUNT(*) AS total_clicks FROM clicks GROUP BY user_id;
```

```yaml
# Legacy key names: the 2.x configuration reference does not list them
state.checkpoints.dir: s3://my-bucket/flink/chk
state.savepoints.dir: s3://my-bucket/flink/savepoints
state.backend.incremental: true
```

### Correct

```sql
SET 'execution.checkpointing.interval' = '60s';
SET 'table.exec.state.ttl'             = '86400000';  -- 24 h in ms
SELECT user_id, COUNT(*) AS total_clicks FROM clicks GROUP BY user_id;
```

```yaml
execution.checkpointing.dir: s3://my-bucket/flink/chk
execution.checkpointing.savepoint-dir: s3://my-bucket/flink/savepoints
execution.checkpointing.incremental: true
```

## Flink 2.0: Disaggregated State (ForSt)

```yaml
# SQL jobs (docs quick start); disaggregated state is still experimental in 2.3
state.backend.type: forst
table.exec.async-state.enabled: true
execution.checkpointing.incremental: true
execution.checkpointing.dir: s3://my-bucket/flink/chk     # required
state.backend.forst.primary-dir: s3://my-bucket/forst-state
table.exec.mini-batch.enabled: false                      # not supported with async state yet
table.optimizer.agg-phase-strategy: ONE_PHASE
```

```java
// DataStream: enable async state on the keyed stream, then use the async state API
KeyedStream<Event, String> keyed = events.keyBy(Event::sourceAddress).enableAsyncState();
// inside the function: state.asyncValue().thenAccept(v -> state.asyncUpdate(next(v)));
```

ForSt suits very large state and fast rescaling; for small state, local state with synchronous access is the better choice.

### Removed in Flink 2.0

| Removed | Replacement |
|---------|-------------|
| `SourceFunction` / `SinkFunction` | Source V2 / Sink V2 |
| Scala DataStream API, DataSet API | Java DataStream API, Table API/SQL |
| `TableSchema`, `TableColumn` | `Schema`, `Column`, `DataTypes` |
| `TableSource` / `TableSink` | `DynamicTableSource` / `DynamicTableSink` |
| `flink-conf.yaml` | `config.yaml` (standard YAML) |

## Flink 2.1 to 2.3

| Release | Date | Highlights |
|---------|------|------------|
| 1.20 | 2024-08-02 | Materialized tables introduced (FLINK-35187), not a 2.0 feature |
| 2.1.0 | 2025-07-31 | `ML_PREDICT` table function for model inference; delta join |
| 2.2.0 | 2025-12-04 | `VECTOR_SEARCH` (used as `LATERAL VECTOR_SEARCH(...)`); model inference in Table API; `FRESHNESS` optional in materialized tables |
| 2.3.0 | 2026-06-25 | `FROM_CHANGELOG` / `TO_CHANGELOG` process table functions; `INSERT ... ON CONFLICT`; experimental `flink-s3-fs-native`; first-class application concept |

```sql
-- Flink 2.1+: model must be registered in the catalog; input must be append-only
SELECT * FROM ML_PREDICT(TABLE support_tickets, MODEL ticket_classifier, DESCRIPTOR(subject, body));
```

## Related

- [stream-processing-fundamentals](../concepts/stream-processing-fundamentals.md)
- [flink-sql-patterns](../patterns/flink-sql-patterns.md)
- [flink-cdc-pipelines](../patterns/flink-cdc-pipelines.md)

## Sources

- https://flink.apache.org/downloads/
- https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/config/
- https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/disaggregated_state/
- https://nightlies.apache.org/flink/flink-docs-release-2.3/docs/dev/datastream/fault-tolerance/state_v2/
- https://nightlies.apache.org/flink/flink-docs-stable/release-notes/flink-2.0/
- https://nightlies.apache.org/flink/flink-docs-release-1.20/release-notes/flink-1.20/
- https://flink.apache.org/2025/07/31/apache-flink-2.1.0-ushers-in-a-new-era-of-unified-real-time-data--ai-with-comprehensive-upgrades/
- https://flink.apache.org/2025/12/04/apache-flink-2.2.0-advancing-real-time-data--ai-and-empowering-stream-processing-for-the-ai-era/
- https://flink.apache.org/2026/06/25/apache-flink-2.3.0-release-announcement/
- https://nightlies.apache.org/flink/flink-docs-stable/docs/sql/reference/queries/model-inference/
