# DuckDB Patterns

> **Purpose**: Runnable DuckDB SQL recipes: files, secrets for S3, Iceberg (read, time travel, REST catalog writes), DuckLake, MERGE INTO, encryption, external databases, CI assertions, export
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## When to Use

- Querying Parquet/CSV/JSON files locally or on object storage without a warehouse
- Reading Iceberg tables, or writing them through an Iceberg REST catalog
- A small lakehouse on DuckLake (catalog in DuckDB/SQLite/PostgreSQL, data in Parquet)
- Fast, credential-free data assertions in CI
- Federated queries across Postgres/SQLite and files

Versions: examples run on DuckDB 1.5.x (current stable, 1.5.6). Lines marked "1.4.0+" also run on the 1.4 LTS line. DuckLake needs 1.5.2+.

## Implementation

```sql
-- ============================================================
-- Files (format options are auto-detected)
-- ============================================================
SELECT * FROM 'data/orders.parquet' LIMIT 10;               -- file path as a table
SELECT count(*) FROM 'data/orders/*.parquet';               -- glob
SELECT * FROM read_csv('data/customers.csv');
SELECT * FROM read_json('data/events/*.json');
SELECT * FROM read_parquet('data/events/**/*.parquet', hive_partitioning = true)
WHERE year = 2026 AND month = 3;                            -- partition pruning
SELECT * FROM read_csv('data/sales_*.csv', union_by_name = true);

-- ============================================================
-- S3 / object storage: secrets (0.10.0+)
-- ============================================================
CREATE OR REPLACE SECRET s3_default (
    TYPE s3,
    PROVIDER credential_chain      -- env vars, profile, instance role; validated at creation
);
-- Or explicit keys (keep them out of source control):
-- CREATE OR REPLACE SECRET s3_keys (TYPE s3, PROVIDER config,
--     KEY_ID 'AKIA...', SECRET '...', REGION 'us-east-1');

SELECT * FROM 's3://my-bucket/warehouse/orders/*.parquet'
WHERE order_date >= DATE '2026-03-01';

-- ============================================================
-- Iceberg, path based (read-only)
-- ============================================================
SELECT count(*) FROM iceberg_scan('s3://my-bucket/lake/db/orders');
SELECT * FROM iceberg_snapshots('s3://my-bucket/lake/db/orders');
-- Time travel: snapshot_from_id OR snapshot_from_timestamp (mutually exclusive)
SELECT count(*) FROM iceberg_scan('s3://my-bucket/lake/db/orders',
    snapshot_from_id = 7635660646343998149);
SELECT count(*) FROM iceberg_scan('s3://my-bucket/lake/db/orders',
    snapshot_from_timestamp = TIMESTAMP '2026-03-01 00:00:00');
-- `version = '...'` selects a metadata file version; it is NOT time travel

-- ============================================================
-- Iceberg REST catalog: read + write (writes 1.4.0+)
-- ============================================================
CREATE SECRET iceberg_secret (
    TYPE iceberg,
    CLIENT_ID 'my-client-id',
    CLIENT_SECRET 'my-client-secret',
    OAUTH2_SERVER_URI 'https://catalog.example.com/v1/oauth/tokens'
);
ATTACH 'warehouse' AS ice (
    TYPE iceberg,
    SECRET iceberg_secret,
    ENDPOINT 'https://catalog.example.com'
);
INSERT INTO ice.sales.events VALUES (1, 'click', now());               -- 1.4.0+
UPDATE ice.sales.events SET event_name = 'view' WHERE id = 1;           -- 1.4.2+, merge-on-read
SELECT * FROM ice.sales.events AT (VERSION => 7635660646343998149);     -- snapshot id
SELECT * FROM ice.sales.events AT (TIMESTAMP => TIMESTAMP '2026-03-01 00:00:00');
COPY FROM DATABASE ice TO local_db;                                      -- bulk copy Iceberg -> DuckDB

-- ============================================================
-- DuckLake (1.5.2+, spec v1.0): catalog DB + Parquet data files
-- ============================================================
INSTALL ducklake;
-- Single client: catalog in a DuckDB file
ATTACH 'ducklake:metadata.ducklake' AS lake (DATA_PATH 'lake_files/');
-- Several local processes: SQLite catalog
-- ATTACH 'ducklake:sqlite:metadata.sqlite' AS lake (DATA_PATH 'lake_files/');
-- Multi-user / remote clients: PostgreSQL 12+ catalog, data on S3
-- ATTACH 'ducklake:postgres:dbname=ducklake_catalog host=localhost' AS lake
--     (DATA_PATH 's3://my-bucket/lake/');
USE lake;
CREATE TABLE orders AS SELECT * FROM read_parquet('data/orders/*.parquet');
UPDATE orders SET status = 'shipped' WHERE order_id = 42;
SELECT snapshot_id FROM lake.snapshots();
SELECT count(*) FROM orders AT (VERSION => 1);                          -- time travel

-- ============================================================
-- MERGE INTO (1.4.0+): upsert without a primary key
-- ============================================================
WITH new_stock(item_id, volume) AS (VALUES (20, 2200), (30, 1900))
MERGE INTO stock
    USING new_stock USING (item_id)
    WHEN MATCHED THEN UPDATE SET balance = balance + volume
    WHEN NOT MATCHED THEN INSERT VALUES (new_stock.item_id, new_stock.volume)
    RETURNING merge_action, *;

-- ============================================================
-- Encrypted database file (1.4.0+, AES-256-GCM)
-- ============================================================
ATTACH 'secure.duckdb' AS secure (ENCRYPTION_KEY 'load-from-a-secret-store');
CREATE TABLE secure.customers AS SELECT * FROM 'data/customers.parquet';

-- ============================================================
-- External databases
-- ============================================================
ATTACH 'host=localhost dbname=app_db' AS pg (TYPE postgres, READ_ONLY);
ATTACH 'legacy.db' AS lite (TYPE sqlite);
SELECT u.name, o.total
FROM pg.public.users AS u
JOIN 'data/orders.parquet' AS o ON u.id = o.user_id;

-- ============================================================
-- CI assertions: error() fails the statement, so the run fails
-- ============================================================
SELECT CASE WHEN count(*) > 0 THEN error('null order_id found') END
FROM 'tests/fixtures/orders.parquet' WHERE order_id IS NULL;

SELECT CASE WHEN count(*) <> count(DISTINCT order_id)
            THEN error('duplicate order_id') END
FROM 'tests/fixtures/orders.parquet';

-- ============================================================
-- Export
-- ============================================================
COPY (SELECT * FROM 'data/orders.parquet' WHERE year = 2026)
TO 'output/orders_2026.parquet' (FORMAT parquet, COMPRESSION zstd);
COPY (FROM 'data/orders.parquet') TO 'output/orders.csv' (HEADER, DELIMITER ',');
```

## Configuration

| Item | Value / Default | Notes |
|------|-----------------|-------|
| Extension autoload | on | `INSTALL x; LOAD x;` only when autoload is disabled |
| `UPDATE EXTENSIONS;` | — | Run after upgrades; `iceberg`/`ducklake` change between releases |
| S3 auth | `CREATE SECRET (TYPE s3, ...)` | Settings-based credentials are the legacy scheme (risk of leaking secrets) |
| Iceberg writes | attached REST catalog only | `iceberg_scan` is read-only; `UPDATE`/`DELETE` are merge-on-read |
| DuckLake catalog | DuckDB / SQLite / PostgreSQL | MySQL 8+ works but is not recommended |
| Encryption writes | OpenSSL via `httpfs` | 1.4.1 blocked Mbed TLS writes; 1.4.2+ auto-loads `httpfs` |
| CLI `-bail` | off | Stop at the first error when piping a SQL file |

## Example Usage

```bash
# CI: install the CLI (official script) and run assertions
curl https://install.duckdb.org | bash
~/.duckdb/cli/latest/duckdb -bail < tests/assertions.sql   # non-zero exit when error() fires

# Pin a version (e.g. the LTS line)
curl https://install.duckdb.org | DUCKDB_VERSION=1.4.5 bash
pip install duckdb==1.4.5
```

## See Also

- [duckdb](../concepts/duckdb.md)
- [local-first-analytics](../patterns/local-first-analytics.md)
- [polars-patterns](../patterns/polars-patterns.md)
- [DuckLake concept (lakehouse KB)](../../lakehouse/concepts/ducklake.md)

## Sources

- https://duckdb.org/docs/current/data/csv/overview.html
- https://duckdb.org/docs/current/core_extensions/httpfs/s3api
- https://duckdb.org/docs/current/core_extensions/httpfs/s3api_legacy_authentication
- https://duckdb.org/docs/current/core_extensions/iceberg/overview.html
- https://duckdb.org/docs/current/core_extensions/iceberg/writing
- https://duckdb.org/2025/11/28/iceberg-writes-in-duckdb
- https://duckdb.org/2025/09/16/announcing-duckdb-140
- https://duckdb.org/2025/11/19/encryption-in-duckdb
- https://duckdb.org/docs/current/sql/functions/utility.html
- https://duckdb.org/docs/current/clients/cli/arguments.html
- https://duckdb.org/docs/current/operations_manual/installing_duckdb/install_script.html
- https://ducklake.select/docs/stable/duckdb/introduction
- https://ducklake.select/docs/stable/duckdb/usage/connecting
- https://ducklake.select/docs/stable/duckdb/usage/choosing_a_catalog_database
