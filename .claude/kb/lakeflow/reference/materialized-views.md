# Materialized Views in Lakeflow Pipelines

> **Purpose**: How materialized views (MVs) refresh, what is incrementalizable, refresh policies, layout, access from outside the pipeline, standalone MVs
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. What an MV is

A materialized view is a pipeline object made of a defining query, a flow that updates it, and cached
results stored as a Unity Catalog table. It is always correct as of its last update (late and
out-of-order data included) and Databricks tries to update it incrementally. MVs are defined and
updated by one pipeline; other pipelines cannot change them. You query them like any table.

```python
from pyspark import pipelines as dp
from pyspark.sql import functions as F

@dp.materialized_view(
    comment="Revenue by region and category",
    cluster_by_auto=True,
    refresh_policy="auto",
)
def regional_sales():
    partners = spark.read.table("main.ref.partners")
    sales = spark.read.table("main.sales.orders")
    return (partners.join(sales, "partner_id")
            .groupBy("region", "category")
            .agg(F.sum("amount").alias("total_sales"),     # amount DECIMAL(18,2)
                 F.count("*").alias("orders")))
```

```sql
CREATE OR REFRESH MATERIALIZED VIEW regional_sales
CLUSTER BY AUTO
REFRESH POLICY AUTO
AS SELECT region, category, SUM(amount) AS total_sales, COUNT(*) AS orders
FROM main.ref.partners JOIN main.sales.orders USING (partner_id)
GROUP BY region, category;
```

## 2. Refresh semantics

| Fact | Detail |
|------|--------|
| Incremental refresh needs serverless | MVs in classic-compute pipelines are always fully recomputed |
| Default refresh | Cost model picks incremental or full; results identical either way |
| Full refresh | Clears table and checkpoints and reprocesses available source data (removed source records stay lost) |
| Standalone MVs | Refreshes run on a serverless pipeline automatically |
| Check what happened | Tables panel "Incrementalization" column; event log technique `FULL_RECOMPUTE`, `NO_OP`, `ROW_BASED`, `PARTITION_OVERWRITE`, `WINDOW_FUNCTION`, `APPEND_ONLY`, `GROUP_AGGREGATE`, `GENERIC_AGGREGATE` |
| Test a query | `EXPLAIN CREATE MATERIALIZED VIEW ...` |

### Refresh policy

| Policy (Python `refresh_policy` / SQL `REFRESH POLICY`) | Behavior |
|----------------------------------------------------------|----------|
| `auto` / `AUTO` (default) | Cost-based choice |
| `incremental` / `INCREMENTAL` | Incremental when possible, else full |
| `incremental_strict` / `INCREMENTAL STRICT` | Fail the refresh if it cannot be incremental |
| `full` / `FULL` | Always full |

There is no `refresh_strategy` table property; use the policy above.

## 3. What refreshes incrementally

| Construct | Incremental? |
|-----------|--------------|
| `SELECT` with deterministic built-ins and immutable UDFs | Yes (needs row tracking on sources) |
| `GROUP BY`, `WITH` (CTE) | Yes |
| `WITH RECURSIVE` | No (full recompute) |
| `UNION ALL` | Yes (row tracking) |
| `WHERE`, `HAVING` | Yes (row tracking) |
| `INNER`, `LEFT`, `RIGHT`, `FULL OUTER JOIN` | Yes (row tracking) |
| Window functions (`OVER`) | Yes, when `PARTITION BY` is specified |
| `QUALIFY` | Yes |
| Expectations | Yes, except when reading a view that has expectations, or with a `DROP` expectation plus `NOT NULL` columns |
| `current_date()`, `current_timestamp()`, `now()` | Only inside `WHERE` |
| Other non-deterministic functions | No |
| `SUM`/`AVG` over `FLOAT`/`DOUBLE` | May force full; cast to `DECIMAL` |
| Sources | Delta tables, MVs, STs (incl. AUTO CDC targets), UC managed Iceberg v2/v3. Not: volumes, external locations, foreign catalogs (foreign Iceberg readable but not incremental) |

Two-sided date filters (`>= :start AND < :end`) force a full refresh on each update; one-sided
(`>= :start`) stays incremental.

Recommended source table properties:

```sql
ALTER TABLE main.sales.orders SET TBLPROPERTIES (
  delta.enableDeletionVectors = true,
  delta.enableRowTracking = true,
  delta.enableChangeDataFeed = true);
```

Declaring primary/foreign keys with `RELY` lets the optimizer simplify MV plans (not enforced).

## 4. Layout and maintenance

- Use `cluster_by` / `cluster_by_auto` (SQL `CLUSTER BY` / `CLUSTER BY AUTO`). Databricks recommends
  liquid clustering instead of Z-order (`pipelines.autoOptimize.zOrderCols`).
- `pipelines.autoOptimize.managed` defaults to `true`; tables under predictive optimization ignore it.
- Do not set both partition and clustering columns (error).

## 5. Running refreshes

| How | Command |
|-----|---------|
| Normal update | `databricks bundle run <pipeline>` or a job `pipeline_task` |
| Full refresh of selected tables | `databricks bundle run <pipeline> --full-refresh mv_a,mv_b` |
| Full refresh of everything | `--full-refresh-all` (or `pipeline_task.full_refresh: true`) |
| Standalone MV (Databricks SQL) | `REFRESH MATERIALIZED VIEW mv_name FULL` |
| Block accidental full refresh | Table property `pipelines.reset.allowed = false` |

## 6. Access and governance

- Consumers in Databricks (SQL warehouses, standard access mode DBR 13.3 LTS+, jobs, other
  pipelines) read MVs with `SELECT` grants. What is not allowed is using an MV as a **streaming
  source** outside its pipeline.
- Non-Databricks engines: external data access (DBR 17.3+, `pipelines.externalMetadata.enabled =
  true`, clients on Delta 4.0+ or Iceberg v3, `EXTERNAL USE SCHEMA` + `SELECT`) or compatibility
  mode (Public Preview).
- No `CLONE`, no time travel; the Catalog Explorer History tab does not show MV history.
- A non-admin needs `REFRESH` on the MV plus pipeline permissions to view the backing pipeline.
- UDF behavior changes that Databricks cannot detect (for example an upgraded library) require a
  manual full refresh.

## 7. Standalone MVs

Outside a pipeline: `CREATE MATERIALIZED VIEW` in Databricks SQL, or `@dp.materialized_view` in a
serverless notebook (Beta; no `private`, no schedule, re-run with `full_refresh=True` for full).
Databricks creates a managed pipeline of type `MV/ST`. Standalone MVs accept `CONSTRAINT ...
EXPECT` (Jun 2026).

## 8. When not to use an MV

- Sources that do not retain history (Kafka) or ingest operations (Auto Loader): use streaming tables.
- Very large inputs where a full recompute is unaffordable: prefer streaming tables (exactly-once).
- Millisecond latency: MVs update in seconds to minutes.

## Sources

- https://docs.databricks.com/aws/en/ldp/concepts/materialized-views
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-materialized-view
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-materialized-view
- https://docs.databricks.com/aws/en/ldp/properties
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/external-access
- https://docs.databricks.com/aws/en/external-access/external-for-pipelines
- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
