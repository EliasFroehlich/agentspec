# Local-First Analytics

> **Purpose**: Run single-node analytics with DuckDB + Polars + Parquet (optionally DuckLake) instead of a cluster or warehouse, with CI validation and a code-first reporting layer
> **Confidence**: MEDIUM
> **MCP Validated**: 2026-10-17

## When to Use

- Analytical workloads that fit one machine (RAM plus local disk spill), where a cluster or warehouse is overhead
- Teams that want fast local iteration without warehouse credentials
- CI pipelines that need zero-infrastructure data validation
- Reports versioned in git next to the SQL that feeds them

## Decision Framework

```text
Workload shape                                   Starting point
───────────────────────────────────────────────────────────────────────────
Ad-hoc / one script                              DuckDB in-process
Python transforms, larger-than-RAM files         Polars LazyFrame (2.0: streaming + spill by default)
Shared, versioned tables for a small team        DuckLake (SQLite or PostgreSQL catalog) + Parquet
Many concurrent writers / distributed compute    Spark, a warehouse, or a lakehouse platform
```

These are starting points, not thresholds. Benchmark representative queries on your own data before you decide.

## Cost Model

A local or CI run costs only the machine it runs on. A managed platform bills for compute and storage. Compare on your own workload with each vendor's current price list:

| Option | Billing model | Official pricing (check before quoting) |
|--------|---------------|------------------------------------------|
| BigQuery on-demand | Per TiB scanned: $6.25/TiB after the first free TiB per month (list price on the pricing page as of 2026-10-17) | https://cloud.google.com/bigquery/pricing |
| Snowflake | Credits per warehouse-second + storage | https://www.snowflake.com/en/pricing-options/ |
| Databricks | DBUs by compute type (Jobs, SQL, serverless) | https://www.databricks.com/product/pricing |
| DuckDB / Polars | Free (MIT) software; you pay for the host or CI runner | — |

Price a managed option with the compute type you would actually use (for example Databricks Jobs compute or serverless jobs), in your region, on the date you decide.

## Implementation

### Python Pipeline: DuckDB → Polars → Parquet

```python
import duckdb
import polars as pl
from pathlib import Path

con = duckdb.connect()
# S3 credentials via a secret; credential_chain needs AWS creds in the environment
con.sql("CREATE OR REPLACE SECRET s3_default (TYPE s3, PROVIDER credential_chain)")

# Extract: DuckDB reads raw files and hands a Polars DataFrame over Arrow
raw = con.sql("""
    SELECT *
    FROM read_parquet('s3://my-bucket/raw/events/**/*.parquet', hive_partitioning = true)
    WHERE event_date >= DATE '2026-03-01'
""").pl()

# Transform: Polars lazy pipeline (collect() uses the streaming engine in Polars 2.0)
result = (
    raw.lazy()
    .filter(pl.col("event_type").is_in(["purchase", "subscription"]))
    .with_columns(
        (pl.col("amount") * pl.col("quantity")).alias("revenue"),
        pl.col("event_date").dt.month().alias("month"),
    )
    .group_by("customer_id", "month")
    .agg(
        pl.col("revenue").sum().alias("total_revenue"),
        pl.col("event_type").n_unique().alias("event_types"),
    )
    .sort("total_revenue", descending=True)   # explicit order: 2.0 does not keep group order
    .collect()
)

# Load: write Parquet for the reporting layer
out = Path("output/marts")
out.mkdir(parents=True, exist_ok=True)
result.write_parquet(out / "customer_revenue.parquet", compression="zstd")
```

### Shared Layer: DuckLake Instead of Loose Files (DuckDB 1.5.2+)

```sql
INSTALL ducklake;
-- PostgreSQL catalog for several users; data files on object storage
ATTACH 'ducklake:postgres:dbname=ducklake_catalog host=localhost' AS lake
    (DATA_PATH 's3://my-bucket/lake/');
CREATE TABLE lake.customer_revenue AS
    SELECT * FROM 'output/marts/customer_revenue.parquet';
```

The catalog gives ACID commits, snapshots and time travel over the same Parquet files. See [duckdb-patterns](duckdb-patterns.md) for the catalog options.

### CI Pipeline: Data Validation Without a Warehouse

```yaml
# .github/workflows/data-quality.yml
name: Data Quality
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install DuckDB CLI (official install script)
        run: |
          curl https://install.duckdb.org | bash
          echo "$HOME/.duckdb/cli/latest" >> "$GITHUB_PATH"

      - name: Run data assertions
        run: duckdb -bail < tests/assertions.sql   # error() => non-zero exit
```

```sql
-- tests/assertions.sql: each check raises via error() when it fails
SELECT CASE WHEN count(*) > 0 THEN error('null order_id') END
FROM read_parquet('data/orders/*.parquet') WHERE order_id IS NULL;

SELECT CASE WHEN count(*) > 0 THEN error('orders without customer') END
FROM read_parquet('data/orders/*.parquet') AS o
ANTI JOIN read_parquet('data/customers/*.parquet') AS c USING (customer_id);

SELECT CASE WHEN count(*) > 0 THEN error('future order_date') END
FROM read_parquet('data/orders/*.parquet') WHERE order_date > current_date;

SELECT CASE WHEN count(*) > 0 THEN error('net_amount out of bounds') END
FROM read_parquet('data/orders/*.parquet') WHERE net_amount < 0 OR net_amount > 1000000;
```

### Replacing a Small Spark Job

```python
# Before (Spark): session startup + cluster for a single-node-sized job
# df = spark.read.parquet("data/events/"); df.groupBy("customer_id").agg(F.sum("amount"))

import duckdb

result = duckdb.sql("""
    SELECT customer_id, sum(amount) AS total
    FROM read_parquet('data/events/**/*.parquet', hive_partitioning = true)
    GROUP BY customer_id
    ORDER BY total DESC
""").pl()   # or .df() for pandas, .arrow() for Arrow
```

### Reporting Layer Options

| Tool | Fit for a DuckDB/Parquet stack |
|------|--------------------------------|
| Malloy (+ Publisher) | Semantic models over DuckDB files; serve them over REST/MCP |
| Observable Framework | Data loaders (Python/SQL) precompute snapshots; static hosting |
| Evidence Core | One warehouse per project via a direct connector; check DuckDB support before choosing it |

## Configuration

| Tool | Role | Install |
|------|------|---------|
| DuckDB | SQL engine on local/S3 files | `pip install duckdb` (LTS: `pip install duckdb==1.4.5`) or `curl https://install.duckdb.org \| bash` |
| Polars | DataFrame transforms | `pip install polars` (2.0 defaults differ from 1.x) |
| SQLMesh | Model framework, DuckDB engine | `pip install sqlmesh` then `sqlmesh init duckdb` |
| dbt + DuckDB | dbt models on DuckDB | dbt v2: adapter built in; dbt v1: `pip install dbt-duckdb` |

## Anti-Patterns

- **Hard-coded size thresholds** taken from blog posts. Measure your own queries and memory.
- **Several processes writing one `.duckdb` file.** One read-write process per file; share Parquet or use DuckLake.
- **CSV as the shared layer.** No schema, no compression, no predicate pushdown; use Parquet.
- **Skipping `hive_partitioning = true`** on directory-partitioned data. You lose partition pruning.
- **Asserting with "returns rows on failure".** Rows do not fail a CLI run; use `error()`.

## See Also

- [duckdb-patterns](../patterns/duckdb-patterns.md)
- [polars-patterns](../patterns/polars-patterns.md)
- [duckdb](../concepts/duckdb.md)
- [polars](../concepts/polars.md)
- [analytics-engineering](../concepts/analytics-engineering.md)
- [cost-optimization](../../cloud-platforms/patterns/cost-optimization.md)

## Sources

- https://duckdb.org/docs/current/operations_manual/installing_duckdb/install_script.html
- https://duckdb.org/docs/current/core_extensions/httpfs/s3api
- https://duckdb.org/docs/current/sql/functions/utility.html
- https://duckdb.org/docs/current/clients/cli/arguments.html
- https://duckdb.org/docs/current/connect/concurrency.html
- https://ducklake.select/docs/stable/duckdb/usage/choosing_a_catalog_database
- https://docs.pola.rs/releases/upgrade/2/
- https://docs.getdbt.com/docs/core/connect-data-platform/duckdb-setup
- https://docs.evidence.dev/migration-guide
- https://cloud.google.com/bigquery/pricing
- https://www.databricks.com/product/pricing
- https://www.snowflake.com/en/pricing-options/
