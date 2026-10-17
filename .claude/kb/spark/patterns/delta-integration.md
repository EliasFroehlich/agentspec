# Delta Integration

> **Purpose**: Delta Lake from Spark — liquid clustering (incl. `CLUSTER BY AUTO`), MERGE upsert and SCD Type 2, schema evolution, time travel, change data feed, UniForm, and what Databricks automates (predictive optimization, file sizing)
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- ACID upserts (MERGE), deletes and time travel on lake storage
- Tables that need a data layout that can evolve (liquid clustering instead of partitions/ZORDER)
- Incremental downstream processing via change data feed
- Iceberg readers on the same data (UniForm)

Versions: Delta Lake OSS 4.4.1 (2026-10-05) builds for Spark 4.0.1/4.1.0/4.2.0 with Spark-suffixed
artifacts (e.g. `delta-spark_4.2_2.13`); DBR 17.3 LTS ships Delta 4.0.0. On Databricks, Delta is
built in — no `spark.sql.extensions` / `DeltaCatalog` configuration is needed.

## Implementation

```sql
-- 1. LAYOUT: liquid clustering (Databricks GA DBR 15.4 LTS; OSS Delta CLUSTER BY 3.1+, ALTER 3.3+)
CREATE TABLE main.sales.orders (order_id BIGINT, customer_id BIGINT, order_date DATE, amount DECIMAL(12,2))
CLUSTER BY (order_date, customer_id);          -- up to 4 keys; not combinable with PARTITIONED BY / ZORDER

ALTER TABLE main.sales.orders CLUSTER BY (customer_id);   -- change keys without rewriting data
OPTIMIZE main.sales.orders FULL;               -- recluster everything after enabling/changing keys

-- Unity Catalog managed tables + predictive optimization: let Databricks pick keys (DBR 15.4+)
CREATE OR REPLACE TABLE main.sales.events (event_id BIGINT, payload VARIANT, ts TIMESTAMP) CLUSTER BY AUTO;
-- CREATE OR REPLACE without CLUSTER BY AUTO turns automatic clustering off again.

-- Convert a partitioned table in place (DBR 18.1+, so DBR 18 LTS / 19; not DBR 17.3)
ALTER TABLE main.sales.orders_legacy REPLACE PARTITIONED BY WITH CLUSTER BY AUTO;

-- 2. MERGE with schema evolution (Databricks DBR 15.2+; Spark SQL grammar 4.0+)
MERGE WITH SCHEMA EVOLUTION INTO main.sales.orders AS t
USING main.sales.orders_updates AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- 3. TIME TRAVEL / CDF / VACUUM
SELECT * FROM main.sales.orders VERSION AS OF 42;
SELECT * FROM main.sales.orders TIMESTAMP AS OF '2026-10-01';
ALTER TABLE main.sales.orders SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
SELECT * FROM table_changes('main.sales.orders', 2, 5) WHERE _change_type IN ('insert', 'update_postimage');
VACUUM main.sales.orders RETAIN 168 HOURS;     -- 7-day minimum safety window
```

```python
from delta.tables import DeltaTable
from pyspark.sql import functions as F

# --- SCD TYPE 2 with a staged-updates MERGE (one statement closes old rows AND inserts new versions)
target = DeltaTable.forName(spark, "main.silver.customers")   # OSS path tables: DeltaTable.forPath
updates = spark.read.table("main.bronze.customers_daily")
current = spark.read.table("main.silver.customers").filter("is_current = true")

changed = (updates.alias("s")
    .join(current.alias("t"), "customer_id")
    .where("NOT (t.email <=> s.email) OR NOT (t.name <=> s.name)")   # null-safe change test
    .selectExpr("CAST(NULL AS BIGINT) AS merge_key", "s.*"))         # forces INSERT of new version
staged = changed.unionByName(updates.selectExpr("customer_id AS merge_key", "*"))

(target.alias("t")
    .merge(staged.alias("s"), "t.customer_id = s.merge_key")
    .whenMatchedUpdate(
        condition="t.is_current = true AND (NOT (t.email <=> s.email) OR NOT (t.name <=> s.name))",
        set={"is_current": "false", "end_date": "s.effective_date"})
    .whenNotMatchedInsert(values={
        "customer_id": "s.customer_id", "name": "s.name", "email": "s.email",
        "is_current": "true", "start_date": "s.effective_date", "end_date": "CAST(NULL AS DATE)"})
    .execute())
```

## Configuration

| Setting / feature | Default | Notes |
|-------------------|---------|-------|
| `delta.enableChangeDataFeed` | `false` | Enable before you need the history |
| `delta.deletedFileRetentionDuration` | 7 days | VACUUM safety window; predictive optimization respects a 7-day floor |
| `delta.logRetentionDuration` | 30 days | Time-travel horizon for the transaction log |
| File size (UC managed tables) | auto-tuned | Manual tuning advice does not apply; only `OPTIMIZE` respects `delta.targetFileSize` |
| Auto compaction / optimized writes | always on for MERGE/UPDATE/DELETE (Databricks) | Configure only for external/legacy tables |
| Predictive optimization | default for accounts created on/after 2024-11-11 (rollout to older accounts through 2026-08) | Runs OPTIMIZE/VACUUM/ANALYZE on UC managed tables via serverless; does not run ZORDER |
| `delta.enableTypeWidening` | unset | Delta 3.2 preview, 4.0+ supported (Databricks DBR 15.4+): widen column types without rewrite |
| `delta.universalFormat.enabledFormats` + `delta.enableIcebergCompatV2` | unset | UniForm (Iceberg read); requires column mapping; not compatible with deletion vectors; Iceberg clients read only |

## Example Usage

```python
# Incremental downstream processing from the change data feed
changes = (spark.read
    .option("readChangeFeed", "true")
    .option("startingVersion", 10)
    .table("main.sales.orders"))
new_and_updated = changes.filter(F.col("_change_type").isin("insert", "update_postimage"))
```

### Legacy layout (existing tables only)

```sql
-- ZORDER is legacy on Databricks; liquid clustering replaces partitioning and ZORDER.
OPTIMIZE main.sales.orders_old ZORDER BY (customer_id);
```

## See Also

- [read-write-patterns](../patterns/read-write-patterns.md)
- [partitioning](../concepts/partitioning.md)
- [delta-lake](../../lakehouse/concepts/delta-lake.md)
- [liquid-clustering](../../lakehouse/concepts/liquid-clustering.md)

## Sources

- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.delta.io/latest/delta-clustering.html
- https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- https://docs.databricks.com/aws/en/delta/tune-file-size
- https://docs.databricks.com/aws/en/sql/language-manual/delta-merge-into
- https://docs.delta.io/latest/delta-update.html
- https://docs.delta.io/latest/delta-change-data-feed.html
- https://docs.delta.io/latest/delta-type-widening.html
- https://docs.databricks.com/aws/en/delta/type-widening
- https://docs.delta.io/latest/delta-uniform.html
- https://github.com/delta-io/delta/releases/tag/v4.4.1
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://spark.apache.org/releases/spark-release-4-0-0.html
