# Cross-Platform Cost Optimization

> **Purpose**: Cost control across Snowflake, Databricks and BigQuery — current list prices (dated), idle-compute shutdown, spend caps, right-sizing, storage tiers and query-cost monitoring
> **Confidence**: HIGH (prices change: re-check the linked pricing pages before quoting)
> **MCP Validated**: 2026-10-17

## When to Use

- Estimating or comparing platform cost for a workload
- Cutting idle compute and runaway queries
- Choosing on-demand vs capacity/committed pricing
- Building a weekly cost review query per platform

## List Prices (as of 2026-10-17)

| Platform | Compute | Storage | Source |
|----------|---------|---------|--------|
| Snowflake | On-demand credit (AWS US East): Standard **$2.00**, Enterprise **$3.00**, Business Critical **$4.00**, VPS $6.00 | On-demand **$23/TB-month**; capacity tiers down to $13.80 | [Credit Consumption Table](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf) (effective 2026-10-15) |
| Databricks | List "starting at": Data Engineering (Lakeflow Jobs/Pipelines/Connect) **$0.15/DBU**, Data Warehousing **$0.22/DBU**, Interactive **$0.40/DBU**, AI **$0.07/DBU**; Lakebase **$0.069/CU**; Genie $0.07/DBU beyond free usage (free through 2027-01-31) | Your cloud object storage (some managed products bill DSUs) | [Databricks pricing](https://www.databricks.com/product/pricing) (varies by cloud, region, tier) |
| BigQuery | On-demand **$6.25/TiB** scanned, first **1 TiB/month free**; editions pay-as-you-go per slot-hour (US): Standard **$0.04**, Enterprise **$0.06**, Enterprise Plus **$0.10** | Active logical **$0.000031507/GiB-hour** (≈ $0.023/GiB-month); long-term logical $0.000021918/GiB-hour (≈ $0.016); first 10 GiB free | [BigQuery pricing](https://cloud.google.com/bigquery/pricing) |

Notes: BigQuery slot commitments (1 or 3 years) exist only for Enterprise and Enterprise Plus; editions
bill per second with a 1-minute minimum unless you opt in to fluid scaling. Databricks prices differ by
cloud, region, tier and classic vs serverless SKU; quote the per-product page, not the "starting at" figure.

## Implementation

### Stop paying for idle compute

```sql
-- Snowflake: suspend after 60 s idle, cap monthly credits
ALTER WAREHOUSE transform_wh SET AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;

CREATE OR REPLACE RESOURCE MONITOR monthly_limit
  WITH CREDIT_QUOTA = 5000
  TRIGGERS ON 75 PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND
           ON 110 PERCENT DO SUSPEND_IMMEDIATE;
ALTER WAREHOUSE transform_wh SET RESOURCE_MONITOR = monthly_limit;
```

```python
# Databricks: prefer serverless jobs (no idle clusters). Cost mode is per job.
job_settings = {"performance_target": "STANDARD"}   # 4-6 min start-up, fewer DBUs, same SKU
# Classic all-purpose compute only: terminate idle clusters
classic_cluster = {"autotermination_minutes": 30,
                   "autoscale": {"min_workers": 1, "max_workers": 8}}
# SQL warehouses: Auto Stop default 10 min (serverless; 5 min UI floor, 1 min via API)
#                 and 45 min (pro/classic; 10 min floor)
```

```bash
# BigQuery: serverless, so cap bytes instead of suspending compute
bq query --use_legacy_sql=false --dry_run 'SELECT order_id FROM analytics.orders'
bq query --use_legacy_sql=false --maximum_bytes_billed=10737418240 'SELECT ...'   # fail > 10 GiB
# Same setting: console Query settings > Advanced, or maximumBytesBilled in the job config.
# Project/user hard caps: custom query quotas.
```

### Right-sizing rules of thumb

```text
Snowflake
  New standard warehouses default to GENERATION = '2' (Gen2) where available; test Gen2 before upsizing.
  Adaptive Warehouses (GA 2026-06-16, Enterprise+): no manual sizing, query-based billing;
    tune MAX_QUERY_PERFORMANCE_LEVEL and QUERY_THROUGHPUT_MULTIPLIER instead of size/clusters.
  Concurrency problem -> multi-cluster (SCALING_POLICY = STANDARD); slow single query -> size up.
  Interactive warehouses + interactive tables for low-latency, high-concurrency dashboards/APIs.

Databricks
  Serverless jobs/pipelines first; Photon is always on there and default on classic jobs compute.
  Classic: right-size from the Spark UI/query profile, enable autoscaling, use job compute, not all-purpose.
  Tag spend with serverless usage policies (Public Preview) or budget policies.

BigQuery
  < ~1 TiB/month scanned: on-demand is effectively free; steady heavy use: editions + autoscaler.
  Set a reservation baseline for steady load and let the autoscaler cover peaks.
```

### Storage tiers

| Tier | Snowflake | Databricks | BigQuery |
|------|-----------|------------|----------|
| Active | $23/TB-month on-demand (capacity cheaper) | Cloud object storage rates | Active logical ≈ $0.023/GiB-month |
| Long-term | Same rate (no automatic tier) | Cloud lifecycle policies (cold tiers) | Long-term logical ≈ $0.016/GiB-month after 90 days unmodified |
| History | Time Travel (1–90 days) + 7-day Fail-safe, billed as storage | Delta log + `VACUUM` retention (7-day floor with predictive optimization) | Time travel window + fail-safe |

### Query cost monitoring

```sql
-- Snowflake: heaviest queries in the last 7 days
SELECT query_id, user_name, warehouse_name,
       total_elapsed_time / 1000 AS duration_s,
       bytes_scanned / POWER(1024, 3) AS gb_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC LIMIT 20;

-- Databricks: DBUs per product and SKU, last 30 days
SELECT billing_origin_product, sku_name, SUM(usage_quantity) AS dbus
FROM system.billing.usage
WHERE usage_unit = 'DBU' AND usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY ALL ORDER BY dbus DESC;

-- BigQuery: on-demand cost estimate per job (US list price as of 2026-10-17)
SELECT job_id, user_email,
       total_bytes_billed / POW(1024, 4)        AS tib_billed,
       total_bytes_billed / POW(1024, 4) * 6.25 AS est_usd,
       total_slot_ms / 1000                     AS slot_seconds
FROM `region-us`.INFORMATION_SCHEMA.JOBS
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY total_bytes_billed DESC LIMIT 20;
```

## Configuration

| Lever | Snowflake | Databricks | BigQuery |
|-------|-----------|------------|----------|
| Idle shutdown | `AUTO_SUSPEND = 60` | Serverless; `autotermination_minutes`; SQL warehouse Auto Stop | N/A (serverless) |
| Spend cap | Resource monitor | Budgets / usage policies | Maximum bytes billed, custom quotas |
| Right-size | Size, Gen2, Adaptive | Serverless mode, worker count/type | Edition + reservation baseline |
| Cost view | `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` | `system.billing.usage` | `INFORMATION_SCHEMA.JOBS` |
| Data layout | Micro-partitions + clustering keys | Liquid clustering, `CLUSTER BY AUTO` (replaces partitioning and ZORDER) | Partition + cluster |

## Example Usage

### Wrong

```python
# Databricks: this is the legacy High-Concurrency profile, NOT serverless compute
new_cluster = {"spark_conf": {"spark.databricks.cluster.profile": "serverless"}}
```

```sql
-- BigQuery: SELECT has no OPTIONS clause; this fails to parse
SELECT * FROM analytics.large_table OPTIONS(max_bytes_billed = 10737418240);
```

### Correct

```python
# Databricks serverless job task: just omit the cluster spec
task = {"task_key": "etl", "notebook_task": {"notebook_path": "/Workspace/Shared/etl/main"}}
```

## See Also

- [Snowflake Patterns](snowflake-patterns.md) — Gen2 and Adaptive warehouse DDL
- [Databricks Patterns](databricks-patterns.md) — serverless jobs, access modes
- [BigQuery Patterns](bigquery-patterns.md) — dry run, partition pruning

## Sources

- https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf
- https://docs.snowflake.com/en/user-guide/warehouses-gen2
- https://docs.snowflake.com/en/user-guide/warehouses-adaptive
- https://docs.snowflake.com/en/release-notes/2026/other/2026-06-16-adaptive-compute-ga
- https://docs.snowflake.com/en/release-notes/2025/other/2025-12-11-interactive-tables-ga
- https://docs.snowflake.com/en/sql-reference/sql/create-resource-monitor
- https://www.databricks.com/product/pricing
- https://docs.databricks.com/aws/en/jobs/run-serverless-jobs
- https://docs.databricks.com/aws/en/compute/sql-warehouse/create
- https://docs.databricks.com/aws/en/compute/photon
- https://docs.databricks.com/aws/en/admin/system-tables/billing
- https://docs.databricks.com/aws/en/delta/clustering
- https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/cluster
- https://cloud.google.com/bigquery/pricing
- https://docs.cloud.google.com/bigquery/docs/best-practices-costs
- https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax
