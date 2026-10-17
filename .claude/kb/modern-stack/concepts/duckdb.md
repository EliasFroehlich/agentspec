# DuckDB

> **Purpose**: In-process columnar OLAP database: release lines (1.4 LTS, 1.5, 2.0 scheduled), core features by version, extensions, DuckLake
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

DuckDB is an in-process analytical database: a library linked into your process (Python, CLI, Java, Node, R, Rust, Wasm), with no server to run. It uses a columnar, vectorized engine and reads Parquet, CSV, JSON, Iceberg, Delta and DuckLake directly. It runs on a single node. MotherDuck is the separate hosted option.

## Release Lines (as of 2026-10-17)

| Line | Codename | Released | Status | End of life |
|------|----------|----------|--------|-------------|
| 1.5.x | Variegata | 1.5.0: 2026-03-09; latest **1.5.6: 2026-09-28** | Current stable | 2027-03-01 |
| 1.4.x LTS | Andium | 1.4.0: 2025-09-16; latest **1.4.5: 2026-06-17** | Long-term support | 2026-11-17 |
| 2.0.0 | — | **Scheduled for 2026-10-21** per the release calendar | Not released yet | — |

Every other minor release is an LTS release with one year of community support (from 1.4.0 on). The 1.4.0 announcement said "supported until 2026-09-16". The release calendar now shows EOL 2026-11-17, and the calendar is the current source. Plan an upgrade from 1.4.x before that date. Do not rely on 2.0 features until 2.0 ships. The planned 2.0 changes documented so far are listed under Common Mistakes.

## Features by Version

| Feature | Since | Notes |
|---------|-------|-------|
| Secrets manager (`CREATE SECRET`) | 0.10.0 | Replaces settings-based S3 credentials, which the docs now call legacy |
| Friendlier SQL (`FROM` first, `map['k']` returns the value) | 1.2.0 | `map[]` semantics changed in 1.2 |
| Python-style lambdas `lambda x: x + 1` | 1.3.0 | Arrow form `x -> x + 1` deprecated in 1.3 |
| Database encryption (AES-256, GCM default) | 1.4.0 | `ATTACH ... (ENCRYPTION_KEY '...')`; writes need OpenSSL from `httpfs` (1.4.1+); 1.4.2+ auto-loads it |
| `MERGE INTO` | 1.4.0 | Upserts without a primary key; supports `DELETE` and `RETURNING merge_action` |
| Iceberg writes through an attached REST catalog | 1.4.0 | `INSERT`/`COPY FROM DATABASE`; `UPDATE`/`DELETE` 1.4.2; `MERGE INTO`, `ALTER TABLE`, partitioned writes, v3 in 1.5.3 |
| `VARIANT` type (binary, typed per row) | 1.5.0 | Dot access `data.name`, `variant_extract`, `variant_typeof`; Parquet VARIANT read |
| `GEOMETRY` type in core | 1.5.0 | Moved out of `spatial`; optional CRS `GEOMETRY('OGC:CRS84')`; core has basic WKB/WKT functions, the full ST_ library stays in `spatial` |
| New CLI (themes, `.tables`, `_` = last result, pager) | 1.5.0 | — |
| DuckLake 1.0 (`ducklake` extension) | 1.5.2 | Spec v1.0 released 2026-04-13 |

## The Concept

```sql
-- Files are tables; reader functions auto-detect the format options
SELECT * FROM read_csv('data/customers.csv');
SELECT * FROM read_json('logs/*.json');
SELECT * FROM read_parquet('s3://my-bucket/events/**/*.parquet', hive_partitioning = true);

-- Credentials: use secrets (credential_chain validates AWS credentials at creation)
CREATE OR REPLACE SECRET s3_default (TYPE s3, PROVIDER credential_chain);

-- 1.4.0+: upsert without a primary key
MERGE INTO stock USING new_stock USING (item_id)
WHEN MATCHED THEN UPDATE SET balance = balance + volume
WHEN NOT MATCHED THEN INSERT VALUES (new_stock.item_id, new_stock.volume);

-- 1.4.0+: encrypted database file
ATTACH 'secure.duckdb' AS secure (ENCRYPTION_KEY 'use-a-secret-manager');

-- 1.5.0+: semi-structured data without a fixed schema
CREATE TABLE events (id INTEGER, data VARIANT);
SELECT data.name, variant_typeof(data) FROM events;
```

## Extensions

Core extensions install and load on first use. `UPDATE EXTENSIONS;` refreshes them between releases. This matters for `iceberg` and `ducklake`, which ship updates often.

| Extension | Use |
|-----------|-----|
| `httpfs` | HTTP(S)/S3 access; also supplies OpenSSL for encryption |
| `aws` / `azure` | Cloud credential providers (`credential_chain`), Azure Blob/ADLS |
| `iceberg` | `iceberg_scan` (read-only, path based); full read/write via `ATTACH ... (TYPE iceberg)` REST catalogs |
| `delta` | `delta_scan`; `ATTACH ... (TYPE delta)` supports blind appends (`INSERT INTO`) |
| `ducklake` | DuckLake lakehouse format (catalog in DuckDB/SQLite/PostgreSQL, data in Parquet) |
| `postgres` / `mysql` / `sqlite` | Attach and query external databases |
| `spatial` | Full ST_ function library, GDAL I/O (the type itself is in core since 1.5) |

## DuckLake in One Paragraph

DuckLake keeps table metadata in a SQL catalog database and data in Parquet files. The spec and the `ducklake` extension reached v1.0 on 2026-04-13. That release is "production-ready with guaranteed backward-compatibility" and needs DuckDB 1.5.2+. Catalog choices: DuckDB file (single client), SQLite (several local processes), PostgreSQL 12+ (multi-user, remote clients). MySQL 8+ works but is not recommended. Syntax: `ATTACH 'ducklake:<catalog>' AS lake (DATA_PATH '<storage>')`. See [DuckDB Patterns](../patterns/duckdb-patterns.md) for runnable SQL and [DuckLake (lakehouse KB)](../../lakehouse/concepts/ducklake.md) for the format itself.

## Common Mistakes

### Wrong

```sql
SELECT * FROM read_csv_auto('data/customers.csv');      -- old alias; docs only document read_csv
SET s3_access_key_id = '...';                           -- legacy settings; can leak credentials
SELECT list_transform([1, 2, 3], x -> x + 1);           -- arrow lambda: warns in 1.5, off by default in 2.0
SELECT * FROM iceberg_scan('s3://lake/db/orders', version => 123);  -- `version` = metadata version, not a snapshot
```

### Correct

```sql
SELECT * FROM read_csv('data/customers.csv');
CREATE SECRET (TYPE s3, PROVIDER credential_chain);
SELECT list_transform([1, 2, 3], lambda x: x + 1);
SELECT * FROM iceberg_scan('s3://lake/db/orders', snapshot_from_id = 7635660646343998149);
```

Planned for 2.0 (official docs): arrow lambdas disabled by default (re-enable with `SET lambda_syntax = 'ENABLE_SINGLE_ARROW'`; the flag goes away in 2.1). The `geometry_always_xy` axis-order warning becomes an error. Also, `date_trunc` on `DATE` already returns `TIMESTAMP` since 1.5.

## When to Use

- Ad-hoc analytics and prototyping on Parquet/CSV/JSON/Iceberg/Delta files without a warehouse
- CI data tests: fast startup, no credentials, same SQL locally and in the pipeline
- Embedded analytics inside Python services, notebooks, or BI tools
- Single-node workloads where a Spark cluster is overhead. Benchmark your own data before setting a size threshold

## Trade-offs

| Strength | Limitation |
|----------|------------|
| No infrastructure; `pip install duckdb` or `curl https://install.duckdb.org \| bash` | Single node; no distributed execution in core |
| Reads open formats directly; writes Iceberg (REST catalog) and DuckLake | Iceberg `UPDATE`/`DELETE` are merge-on-read only (positional deletes) |
| Free, MIT license, LTS line available | One read-write process per database file (many read-only processes); multi-process writes need the Quack remote protocol, beta as of 1.5.2 |
| Fast release cadence | Behavior changes between minors (e.g. `map[]` in 1.2, `date_trunc` in 1.5); pin versions |

## Related

- [DuckDB Patterns](../patterns/duckdb-patterns.md): files, secrets, Iceberg, DuckLake, MERGE, CI tests
- [Local-First Analytics](../patterns/local-first-analytics.md): DuckDB + Polars stacks
- [Polars](polars.md): Arrow interop with DuckDB
- [DuckLake](../../lakehouse/concepts/ducklake.md)

## Sources

- https://duckdb.org/release_calendar
- https://duckdb.org/2026/09/28/announcing-duckdb-156
- https://duckdb.org/2026/03/09/announcing-duckdb-150
- https://duckdb.org/2025/09/16/announcing-duckdb-140
- https://duckdb.org/2025/02/05/announcing-duckdb-120
- https://duckdb.org/docs/current/connect/concurrency.html
- https://duckdb.org/docs/current/sql/functions/geometry.html
- https://duckdb.org/2025/11/12/announcing-duckdb-142
- https://duckdb.org/2025/11/19/encryption-in-duckdb
- https://duckdb.org/2026/05/29/new-iceberg-features
- https://duckdb.org/docs/current/core_extensions/iceberg/overview.html
- https://duckdb.org/docs/current/core_extensions/delta
- https://duckdb.org/docs/current/core_extensions/httpfs/s3api_legacy_authentication
- https://duckdb.org/docs/current/sql/functions/lambda
- https://duckdb.org/docs/current/operations_manual/installing_duckdb/install_script.html
- https://ducklake.select/2026/04/13/ducklake-10/
- https://ducklake.select/docs/stable/duckdb/usage/choosing_a_catalog_database
