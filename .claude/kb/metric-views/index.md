# Unity Catalog Metric Views Knowledge Base

> **Purpose**: Unity Catalog metric views, the core implementation of Unity Catalog semantics: YAML-defined fields and measures, `MEASURE()` queries, declarative star/snowflake joins, window measures, parameters, materialization, agent metadata, lifecycle and deployment as code
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

Docs moved from `/metric-views/` to `/uc-semantics/metric-views/` (old URLs return HTTP 301 to the new path).

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/fundamentals.md](concepts/fundamentals.md) | What a metric view is, fields vs measures, why ratios re-aggregate safely |
| [concepts/requirements-versions.md](concepts/requirements-versions.md) | YAML spec versions (`0.1`, `1.1`), DBR feature matrix, permissions, create/edit methods |
| [concepts/limitations.md](concepts/limitations.md) | Query-time joins, `SELECT *`, ARRAY/MAP in joins, one-to-many rules, data profiling, YAML traps |
| [concepts/lifecycle-replace-semantics.md](concepts/lifecycle-replace-semantics.md) | `CREATE OR REPLACE` vs `ALTER VIEW ... AS` vs `COPY *`/`NOCOPY *`: owner, grants, tags, comments |
| [concepts/materialization.md](concepts/materialization.md) | Aggregated/unaggregated materializations, exact/rollup/unaggregated match, restrictions, refresh, billing |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/create-and-query.md](patterns/create-and-query.md) | Canonical `CREATE VIEW ... WITH METRICS LANGUAGE YAML` DDL + `MEASURE()` query + inspection |
| [patterns/ratio-and-filtered-measures.md](patterns/ratio-and-filtered-measures.md) | Ratio, composed (`MEASURE(a)/MEASURE(b)`) and filtered measures, NULL policy, distinct-count grain |
| [patterns/joins-star-schema.md](patterns/joins-star-schema.md) | Star/snowflake joins, `cardinality: one_to_many`, `rely.at_most_one_match`, multi-fact bridge |
| [patterns/window-measures.md](patterns/window-measures.md) | Trailing, cumulative, period-to-date, semiadditive, `offset` (YoY/MoM), numeric index for non-calendar periods |
| [patterns/parameters.md](patterns/parameters.md) | Parameterized metric views queried as table-valued functions |
| [patterns/agent-metadata.md](patterns/agent-metadata.md) | `display_name`, `format`, `synonyms`, `comment` for AI/BI dashboards and Genie Agents |
| [patterns/migrate-from-view.md](patterns/migrate-from-view.md) | Converting an aggregating view without losing consumers or grants |
| [patterns/deploy-with-bundles.md](patterns/deploy-with-bundles.md) | Metric views as code: bundle `sql_task`, dbt-databricks `metric_view`, Statement Execution API |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Metric View** | YAML definition registered in Unity Catalog (`CREATE VIEW ... WITH METRICS LANGUAGE YAML AS $$ ... $$`); `version` is required |
| **Field (dimension)** | Scalar expression used in `SELECT`/`WHERE`/`GROUP BY`; `fields:` is the preferred keyword, `dimensions:` is an accepted synonym |
| **Measure** | Aggregate expression with no fixed grain; evaluated with `MEASURE(name)` (`agg(name)` is a synonym on DBR 18.1+) |
| **Composed Measure** | Measure built from other measures: `MEASURE(total_revenue) / MEASURE(order_count)` |
| **Window Measure** | Rolling, cumulative, period-over-period or semiadditive measure (`window:` block); GA since 2026-10-06 |
| **Parameter** | Named value bound at query time (`mv(discount => 0.15)`); Public Preview, DBR 18.2+ |
| **Declarative Join** | Star/snowflake joins declared in YAML (`joins:`), LEFT OUTER, many-to-one by default |
| **Materialization** | Pre-computed materialized views kept by a managed Lakeflow pipeline; serverless + DBR 17.3+ |
| **Agent Metadata** | `display_name`, `format`, `synonyms` (+ `comment`); DBR 17.3+ and spec `1.1` |

## Integrations

| Surface | Notes |
|---------|-------|
| AI/BI dashboards | `MEASURE()` applied automatically; display names/formats surfaced; UC metric views open in the low-code UI and can be extended locally (not when they define parameters); local metric views are GA |
| Genie Agents (formerly Genie Spaces) | Synonyms are imported automatically; a Genie Agent accepts up to 50 tables/views/metric views as data sources |
| Alerts | Thresholds on measures |
| SQL editor / notebooks | Plain SQL with `MEASURE()` |
| JDBC/ODBC | Through the Thrift server (DBR 17.3+); metadata APIs expose the metric view structure |
| External BI (Power BI, Tableau, Sigma) | Custom SQL or BI compatibility mode (DBR 18.0+) |
| Excel Add-in, Google Sheets | Databricks connectors |
| OpenSharing (formerly Delta Sharing) | Sharing metric views is GA since 2026-09-24 (release notes); recipients query with `MEASURE()` on serverless. The Manage page still shows a Beta banner |
| Access control | `GRANT SELECT ON <metric_view> TO <principal>`; same UC hierarchy as any view |

## Product names used in this KB (2026)

| Former name | Current name | Since |
|-------------|--------------|-------|
| Delta Sharing | OpenSharing | 2026-06-10 |
| Genie Spaces | Genie Agents | 2026-07-08 |
| Databricks Asset Bundles (DABs) | Declarative Automation Bundles | — (bundles docs) |
| Lakehouse Monitoring | Data profiling (part of data quality monitoring) | — |
| `docs.../metric-views/` | `docs.../uc-semantics/metric-views/` | — |

**Field note:** a generator that emits `CREATE OR REPLACE` for metric views must re-apply tags and grants and verify them after each deploy — measured in production, replacing a metric view on a SQL warehouse dropped both. Read [lifecycle-replace-semantics.md](concepts/lifecycle-replace-semantics.md) before choosing a deploy flow.

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/fundamentals.md -> concepts/requirements-versions.md -> patterns/create-and-query.md |
| **Intermediate** | patterns/ratio-and-filtered-measures.md -> patterns/joins-star-schema.md -> patterns/agent-metadata.md |
| **Advanced** | patterns/window-measures.md -> patterns/parameters.md -> concepts/materialization.md -> concepts/limitations.md |
| **Operations** | concepts/lifecycle-replace-semantics.md -> patterns/deploy-with-bundles.md -> patterns/migrate-from-view.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| sql-optimizer | patterns/, concepts/limitations.md, concepts/materialization.md | Authoring/reviewing metric view YAML and measure SQL |
| schema-designer | patterns/joins-star-schema.md, concepts/fundamentals.md | Star/snowflake and multi-fact modeling |
| data-platform-engineer | concepts/requirements-versions.md, concepts/lifecycle-replace-semantics.md | DBR/permission prerequisites, grants and ownership |
| ci-cd-specialist | patterns/deploy-with-bundles.md, concepts/lifecycle-replace-semantics.md | Metric views in bundles and CI |
| build-agent | All files | Implementing a headless-BI / semantic layer |

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/query
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/manage
- https://docs.databricks.com/aws/en/uc-semantics/agent-metadata
- https://docs.databricks.com/aws/en/sql/language-manual/functions/measure
- https://docs.databricks.com/aws/en/release-notes/product/2026/june (Delta Sharing is now OpenSharing)
- https://docs.databricks.com/aws/en/release-notes/product/2026/july (Genie Spaces is now Genie Agents)
- https://docs.databricks.com/aws/en/release-notes/product/2026/september (OpenSharing of metric views GA)
- https://docs.databricks.com/aws/en/release-notes/product/2026/october (window measures GA)
- https://docs.databricks.com/aws/en/ai-bi/release-notes/2026 (local metric views GA, low-code UI, Genie Agent 50 sources)
- https://docs.databricks.com/aws/en/dev-tools/bundles/resources (Declarative Automation Bundles)
- https://docs.databricks.com/aws/en/data-governance/unity-catalog/data-quality-monitoring/ (data profiling)
