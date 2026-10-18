# Analytics Engineering

> **Purpose**: Code-first analytics tools: Malloy (+ Publisher), Evidence (Core + Studio), Observable Framework, and the analytics-as-code approach
> **Confidence**: MEDIUM
> **MCP Validated**: 2026-10-17

## Overview

Code-first analytics tools treat semantic models, reports and dashboards as version-controlled code reviewed like software. This page covers three of them, checked against their official docs on 2026-10-17. Transformation frameworks (dbt, SQLMesh) are in [sqlmesh](sqlmesh.md).

## Key Concepts

### Malloy: Semantic Modeling Language

Malloy separates the semantic model (sources, joins, dimensions, measures) from queries and compiles to SQL. The README lists these SQL engines: BigQuery, Snowflake, DuckDB, MotherDuck, PostgreSQL, MySQL, Trino, Presto and Databricks. The VS Code extension is the main authoring tool.

```malloy
source: orders is duckdb.table('data/orders.parquet') extend {
  join_one: customers is duckdb.table('data/customers.parquet')
    on customer_id = customers.customer_id

  dimension: order_month is order_date.month
  measure:
    total_revenue is net_amount.sum()
    avg_order_value is net_amount.avg()
    order_count is count()
}

// Query the semantic model
run: orders -> {
  group_by: customers.segment, order_month
  aggregate: total_revenue, order_count
  order_by: total_revenue desc
}
```

**Malloy Publisher** is "the open-source semantic model server for Malloy". It serves `.malloy` models through REST and MCP APIs, so apps, BI tools and AI agents can query one governed model. It also includes a local web UI to browse models and run queries.

**When to use**: reusable dimensions/measures shared by analysts and agents, or as an alternative to a BI-tool-specific semantic layer.

### Evidence: Reports as Markdown + SQL

Evidence today has two products:

- **Evidence Studio** (launched 2025-06-06): the hosted BI platform. It runs queries server-side on a managed engine and includes a cloud IDE with git, an AI agent, page/row-level access rules, SSO/SCIM and audit logging.
- **Evidence Core** (announced 2026-08-26): the MIT-licensed open-source framework underneath Studio. It has a single CLI binary (`evidence init`, `evidence dev`, `evidence validate`, `evidence serve`), metrics in YAML, live queries, and self-hosting as a static site or Docker image. Direct warehouse connections use a root `connection.yaml`. `evidence init --warehouse` scaffolds one for `snowflake` or `bigquery`.

Per the official migration guide, Core differs from **legacy Evidence** (Svelte components such as `<LineChart/>`, a `sources/` directory, DuckDB SQL run in the browser via WASM, a template-clone scaffold):

- Components use **Markdoc** syntax.
- SQL runs on a backend in the warehouse's native dialect (ClickHouse on Studio).
- A project supports one data warehouse.
- `evidence migrate` converts legacy projects.

````markdown
# Revenue Dashboard

```sql top_segments
select segment, sum(net_amount) as revenue
from analytics.orders
group by segment
order by revenue desc
```

{% line_chart data="analytics.orders" x="order_date" y="sum(net_amount)" date_grain="month" /%}
````

**When to use**: version-controlled reports reviewed in pull requests, against a warehouse (Core) or a hosted platform (Studio). For a DuckDB-only local stack, the legacy framework was the DuckDB-friendly option. Check current connector support before choosing Core.

### Observable Framework: Static Data Apps

Observable Framework is a free, open-source (ISC) static site generator for data apps, dashboards and reports. Front-end code is JavaScript. **Data loaders** written in any language (Python, R, SQL via CLI tools) precompute static data snapshots at build time, so pages load instantly.

**When to use**: custom interactive visualizations and public-facing data stories where you control the front end.

### Code-First vs Traditional BI

| Traditional BI | Code-First Analytics |
|----------------|----------------------|
| Drag-and-drop dashboards | SQL/Markdown/model files in git |
| Proprietary formats | Open formats (SQL, Markdown, YAML, Malloy) |
| Click to deploy | CI/CD deploy (build, validate, publish) |
| Semantic layer inside the BI tool | Semantic layer in code (Malloy, dbt metrics) |

## When to Use

| Tool | Best For | Runs On |
|------|----------|---------|
| **Malloy (+ Publisher)** | Reusable semantic models served to BI tools and agents | DuckDB, BigQuery, Snowflake, Postgres, Trino, Databricks, ... |
| **Evidence Core / Studio** | Version-controlled reports, self-hosted or hosted | One warehouse via a direct connector (Core); managed engine (Studio) |
| **Observable Framework** | Custom interactive visualizations, static hosting | Any data source via data loaders |
| **Warehouse-native BI** | Governed enterprise BI | Platform-specific |

## Common Mistakes

### Wrong

```bash
npx degit evidence-dev/template my-report     # legacy Evidence framework scaffold
```

### Correct

```bash
curl -fsSL https://evidence.studio/install.sh | sh   # Evidence CLI (macOS/Linux)
evidence init my-report
cd my-report && evidence dev
```

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Code-first | Version control, review, reproducibility | Harder for non-engineers to edit |
| Traditional BI | Accessible to business users | Harder to version and test |
| Hybrid | Engineers own models, business users explore | More tools to operate |

## Related

- [duckdb](../concepts/duckdb.md)
- [sqlmesh](../concepts/sqlmesh.md)
- [local-first-analytics](../patterns/local-first-analytics.md)

## Sources

- https://docs.malloydata.dev/documentation/language/source
- https://docs.malloydata.dev/documentation/language/join
- https://docs.malloydata.dev/documentation/language/order_by
- https://docs.malloydata.dev/documentation/language/expressions
- https://github.com/malloydata/malloy/
- https://evidence.dev/blog/evidence-core
- https://evidence.dev/blog/evidence-studio
- https://docs.evidence.dev/
- https://docs.evidence.dev/cli
- https://docs.evidence.dev/cli/commands
- https://docs.evidence.dev/cli/connections
- https://docs.evidence.dev/migration-guide
- https://docs.evidence.dev/core-concepts/components
- https://github.com/observablehq/framework
