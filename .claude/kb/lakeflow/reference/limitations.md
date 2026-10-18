# Lakeflow Pipelines Limitations

> **Purpose**: Current limits and restrictions of Lakeflow pipelines (workspace, datasets, flows, Unity Catalog, serverless, sinks, external access) with workarounds
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Quotas

| Limit | Value |
|-------|-------|
| Concurrent pipeline updates per workspace | 1,000 |
| Datasets per pipeline | 1,000 (an update may run out of memory earlier) |
| Source files when the configuration lists individual files/notebooks | 100 |
| Source entries when the configuration includes folders | 50 entries; 1,000 files referenced in total |
| Updates per pipeline at a time | 1 (jobs cap concurrency to 1 for pipeline tasks) |
| Update history kept in UI/API | 60 days (event log keeps older events) |

Workaround for many sources: organize files into folders (`glob` with `/**`).

## 2. Dataset and flow rules

| Rule | Detail |
|------|--------|
| Define once | A dataset is the target of a single operation across all pipelines |
| Exception: append flows | Any number of append flows can write to one streaming table |
| Multiple AUTO CDC flows | Allowed into one streaming table when each flow has a unique `name` (2025.30+); an AUTO CDC target can only be targeted by other AUTO CDC flows |
| Identity columns | Not supported on AUTO CDC targets; may be recomputed in materialized views, so use them only in streaming tables |
| `pivot()` | Not supported (needs eager schema computation); use conditional aggregation |
| Time travel | Streaming tables only, not materialized views |
| `CLONE` | Materialized views cannot be the source or target of deep/shallow clone |
| Partitioning + clustering | Setting both on one table errors |
| Writes to other tables inside dataset code | Never call `save()`, `saveAsTable()`, `toTable()`, `start()`, `collect()`, `count()`, `toPandas()` |

```python
from pyspark import pipelines as dp

# Multiple append flows into one target are valid
dp.create_streaming_table("events")

@dp.append_flow(target="events", name="from_web")
def from_web():
    return spark.readStream.table("main.raw.web_events")

@dp.append_flow(target="events", name="from_app")
def from_app():
    return spark.readStream.table("main.raw.app_events")
```

```sql
-- PIVOT replacement
CREATE OR REFRESH MATERIALIZED VIEW sales_by_month AS
SELECT product_id,
       SUM(CASE WHEN month = 1 THEN amount END) AS jan,
       SUM(CASE WHEN month = 2 THEN amount END) AS feb
FROM sales GROUP BY product_id;
```

## 3. Cross-pipeline and outside access

| Statement | Correct reading |
|-----------|-----------------|
| Reading another pipeline's tables | Allowed: they are Unity Catalog tables |
| Writing another pipeline's tables | Not allowed: only the owning pipeline updates them (use move tables to change ownership) |
| Materialized view as a streaming source outside its pipeline | Not allowed (another pipeline or notebook cannot stream from it) |
| Querying MVs/STs from Databricks SQL, notebooks, jobs, other pipelines | Allowed with `SELECT` grants |
| Access from non-Databricks clients | Not by default. Options: external data access (no copy; clients on Delta 4.0+ or Iceberg v3 via UC / Iceberg REST APIs) or compatibility mode (Public Preview; read-only copy, scheduled refresh, any Delta/Iceberg client) |
| Iceberg | MV/ST exposure uses IcebergCompatV3 through external data access; IcebergCompatV2 is not supported for MV/ST |

## 4. Unity Catalog pipelines

- Compute must use standard access mode (dedicated is not supported for running UC pipelines).
- JARs are not supported; only third-party Python libraries (use the pipeline `environment`).
- DML that modifies the schema of a streaming table is not supported. `ALTER TABLE` is disallowed
  on streaming tables; use `CREATE OR REFRESH` or `ALTER STREAMING TABLE`. Column-level ALTER on
  streaming tables became metadata-only in Aug 2026 (Beta, needs external metadata enabled).
- `LOCATION` is not supported; tables use the schema/catalog managed storage.
- Unity Catalog UDFs cannot be created inside a pipeline (existing UC UDFs can be called; Python UDF
  support is Public Preview).
- Hive metastore pipelines cannot be upgraded in place; clone into a new UC pipeline and re-ingest.
- A UC pipeline cannot publish to the Hive metastore; global init scripts are not supported.
- Run-as identity: user, service principal, or an account group assigned to the workspace
  (not workspace-local groups, `admins`, `users`, or `account users`).
- The Catalog Explorer History tab does not show history for materialized views.
- Backing files of an MV can contain upstream values not visible in its schema; do not share the
  underlying storage with untrusted consumers.

## 5. Serverless pipelines

- Require Unity Catalog, the `ADVANCED` edition, accepted serverless terms and a supported region.
- No `clusters` block (error if present); the `configuration` map (Spark conf and parameters)
  still works.
- `dbutils.library.restartPython()` is not supported; declare dependencies in `environment`.
- AWS PrivateLink needs your Databricks account team.
- Structured Streaming trigger limitations of serverless compute do not apply to pipeline modes;
  serverless pipelines support triggered, continuous and real-time modes.

## 6. Sinks and ForEachBatch

- Python only; streaming queries only; written by `append_flow` or `update_flow` only.
- No expectations; a full refresh does not clean the sink (reprocessed data is appended).
- Delta sinks need fully qualified names (`<catalog>.<schema>.<table>`).
- A sink cannot be read back as a pipeline dataset.
- ForEachBatch: you own downstream cleanup on full refresh; functions must be serializable for
  Databricks Connect (no `dbutils` inside).

## 7. Expectations

- Supported on streaming tables, materialized views and temporary views only.
- Not supported on sinks or `AUTO CDC FROM SNAPSHOT` flows.
- No subqueries on other tables, Python UDFs or external calls in constraints.
- `fail` records no metrics.

## Summary

| Area | Limitation | Workaround |
|------|-----------|------------|
| Concurrent updates | 1,000 per workspace | Spread schedules; use triggered mode |
| Datasets | 1,000 per pipeline | Split pipelines; move tables between them |
| Source files | 100 files or 50 folder entries | Use folders and `glob` |
| Same target | One defining operation | Append flows / named AUTO CDC flows |
| Identity columns | Not with AUTO CDC | Surrogate keys in a downstream MV or `xxhash64` of business keys |
| External engines | Not by default | External data access or compatibility mode |
| Time travel | STs only | Snapshot an MV into a regular Delta table if history is needed |
| PIVOT | Not supported | Conditional aggregation |
| JAR | Not in UC | Python libraries |

## Sources

- https://docs.databricks.com/aws/en/ldp/limitations
- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/concepts/flows
- https://docs.databricks.com/aws/en/ldp/concepts/materialized-views
- https://docs.databricks.com/aws/en/ldp/external-access
- https://docs.databricks.com/aws/en/external-access/external-for-pipelines
- https://docs.databricks.com/aws/en/delta/iceberg-reads
- https://docs.databricks.com/aws/en/ldp/serverless
- https://docs.databricks.com/aws/en/ldp/ldp-sinks
- https://docs.databricks.com/aws/en/ldp/for-each-batch
- https://docs.databricks.com/aws/en/ldp/expectations
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/developer/python-ref
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-streaming-table
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/30/
