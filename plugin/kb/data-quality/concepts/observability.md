# Observability

> **Purpose**: Freshness monitoring, volume tracking, distribution drift and the tool landscape — including Databricks-native anomaly detection and data profiling
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Data observability extends monitoring beyond "did the pipeline succeed?" to "is the data still right?" The
five pillars are **freshness**, **volume**, **schema**, **distribution** and **lineage**. Observability tools
learn normal behavior from metadata and history instead of fixed thresholds; quality checks (tests,
contracts) assert known rules. Most teams need both.

**Tool landscape (verified 2026-10):**
- **Platform-native**: Databricks **data quality monitoring** — anomaly detection (freshness + completeness
  per schema/catalog, Public Preview) and data profiling (per-table profile and drift metrics, formerly
  Lakehouse Monitoring); Unity Catalog lineage. See [databricks-data-quality](databricks-data-quality.md).
- **Commercial platforms**: Monte Carlo (now positioned as data + AI observability), Soda Cloud (anomaly
  detection and metric monitors on top of Soda Core contracts), Coalesce Quality (SYNQ, acquired by
  **Coalesce** on 2026-03-10).
- **dbt-native**: Elementary — OSS CLI (0.24.x) + dbt package (0.26.x), Apache 2.0, with a paid Elementary Cloud.
- **Open-source checks**: GX Core, Soda Core, pandera — rules you define, not learned baselines.

## The Concept

```sql
-- Databricks SQL. Hand-rolled monitors — use them where anomaly detection is not enabled.

-- 1. FRESHNESS: hours since the newest record
SELECT 'main.sales.orders' AS table_name,
       MAX(updated_at) AS last_record_at,
       timestampdiff(SECOND, MAX(updated_at), current_timestamp()) / 3600.0 AS staleness_hours,
       CASE WHEN MAX(updated_at) < current_timestamp() - INTERVAL 2 HOURS
            THEN 'STALE' ELSE 'FRESH' END AS freshness_status
FROM main.sales.orders;

-- 2. VOLUME: today's rows vs the trailing 7-day average
WITH daily_counts AS (
  SELECT order_date, COUNT(*) AS row_count
  FROM main.sales.orders
  WHERE order_date >= current_date() - INTERVAL 30 DAYS
  GROUP BY order_date
)
SELECT order_date, row_count,
       AVG(row_count) OVER (ORDER BY order_date ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING) AS rolling_avg,
       row_count / NULLIF(AVG(row_count) OVER (
         ORDER BY order_date ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING), 0) AS volume_ratio  -- alert < 0.5 or > 2.0
FROM daily_counts;

-- 3. DISTRIBUTION: today's category mix, to compare with history
SELECT status, COUNT(*) AS current_count,
       COUNT(*) / SUM(COUNT(*)) OVER () AS current_pct
FROM main.sales.orders
WHERE order_date = current_date()
GROUP BY status;
```

`EXTRACT(EPOCH FROM …)` is PostgreSQL; Databricks SQL has no `EPOCH` field — use `timestampdiff` or
`unix_timestamp`.

## Quick Reference

| Pillar | What to monitor | Alert when | Databricks-native | Other tools |
|--------|-----------------|------------|-------------------|-------------|
| Freshness | Last commit / `MAX(updated_at)` | Later than learned or contracted SLA | Anomaly detection (commit history) | dbt source freshness, Soda `freshness`, Monte Carlo |
| Volume | Rows per day / per commit | Outside learned range | Anomaly detection (completeness, last 24 h) | Elementary, Soda Cloud |
| Schema | Added/removed/retyped columns | Any unexpected change | Delta schema enforcement; Lakeflow schema evolution | dbt contracts / `on_schema_change`, Soda `schema` |
| Distribution | Null rate, value mix, statistics | Drift beyond threshold | Data profiling drift metrics (`<table>_drift_metrics`) | Elementary, GX |
| Lineage | Upstream/downstream impact | Upstream failure or change | Unity Catalog lineage | dbt docs, Monte Carlo |

## Common Mistakes

### Wrong

```sql
-- Fixed threshold: breaks with growth and seasonality
SELECT CASE WHEN COUNT(*) < 10000 THEN 'ALERT' END FROM main.sales.orders;
```

### Correct

```sql
-- Dynamic threshold from rolling statistics (or let anomaly detection learn it)
SELECT CASE
         WHEN today_count < rolling_avg * 0.5 THEN 'LOW_VOLUME_ALERT'
         WHEN today_count > rolling_avg * 3.0 THEN 'HIGH_VOLUME_ALERT'
         ELSE 'OK'
       END AS volume_status
FROM volume_stats;
```

## Related

- [databricks-data-quality](databricks-data-quality.md)
- [quality-dimensions](quality-dimensions.md)
- [data-contracts](data-contracts.md)

## Sources

- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/anomaly-detection/
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/data-profiling/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/extract
- https://docs.databricks.com/aws/en/sql/language-manual/functions/timestampdiff
- https://coalesce.io/company-news/coalesce-announces-acquisition-of-synq-and-launch-of-coalesce-quality/
- https://github.com/elementary-data/elementary
- https://github.com/elementary-data/dbt-data-reliability/releases
- https://montecarlo.ai/
- https://github.com/sodadata/soda-core
