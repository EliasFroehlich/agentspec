# SQL Table Syntax for Lakeflow Pipelines

> **Purpose**: `CREATE OR REFRESH STREAMING TABLE` / `MATERIALIZED VIEW`, `read_files`, constraints, clustering, private tables, targets and parameters in pipeline SQL
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## When to Use

- Writing pipeline source files in SQL (`transformations/*.sql`)
- Ingesting files with `STREAM read_files(...)` (Auto Loader)
- Publishing to non-default catalogs/schemas or parameterizing per environment

## Implementation

```sql
-- Bronze: Auto Loader via read_files (options have NO cloudFiles. prefix)
CREATE OR REFRESH STREAMING TABLE customers_bronze
CLUSTER BY AUTO
COMMENT "Raw customers"
TBLPROPERTIES ("quality" = "bronze")
AS SELECT *, _metadata.file_path AS _source_file
FROM STREAM read_files(
  "/Volumes/main/landing/files/customers/",
  format => "json",
  inferColumnTypes => true,
  schemaHints => "customer_id BIGINT"
);

-- Silver: constraints (expectations) on a streaming table
CREATE OR REFRESH STREAMING TABLE customers_silver (
  CONSTRAINT valid_id    EXPECT (customer_id IS NOT NULL) ON VIOLATION DROP ROW,
  CONSTRAINT valid_email EXPECT (email LIKE '%@%.%')      ON VIOLATION DROP ROW
)
CLUSTER BY (customer_id)
COMMENT "Clean customers"
AS SELECT customer_id, name, email, current_timestamp() AS processed_at
FROM STREAM(customers_bronze);

-- Gold: materialized view with an explicit refresh policy
CREATE OR REFRESH MATERIALIZED VIEW customer_domains
REFRESH POLICY AUTO
AS SELECT split(email, '@')[1] AS domain, count(*) AS customer_count
FROM customers_silver
GROUP BY split(email, '@')[1];

-- Private: persisted for the pipeline lifetime, not published to the catalog
CREATE OR REFRESH PRIVATE MATERIALIZED VIEW customer_ids AS
SELECT DISTINCT customer_id FROM customers_silver;
```

### Targets: fully qualified names or USE CATALOG / USE SCHEMA

```sql
-- Writes to main.reporting regardless of the pipeline defaults
CREATE OR REFRESH MATERIALIZED VIEW main.reporting.customer_domains_by_day AS
SELECT * FROM customer_domains;

-- Scoped to the rest of this file
USE CATALOG main;
USE SCHEMA reporting;
```

### Parameters

```sql
-- Pipeline parameters (Beta): `parameters` map in settings, referenced with :name
CREATE OR REFRESH MATERIALIZED VIEW sales_since AS
SELECT * FROM :source_catalog.sales.orders
WHERE order_date >= :start_date;

-- In identifier-only positions wrap the parameter in IDENTIFIER()
USE CATALOG IDENTIFIER(:source_catalog);

-- Legacy/Configuration mechanism: `configuration` map, referenced with ${key}
CREATE OR REFRESH STREAMING TABLE orders_bronze AS
SELECT * FROM STREAM read_files("${source_path}/orders/", format => "json");
```

## Configuration

| Clause | Notes |
|--------|-------|
| `CLUSTER BY (cols)` / `CLUSTER BY AUTO` | Liquid clustering; docs: "Consider using CLUSTER BY instead of PARTITIONED BY" |
| `PARTITIONED BY (cols)` | Mutually exclusive with `CLUSTER BY` |
| `PRIVATE` | Replaces legacy `TEMPORARY` |
| `CONSTRAINT ... EXPECT ... [ON VIOLATION DROP ROW \| FAIL UPDATE]` | Default action is warn |
| `REFRESH POLICY AUTO \| INCREMENTAL \| INCREMENTAL STRICT \| FULL` | Materialized views only |
| `WITH ROW FILTER`, column `MASK` | Manage through `CREATE OR REFRESH` |
| `LOCATION` | Not supported in Unity Catalog pipelines |
| `CREATE OR REFRESH LIVE TABLE`, `LIVE.` prefix | Legacy; `LIVE` is ignored in the default publishing mode |

| `read_files` option | Meaning |
|---------------------|---------|
| `format` | `json`, `csv`, `parquet`, `avro`, `orc`, `text`, `binaryFile`, `xml` |
| `inferColumnTypes`, `schemaHints` | Type inference / pinned types |
| `includeExistingFiles` | `false` = only files that arrive after creation |
| `maxFilesPerTrigger`, `maxBytesPerTrigger` | Batch sizing (don't set on serverless SQL warehouse STs) |
| `schemaLocation` | Not required inside a streaming table query |

## Example Usage

```sql
-- Wrong: Auto Loader option names with the cloudFiles. prefix inside read_files
SELECT * FROM STREAM read_files("/Volumes/main/landing/files/x/",
  format => "json", cloudFiles.inferColumnTypes => "true");

-- Correct
SELECT * FROM STREAM read_files("/Volumes/main/landing/files/x/",
  format => "json", inferColumnTypes => true);
```

`ALTER TABLE` is not allowed on streaming tables; use `CREATE OR REFRESH` or
`ALTER STREAMING TABLE` / `ALTER MATERIALIZED VIEW`.

## See Also

- [SQL streaming and CDC](sql-streaming.md)
- [Expectations](expectations.md)
- [Parameters reference](../reference/parameters.md)
- [Unity Catalog reference](../reference/unity-catalog.md)

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/sql-dev
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-streaming-table
- https://docs.databricks.com/aws/en/ldp/developer/ldp-sql-ref-create-materialized-view
- https://docs.databricks.com/aws/en/sql/language-manual/functions/read_files
- https://docs.databricks.com/aws/en/ldp/target-schema
- https://docs.databricks.com/aws/en/ldp/parameters
- https://docs.databricks.com/aws/en/ldp/incremental-refresh
- https://docs.databricks.com/aws/en/ldp/unity-catalog
