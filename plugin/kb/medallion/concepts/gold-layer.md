# Gold Layer

> **Purpose**: Business layer — dimensional models and aggregates per domain, materialized views, DDL-owned schemas, semantic layer on top
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Gold holds refined, business-aligned data for dashboards, ML and applications. Per the Databricks medallion guidance, this is where you **model dimensionally** (facts, dimensions, measures) and **aggregate**; because Gold models a business domain, several Gold layers (finance, HR, ...) are normal. Pre-aggregations are typically **materialized views** with incremental refresh. Gold tables are queried constantly, so their layout matters (`CLUSTER BY AUTO`) and their schema is a contract: change it explicitly (DDL), never implicitly from a DataFrame. Governed business metrics go one step further into Unity Catalog metric views — see [metric views](../../metric-views/index.md).

## The Pattern

```sql
-- 1. Aggregate as a materialized view (Pro/serverless SQL warehouse or a pipeline)
CREATE OR REPLACE MATERIALIZED VIEW main.gold.agg_monthly_revenue
CLUSTER BY AUTO
SCHEDULE EVERY 1 HOUR
AS SELECT date_trunc('month', o.order_ts) AS order_month,
          c.customer_segment, c.region,
          SUM(o.amount)                 AS total_revenue,
          COUNT(DISTINCT o.customer_id) AS unique_customers,
          COUNT(o.order_id)             AS total_orders
   FROM main.silver.orders o
   JOIN main.silver.customers c USING (customer_id)
   GROUP BY ALL;

-- 2. Fact table whose schema is owned by DDL; loads never change the schema
CREATE TABLE IF NOT EXISTS main.gold.fact_orders (
  order_id     STRING NOT NULL,
  customer_sk  BIGINT NOT NULL,
  date_sk      INT    NOT NULL,
  quantity     INT,
  net_amount   DECIMAL(12,2),
  _computed_at TIMESTAMP,
  CONSTRAINT fact_orders_pk PRIMARY KEY (order_id) RELY   -- informational, not enforced
) CLUSTER BY AUTO;

INSERT INTO main.gold.fact_orders
REPLACE WHERE date_sk >= 20261001                -- atomic slice refresh (DBR 12.2 LTS+)
SELECT order_id, customer_sk, date_sk, quantity, net_amount, current_timestamp()
FROM main.silver.orders_enriched WHERE date_sk >= 20261001;
```

```python
# PySpark full refresh of a DDL-created table: plain overwrite keeps the table's schema;
# a DataFrame that does not match it fails instead of silently changing the contract.
gold_df.write.mode("overwrite").saveAsTable("main.gold.agg_customer_revenue")
```

## Quick Reference

| Property | Value | Notes |
|---|---|---|
| Content | Dimensional model + aggregates per business domain | Several Gold layers allowed |
| Aggregates | Materialized views (`CREATE OR REPLACE MATERIALIZED VIEW`) | Incremental refresh; expectations allowed |
| Tables | `CREATE TABLE` once, then `MERGE` / `INSERT ... REPLACE WHERE / USING` / plain overwrite | Schema changes via `ALTER TABLE` |
| Layout | `CLUSTER BY AUTO` (UC managed) or `CLUSTER BY (date_sk, customer_sk)` | ZORDER/partitioning are legacy here |
| Keys | PK/FK constraints are informational; `RELY` lets Photon ([photon](../../spark/concepts/photon.md)) use them for rewrites | NOT NULL and CHECK are enforced |
| Refresh tracking | `_computed_at` column, or `DESCRIBE HISTORY` / MV refresh history | — |
| Semantic layer | Unity Catalog metric views over Gold | [../../metric-views/index.md](../../metric-views/index.md) |

| Replace option | Effect (Databricks docs) |
|---|---|
| `CREATE OR REPLACE TABLE` | Keeps history, grants, row filters, column masks; the new definition replaces the schema; restate `CLUSTER BY AUTO`; cannot toggle catalog commits |
| `DROP TABLE` + `CREATE TABLE` | Loses history and grants; Databricks recommends `REPLACE` instead |
| `.option("overwriteSchema", "true")` | Avoid — see below |

## Common Mistakes

### Wrong

```python
# overwriteSchema replaces the table schema and partitioning under every consumer in one
# commit: dashboards and views break silently, streaming readers fail on the schema change,
# and comments on columns the DataFrame recomputed are lost.
(gold_df.write.format("delta").mode("overwrite")
    .option("overwriteSchema", "true").saveAsTable("main.gold.agg_revenue"))

bronze_df.groupBy("region").agg(sum("amount"))   # aggregating Bronze: duplicates, dirty data
```

### Correct

```sql
ALTER TABLE main.gold.agg_revenue ADD COLUMN channel STRING;   -- explicit, reviewable change
-- then write with a DataFrame matching the new schema (plain overwrite, MERGE or REPLACE WHERE)
```

## Gold for ML

Publish certified Gold features with Feature Engineering in Unity Catalog: `FeatureEngineeringClient().create_table(name=..., primary_keys=[...], timeseries_columns="event_ts", df=...)`, then `fe.write_table(name=..., df=..., mode="merge")`. Point-in-time joins require the timestamp key to be declared in `timeseries_columns`.

## Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| One massive Gold table for everything | Purpose-specific facts, dims and MVs per domain |
| Nightly `OPTIMIZE ... ZORDER` jobs | `CLUSTER BY AUTO` + predictive optimization |
| Metrics re-implemented in every dashboard | Define them once in a metric view |
| Uncertified tables consumed by AI/BI and agents | Tag certified Gold tables; add comments and constraints |

## Related

- [Silver Layer](silver-layer.md)
- [Layer Transitions](../patterns/layer-transitions.md)
- [Domain Modeling](domain-modeling.md)
- [Star schema](../../data-modeling/patterns/star-schema.md)

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-using
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-dml-insert-into
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-constraint
- https://docs.databricks.com/aws/en/tables/update-schema
- https://docs.databricks.com/aws/en/structured-streaming/delta-lake
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/machine-learning/feature-store/uc/feature-tables-uc
- https://docs.databricks.com/aws/en/machine-learning/feature-store/time-series
