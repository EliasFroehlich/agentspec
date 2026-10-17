# Flink CDC Pipelines

> **Purpose**: Database-to-lake CDC without Kafka using Flink CDC 3.x: YAML pipelines (route, transform, schema evolution) and the SQL `mysql-cdc` connector
> **Version Coverage**: Flink CDC 3.7.0 (quickstarts for Flink 1.20 and Flink 2.2)
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Sync whole databases (many tables, regex selection) into Iceberg, Paimon, Fluss, Doris, StarRocks or Kafka
- Upstream DDL (new tables, added or renamed columns) must flow downstream automatically
- You want per-table routing, projection and filtering without writing a Flink job
- Use the SQL connector instead when CDC is one input of a larger Flink SQL job

## Implementation

```yaml
# mysql-to-iceberg.yaml: whole-database sync with routing, transform and schema evolution
source:
  type: mysql
  name: MySQL Source
  hostname: mysql-primary
  port: 3306
  username: flink_cdc
  password: <from-secret-store>        # template it in at deploy time; never commit it
  tables: shop.\.*                     # regex over <db>.<table>
  server-id: 5401-5404                 # one unique id per parallel reader
  server-time-zone: UTC

sink:
  type: iceberg
  name: Iceberg Sink
  catalog.properties.type: glue
  catalog.properties.warehouse: s3://my-bucket/warehouse
  catalog.properties.io-impl: org.apache.iceberg.aws.s3.S3FileIO
  catalog.properties.client.region: us-east-1

transform:
  - source-table: shop.orders
    projection: order_id, customer_id, amount, status, updated_at
    filter: amount > 0
    description: drop zero-amount test orders

route:
  - source-table: shop.\.*
    sink-table: ods_shop.<>
    replace-symbol: <>
    description: shop.X -> ods_shop.X

pipeline:
  name: MySQL to Iceberg
  parallelism: 4
  schema.change.behavior: lenient      # default; also exception | evolve | try_evolve | ignore
```

```bash
# Submit to a running Flink cluster with the Flink CDC CLI
bash bin/flink-cdc.sh mysql-to-iceberg.yaml
# Some catalogs need extra jars (e.g. iceberg-flink-runtime, iceberg-aws): pass them with --jar
```

```sql
-- SQL connector alternative: one table, used inside a larger Flink SQL job
CREATE TABLE mysql_orders (
    order_id     BIGINT,
    customer_id  BIGINT,
    amount       DECIMAL(10, 2),
    status       STRING,
    updated_at   TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'        = 'mysql-cdc',
    'hostname'         = 'mysql-primary',
    'port'             = '3306',
    'username'         = 'flink_cdc',
    'password'         = '<from-secret-store>',
    'database-name'    = 'shop',
    'table-name'       = 'orders',
    'server-id'        = '5405-5408',
    'server-time-zone' = 'UTC'
    -- incremental snapshot reading is on by default (parallel, lock-free; 'scan.incremental.snapshot.*' options)
);

CREATE TABLE iceberg_orders (
    order_id BIGINT, customer_id BIGINT, amount DECIMAL(10, 2),
    status STRING, updated_at TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector'            = 'iceberg',
    'catalog-name'         = 'lake',
    'catalog-type'         = 'rest',
    'uri'                  = 'http://rest-catalog:8181',
    'format-version'       = '2',              -- upsert needs Iceberg v2
    'write.upsert.enabled' = 'true'
);

INSERT INTO iceberg_orders SELECT * FROM mysql_orders;   -- continuous streaming job
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `pipeline.schema.change.behavior` | `lenient` | `exception`, `evolve`, `try_evolve`, `lenient`, `ignore` |
| `pipeline.route-mode` | `ALL_MATCH` | `FIRST_MATCH` applies only the first matching route |
| `pipeline.parallelism` | `1` | Global pipeline parallelism |
| `pipeline.execution.runtime-mode` | `STREAMING` | `BATCH` for one-off loads |
| `source.server-id` | random 5400-6400 | Set an explicit range at least as large as the parallelism |
| `source.scan.incremental.snapshot.chunk.size` | `8096` | Rows per snapshot chunk |
| `sink.existing-table.schema-expansion.mode` | `DISABLED` | Widen an existing target table on the first `CreateTableEvent` |

## Example Usage

| Behavior | What happens to upstream DDL |
|----------|------------------------------|
| `exception` | Any schema change fails the job |
| `evolve` | Applied downstream; failure triggers a global failover |
| `try_evolve` | Applied when supported; unsupported changes tolerated, data converted (may be lossy) |
| `lenient` | Converted to safe changes (type change becomes rename + add column); truncate/drop not forwarded |
| `ignore` | Swallowed; only unchanged columns keep flowing |

## See Also

- [cdc-patterns](cdc-patterns.md)
- [debezium-operations](debezium-operations.md)
- [flink-sql-patterns](flink-sql-patterns.md)
- [flink-architecture](../concepts/flink-architecture.md)

## Sources

- https://flink.apache.org/downloads/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/core-concept/data-pipeline/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/core-concept/schema-evolution/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/core-concept/route/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/core-concept/transform/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/connectors/pipeline-connectors/iceberg/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/connectors/pipeline-connectors/mysql/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/connectors/flink-sources/mysql-cdc/
- https://nightlies.apache.org/flink/flink-cdc-docs-release-3.7/docs/get-started/quickstart-for-2.2/mysql-to-kafka/
- https://iceberg.apache.org/docs/latest/flink-connector/ · https://iceberg.apache.org/docs/latest/flink-writes/
