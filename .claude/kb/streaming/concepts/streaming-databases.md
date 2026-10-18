# Streaming Databases

> **Purpose**: RisingWave and Materialize: Postgres-compatible streaming SQL with incrementally maintained materialized views
> **Version Coverage**: RisingWave 3.1.0 (2026-09-21) / 3.0.5 (2026-10-12); Materialize Cloud and Self-Managed (v26.x)
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Streaming databases run SQL continuously over event streams and keep materialized views up to date as data arrives. You write Postgres-style SQL, query the views like tables, and the engine handles state, consistency and recovery. They trade the flexibility of Flink/Spark code for a much smaller operational surface.

## The Concept

```sql
-- RisingWave: Kafka source + incrementally maintained view
CREATE SOURCE kafka_events (
    event_id    VARCHAR,
    user_id     VARCHAR,
    event_type  VARCHAR,
    amount      DECIMAL,
    event_ts    TIMESTAMPTZ
) WITH (
    connector = 'kafka',
    topic = 'raw_events',
    properties.bootstrap.server = 'broker1:9092',
    scan.startup.mode = 'earliest'
) FORMAT PLAIN ENCODE JSON;

CREATE MATERIALIZED VIEW mv_revenue_per_minute AS
SELECT date_trunc('minute', event_ts) AS minute_window,
       event_type,
       COUNT(*)                AS event_count,
       SUM(amount)             AS total_revenue,
       COUNT(DISTINCT user_id) AS unique_users
FROM kafka_events
WHERE event_type = 'purchase'
GROUP BY 1, 2;

-- Reads are plain SQL over always-fresh results
SELECT * FROM mv_revenue_per_minute
WHERE minute_window > NOW() - INTERVAL '1 hour'
ORDER BY minute_window DESC;
```

## Quick Reference

| Feature | RisingWave | Materialize | Flink SQL |
|---------|-----------|-------------|-----------|
| SQL dialect | PostgreSQL | PostgreSQL | Flink SQL |
| Deployment | Cloud (Basic, Pro incl. BYOC) or self-managed (Apache 2.0) | Cloud or Self-Managed on Kubernetes (Helm/Terraform, AWS/Azure/GCP) | Self-hosted or managed services |
| Sources | Kafka, Postgres/MySQL/SQL Server CDC, S3, WebSocket (3.0+) | Kafka, Postgres/MySQL/SQL Server CDC, webhooks | Kafka, files, JDBC, CDC connectors |
| Sinks | Kafka, JDBC, Iceberg, HTTP (3.0+), others | Kafka, Iceberg | Kafka, JDBC, Iceberg, others |
| State | Hummock (LSM tree on object storage) | Differential dataflow | HashMap, RocksDB or ForSt (2.0+) |
| Best for | Streaming SQL apps, Iceberg-centric lakehouse | Strongly consistent views over complex joins | Full streaming programming model |

Materialize Self-Managed needs a license key since v26.0; the Community license is free up to 24 GiB memory / 48 GiB disk.

## Common Mistakes

### Wrong

```sql
-- Polling a regular table every second from application code
SELECT COUNT(*) FROM events WHERE event_ts > NOW() - INTERVAL '5 min';
```

### Correct

```sql
-- Let the streaming database maintain the result continuously
CREATE MATERIALIZED VIEW live_counts AS
SELECT COUNT(*) AS recent_events
FROM kafka_events
WHERE event_ts > NOW() - INTERVAL '5 minutes';   -- temporal filter keeps state bounded
```

## RisingWave Release Highlights

| Version | Date | Highlights |
|---------|------|------------|
| 2.5.0 | 2025-07-29 | `backfill_order` (technical preview), `openai_embedding`, automatic Iceberg maintenance |
| 2.6.0 | 2025-09-25 | `vector(n)` type; `openai_embedding` now takes one constant JSONB config |
| 3.0.0 | 2026-06-11 | Iceberg V3, native pgvector ingestion, WebSocket source, HTTP sink, exactly-once delivery, DataFusion default for batch queries, automatic MV reuse in batch queries |
| 3.1.0 | 2026-09-21 | `VARIANT` type, periodically refreshed materialized views, partitioned gap filling |

### Embeddings (2.6+ signature)

```sql
-- openai_embedding(config constant jsonb, input text) -> real[]
CREATE FUNCTION get_embedding(varchar) RETURNS REAL[] LANGUAGE sql AS $$
    SELECT openai_embedding('{"model": "text-embedding-3-small", "api_key": "<API_KEY>"}'::jsonb, $1)
$$;

CREATE MATERIALIZED VIEW product_embeddings AS
SELECT product_id, product_name, get_embedding(description) AS embedding
FROM products;
```

### Backfill Order Control (technical preview, MVs only)

```sql
-- Backfill dimension tables before the fact table to avoid join amplification
CREATE MATERIALIZED VIEW mv_orders_enriched
WITH (backfill_order = FIXED(dim_customer -> fact_orders, dim_product -> fact_orders))
AS SELECT f.order_id, c.segment, p.category, f.amount
FROM fact_orders f
JOIN dim_customer c ON f.customer_id = c.customer_id
JOIN dim_product  p ON f.product_id  = p.product_id;
-- source_rate_limit (in WITH) throttles backfill; it does not control order
```

### Iceberg Sink with Automatic Maintenance

```sql
CREATE SINK iceberg_orders FROM mv_orders_enriched
WITH (
    connector = 'iceberg',
    type = 'upsert',
    primary_key = 'order_id',
    warehouse.path = 's3://my-bucket/warehouse',
    database.name = 'sales',
    table.name = 'orders_enriched',
    catalog.type = 'rest',
    catalog.uri = 'http://rest-catalog:8181',
    enable_compaction = true,                  -- needs a dedicated Iceberg compactor node
    enable_snapshot_expiration = true,         -- default true
    snapshot_expiration_max_age_millis = 604800000
);
-- is_exactly_once defaults to true (requires sink decoupling, the default)
```

## Related

- [stream-processing-fundamentals](stream-processing-fundamentals.md)
- [flink-architecture](flink-architecture.md)
- [flink-sql-patterns](../patterns/flink-sql-patterns.md)

## Sources

- https://docs.risingwave.com/changelog/release-notes
- https://risingwave.com/blog/announcing-risingwave-3-0-the-real-time-data-platform-for-agentic-ai/
- https://docs.risingwave.com/sql/functions/ai
- https://docs.risingwave.com/sql/commands/sql-create-mv
- https://docs.risingwave.com/iceberg/deliver-to-iceberg · https://docs.risingwave.com/iceberg/maintenance
- https://risingwave.com/pricing/
- https://materialize.com/docs/self-managed-deployments/ · https://materialize.com/docs/license/
- https://materialize.com/changelog/2025-11-18-v26-release/ · https://materialize.com/docs/sql/create-source/ · https://materialize.com/docs/sql/create-sink/
