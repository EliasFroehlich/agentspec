# Catalog Setup

> **Purpose**: Configure Unity Catalog (namespaces, storage, grants, ABAC, Iceberg REST access), Gravitino's Iceberg REST service, Nessie branching and Polaris with OPA
> **Confidence**: 0.88
> **MCP Validated**: 2026-10-17

## When to Use

- Standing up the namespace and permission model for a lakehouse on Databricks
- Letting external engines (Spark, Trino, Flink) read or write Unity Catalog tables over the Iceberg REST Catalog (IRC)
- Running an engine-neutral Iceberg REST catalog (Gravitino, Polaris) or git-like branches (Nessie)

## Implementation

```sql
-- ============================================================
-- Unity Catalog (Databricks): namespace, storage, privileges
-- ============================================================
CREATE CATALOG IF NOT EXISTS analytics;
CREATE SCHEMA IF NOT EXISTS analytics.bronze;
CREATE SCHEMA IF NOT EXISTS analytics.silver;
CREATE SCHEMA IF NOT EXISTS analytics.gold;

-- External location for landing files read by ingestion (tables stay managed)
CREATE EXTERNAL LOCATION IF NOT EXISTS landing_zone
URL 's3://my-landing-bucket/landing/'
WITH (STORAGE CREDENTIAL landing_cred);

GRANT USE CATALOG ON CATALOG analytics TO `data-engineers`;
GRANT USE SCHEMA, CREATE TABLE, MODIFY, SELECT ON SCHEMA analytics.silver TO `data-engineers`;
GRANT USE SCHEMA, SELECT ON SCHEMA analytics.gold TO `analysts`;

-- Maintenance and layout are delegated to the platform
ALTER CATALOG analytics ENABLE PREDICTIVE OPTIMIZATION;

-- ============================================================
-- ABAC: one policy for every table tagged sensitivity=high
-- Requires governed tags; serverless, or DBR 16.4+ standard/dedicated compute
-- ============================================================
CREATE FUNCTION analytics.gold.non_eu_region(geo_region STRING) RETURNS BOOLEAN
  RETURN geo_region <> 'eu';

CREATE POLICY hide_eu_customers
ON SCHEMA analytics.gold
COMMENT 'Exclude EU customers from sensitive tables for US analysts'
ROW FILTER analytics.gold.non_eu_region
TO `us-analysts`
FOR TABLES
WHEN has_tag_value('sensitivity', 'high')
MATCH COLUMNS has_tag('geo_region') AS region
USING COLUMNS (region);

-- ============================================================
-- Iceberg REST access for external engines
-- ============================================================
-- Metastore admin: enable "External data access" on the metastore, then:
GRANT EXTERNAL USE SCHEMA ON SCHEMA analytics.gold TO `external-engine-sp`;
```

```python
# External Apache Spark reading/writing Unity Catalog tables through IRC (OAuth M2M)
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .config("spark.jars.packages",
            "org.apache.iceberg:iceberg-spark-runtime-<spark>_<scala>:<iceberg-version>,"
            "org.apache.iceberg:iceberg-aws-bundle:<iceberg-version>")      # 1.9.2+ recommended
    .config("spark.sql.catalog.uc", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.uc.type", "rest")
    .config("spark.sql.catalog.uc.rest.auth.type", "oauth2")
    .config("spark.sql.catalog.uc.uri", "https://<workspace-url>/api/2.1/unity-catalog/iceberg-rest")
    .config("spark.sql.catalog.uc.oauth2-server-uri", "https://<workspace-url>/oidc/v1/token")
    .config("spark.sql.catalog.uc.credential", "<client_id>:<client_secret>")
    .config("spark.sql.catalog.uc.warehouse", "analytics")                  # UC catalog name
    .config("spark.sql.catalog.uc.scope", "all-apis")
    .getOrCreate()
)
spark.table("uc.gold.orders").show()   # on AWS also set AWS_REGION for the Iceberg client
```

```properties
# Gravitino Iceberg REST service (standalone: conf/gravitino-iceberg-rest-server.conf)
gravitino.iceberg-rest.host = 0.0.0.0
gravitino.iceberg-rest.httpPort = 9001
gravitino.iceberg-rest.catalog-backend = jdbc
# ...plus the jdbc backend keys (uri, warehouse, driver, credentials) from the Gravitino docs.
# Engines connect to http://<host>:9001/iceberg/
```

```python
# Nessie with Spark: the active branch comes from the catalog "ref" setting
spark = (
    SparkSession.builder
    .config("spark.sql.extensions",
            "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,"
            "org.projectnessie.spark.extensions.NessieSparkSessionExtensions")
    .config("spark.sql.catalog.nessie", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.nessie.catalog-impl", "org.apache.iceberg.nessie.NessieCatalog")
    .config("spark.sql.catalog.nessie.uri", "http://nessie:19120/api/v1")
    .config("spark.sql.catalog.nessie.ref", "main")
    .config("spark.sql.catalog.nessie.warehouse", "s3://my-bucket/warehouse")
    .getOrCreate()
)
spark.sql("CREATE BRANCH IF NOT EXISTS feature_new_model IN nessie FROM main")
spark.sql("MERGE BRANCH feature_new_model INTO main IN nessie")
```

```properties
# Polaris (image apache/polaris:<version>, 1.8.0 current): OPA authorization is a PREVIEW feature
polaris.authorization.type=opa
polaris.authorization.opa.policy-uri=http://opa:8181/v1/data/polaris/authz
polaris.authorization.opa.http.timeout=PT2S
```

## Configuration

| Setting | Default | Description |
|---|---|---|
| Predictive optimization | Enabled for accounts created on/after 2024-11-11 | `ALTER CATALOG/SCHEMA/TABLE ... ENABLE | DISABLE | INHERIT PREDICTIVE OPTIMIZATION` |
| `EXTERNAL USE SCHEMA` | Not granted | Required, with metastore "External data access", for IRC clients |
| ABAC quotas | 100 policies per catalog/schema, 50 per table, 20 principals per policy | From the ABAC requirements page |
| `gravitino.iceberg-rest.catalog-backend` | `memory` | Use `hive`, `jdbc` or `rest` in production |
| `polaris.authorization.type` | `internal` | `opa` delegates decisions to an OPA policy endpoint |

## Example Usage

```sql
-- Who can read what (UC)
SHOW GRANTS ON SCHEMA analytics.gold;
DESCRIBE TABLE EXTENDED analytics.gold.orders;   -- shows predictive optimization status
-- Polaris from Spark: spark.sql.catalog.polaris.type=rest, uri=http://polaris:8181/api/catalog
```

## See Also

- [catalog-wars concept](../concepts/catalog-wars.md)
- [unity-catalog-iceberg concept](../concepts/unity-catalog-iceberg.md)
- [iceberg-operations](iceberg-operations.md)
- [delta-operations](delta-operations.md)

## Sources

- https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/policies
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/requirements
- https://docs.databricks.com/aws/en/external-access/iceberg
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- https://gravitino.apache.org/docs/1.3.1/iceberg-rest-service
- https://projectnessie.org/iceberg/spark/
- https://projectnessie.org/nessie-latest/spark-sql/
- https://polaris.apache.org/releases/1.7.0/managing-security/external-pdp/opa/
- https://polaris.apache.org/downloads/
- https://polaris.apache.org/in-dev/unreleased/generic-table/
