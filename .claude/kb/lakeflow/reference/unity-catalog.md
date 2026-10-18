# Unity Catalog with Lakeflow Pipelines

> **Purpose**: Publishing targets (default publishing mode), permissions, compute requirements, sources, governance (grants, row filters, column masks), DML, inactive tables and limitations
> **Confidence**: High
> **MCP Validated**: 2026-10-17

---

## 1. Publishing model

Unity Catalog is the default for new pipelines. In the **default publishing mode** (all pipelines
created since 2025-02-05; older ones may use the legacy `LIVE` mode):

- `catalog` + `schema` in the settings are only the **defaults** for unqualified names.
- Any dataset can be written to another catalog/schema with a fully qualified name, or after
  `USE CATALOG` / `USE SCHEMA` in a source file (scope: rest of that file).
- The `LIVE` keyword is ignored.
- Writing to a schema that does not exist creates it (needs `CREATE SCHEMA`); reading a missing
  dataset fails the update.

```json
{ "name": "sales_pipeline", "catalog": "main", "schema": "sales", "serverless": true }
```

```python
from pyspark import pipelines as dp

@dp.materialized_view(name="main.reporting.regional_sales")   # outside the default schema
def regional_sales():
    return spark.read.table("partners")                          # resolves to main.sales.partners
```

```sql
CREATE OR REFRESH MATERIALIZED VIEW main.reporting.regional_sales AS SELECT * FROM partners;
USE CATALOG main;
USE SCHEMA reporting;
```

There is no `unity_catalog_enabled` flag: setting `catalog` enables UC. The legacy `target` field
held only a schema name; use `schema`.

## 2. Permissions

| To | Privileges (for the run-as identity) |
|----|--------------------------------------|
| Publish streaming tables | `USE CATALOG` on the catalog; `USE SCHEMA` + `CREATE TABLE` on the schema |
| Publish materialized views | `USE CATALOG`; `USE SCHEMA` + `CREATE MATERIALIZED VIEW` |
| Create schemas | `USE CATALOG` + `CREATE SCHEMA` |
| Update existing tables | `MODIFY` |
| Let others read | `GRANT SELECT ON TABLE <cat>.<sch>.<t> TO <principal>` |

```sql
GRANT USE CATALOG ON CATALOG main TO `etl-service-principal`;
GRANT USE SCHEMA, CREATE TABLE, CREATE MATERIALIZED VIEW ON SCHEMA main.sales TO `etl-service-principal`;
GRANT SELECT ON TABLE main.sales.daily_revenue TO `analysts`;
```

By default only the pipeline owner can query the pipeline's datasets; `MANAGE` is propagated to
MVs/STs (Jan 2026). Ownership of MVs/STs can be changed to a service principal with
`ALTER MATERIALIZED VIEW | STREAMING TABLE ... SET OWNER TO` (May 2026); a group owner must be set
in the UI. Changing the run-as identity also changes the owner of published tables.

## 3. Compute

| Purpose | Requirement |
|---------|-------------|
| Run a UC pipeline | Standard access mode (dedicated is not supported) |
| Query its MVs/STs | SQL warehouses; standard access mode DBR 13.3 LTS+; dedicated DBR 15.4+ with fine-grained access control (serverless enabled); dedicated 13.3-15.3 only for the table owner |
| DML on streaming tables | Shared/standard UC cluster or SQL warehouse, DBR 13.3 LTS+ |

## 4. Sources a UC pipeline can read

- UC managed and external tables, views, MVs and STs; Hive metastore tables (`hive_metastore` or an
  HMS federation catalog).
- Files via Auto Loader / `read_files()` from UC volumes and external locations.
- Apache Kafka and Amazon Kinesis (plus other message buses documented for pipelines).
- Change feeds: `spark.readStream.option("readChangeFeed", "true").table(...)`; for sources with
  updates/deletes use `skipChangeCommits` or read them in a materialized view.

## 5. Governance on pipeline tables

- Row filters and column masks are declared in the dataset definition and managed through
  `CREATE OR REFRESH` (not separate `ALTER TABLE` statements). During refresh the functions run with
  the pipeline owner's identity; at query time with the reader's identity. Since Mar 2026 updates
  preserve these policies.

```sql
CREATE OR REFRESH MATERIALIZED VIEW main.sales.customers_secure (
  customer_id BIGINT,
  region STRING,
  ssn STRING MASK main.sec.mask_ssn USING COLUMNS (region)
)
WITH ROW FILTER main.sec.region_filter ON (region)
AS SELECT customer_id, region, ssn FROM main.crm.customers;
```

```python
@dp.materialized_view(
    schema="customer_id BIGINT, region STRING, ssn STRING MASK main.sec.mask_ssn USING COLUMNS (region)",
    row_filter="ROW FILTER main.sec.region_filter ON (region)",
)
def customers_secure():
    return spark.read.table("main.crm.customers").select("customer_id", "region", "ssn")
```

- Lineage for pipeline tables is shown in Catalog Explorer, linking to the producing pipeline.
- Comments and RLS/CLM can also be set with `ALTER` on MVs/STs (2025.29); values set in source code
  win on refresh.
- Do not share an MV's underlying storage with untrusted consumers: backing files can contain
  upstream values not visible in the MV schema.

## 6. DML on streaming tables

`INSERT`, `UPDATE`, `DELETE`, `MERGE` are allowed on streaming tables published to UC (for example
GDPR deletes). DML that changes the table schema is not. Downstream streams reading a modified
streaming table need `skipChangeCommits`.

```sql
DELETE FROM main.sales.customers_bronze WHERE customer_id = 123;
```

## 7. Inactive tables and deleting pipelines

- Removing a dataset from the code marks its table **inactive** on the next update; it stays
  queryable but is no longer refreshed. `DROP` to clean up; `UNDROP` within 7 days.
- `"pipelines.dropInactiveTables": "true"` restores the old behavior of removing inactive entries.
- Changing the default catalog/schema without fully qualified names creates new tables in the new
  location and marks the old ones inactive.
- Deleting a pipeline in the default publishing mode **retains** its MVs, STs and views by default
  (queryable, not updatable until moved into another pipeline). `DELETE
  /api/2.0/pipelines/{id}?cascade=true` deletes them too; in bundles, `cascade_on_destroy: false`
  keeps datasets on `bundle destroy`.
- Move a table to another pipeline (DBR 16.3+ to run the command; 17.2+ across workspaces):

```sql
ALTER STREAMING TABLE main.sales.orders
  SET TBLPROPERTIES ("pipelines.pipelineId" = "<destination-pipeline-id>");
```

## 8. External (non-Databricks) access

Off by default. External data access: DBR 17.3+, pipeline configuration
`pipelines.externalMetadata.enabled = true` (or table property for standalone objects), clients on
Delta 4.0+ or Iceberg v3 REST APIs, reader needs `EXTERNAL USE SCHEMA` + `SELECT`; add
`delta.enableIcebergCompatV3 = true` for Iceberg readers. Compatibility mode (Public Preview) makes
a read-only copy for older clients.

## 9. Limitations

- JARs not supported; Python libraries only. Global init scripts not supported.
- `LOCATION` not supported; data lives in the schema/catalog/metastore managed storage.
- UC UDFs cannot be created inside a pipeline (existing ones can be called).
- MV cannot be a streaming source outside its pipeline; no time travel on MVs; no `CLONE`.
- `ALTER TABLE` is disallowed on streaming tables (use `ALTER STREAMING TABLE` or `CREATE OR REFRESH`).
- Hive metastore pipelines cannot be upgraded in place; a UC pipeline cannot publish to HMS.
- Driver logs of UC pipelines are visible only to the owner and admins by default.
- Backing data for MVs/STs lives in the reserved `__databricks_internal` catalog.

## Sources

- https://docs.databricks.com/aws/en/ldp/unity-catalog
- https://docs.databricks.com/aws/en/ldp/target-schema
- https://docs.databricks.com/aws/en/ldp/configure-pipeline
- https://docs.databricks.com/aws/en/ldp/move-table
- https://docs.databricks.com/aws/en/ldp/external-access
- https://docs.databricks.com/aws/en/external-access/external-for-pipelines
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-materialized-view
- https://docs.databricks.com/aws/en/ldp/developer/ldp-python-ref-materialized-view
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-streaming-table
- https://docs.databricks.com/aws/en/ldp/load
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources
- https://docs.databricks.com/aws/en/release-notes/dlt/2026
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/29/
- https://docs.databricks.com/aws/en/release-notes/dlt/2025/04/
