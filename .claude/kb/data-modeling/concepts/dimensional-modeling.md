# Dimensional Modeling

> **Purpose**: Kimball methodology — facts, dimensions, conformed dims, bus matrix, grain definition
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Dimensional modeling (Kimball) organizes data into fact tables (measures/events) and dimension tables (context/attributes). The grain — what one row represents — is the single most important decision. Facts are numeric and additive; dimensions are descriptive and filterable. The DDL below is ANSI-style; on Databricks/Delta, `PRIMARY KEY`/`REFERENCES` are **informational** (not enforced) and identity columns disable concurrent writes — see the table at the end.

## The Concept

```sql
-- Grain: one row = one order line item
CREATE TABLE fact_order_items (
    order_item_sk   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        VARCHAR(36)   NOT NULL,  -- degenerate dimension
    customer_sk     BIGINT        NOT NULL REFERENCES dim_customer(customer_sk),
    product_sk      BIGINT        NOT NULL REFERENCES dim_product(product_sk),
    date_sk         INT           NOT NULL REFERENCES dim_date(date_sk),
    quantity        INT           NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    discount_amount DECIMAL(12,2) NOT NULL DEFAULT 0,
    net_amount      DECIMAL(12,2) NOT NULL,  -- additive measure
    created_at      TIMESTAMP     NOT NULL
);

CREATE TABLE dim_customer (
    customer_sk     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     VARCHAR(36)   NOT NULL,  -- natural key
    customer_name   VARCHAR(200)  NOT NULL,
    email           VARCHAR(200),
    segment         VARCHAR(50),             -- e.g. 'Enterprise', 'SMB'
    region          VARCHAR(50),             -- conformed dimension attribute
    effective_from  DATE          NOT NULL,
    effective_to    DATE          NOT NULL DEFAULT '9999-12-31',
    is_current      BOOLEAN       NOT NULL DEFAULT TRUE
);
```

## Quick Reference

| Concept | Definition | Example |
|---------|-----------|---------|
| Grain | What one row represents | "One order line item" |
| Fact | Numeric, measurable event | revenue, quantity, duration |
| Dimension | Descriptive context | customer, product, date |
| Surrogate key | System-generated PK | customer_sk (IDENTITY) |
| Natural key | Business identifier | customer_id (from source) |
| Degenerate dimension | Dimension stored in fact | order_id (no separate table) |
| Conformed dimension | Shared across facts | dim_date, dim_customer |
| Bus matrix | Facts × Dimensions grid | Planning tool for warehouse |
| Additive measure | SUM-able across all dims | revenue, quantity |
| Semi-additive | SUM-able across some dims | account_balance (not time) |
| Non-additive | Cannot SUM | unit_price, ratio |

## Common Mistakes

### Wrong

```sql
-- No grain defined, mixing granularities
CREATE TABLE sales_data (
    customer_name  VARCHAR(200),
    total_revenue  DECIMAL(12,2),  -- aggregated? per-order? per-item?
    product_name   VARCHAR(200),
    order_date     DATE
);
```

### Correct

```sql
-- Grain explicitly stated: one row = one order line item
-- Surrogate keys for joins, natural key preserved
-- Measures are atomic (not pre-aggregated)
CREATE TABLE fact_order_items (
    order_item_sk  BIGINT PRIMARY KEY,
    customer_sk    BIGINT NOT NULL,
    product_sk     BIGINT NOT NULL,
    date_sk        INT NOT NULL,
    quantity       INT NOT NULL,
    unit_price     DECIMAL(12,2) NOT NULL,
    net_amount     DECIMAL(12,2) NOT NULL
);
```

## Dimensional Modeling in 2025+

Kimball dimensional modeling remains the dominant approach for analytics, but the modern data stack introduces key enhancements:

| Evolution | Impact |
|----------|--------|
| **Semantic layers** (Unity Catalog metric views, dbt Semantic Layer, Cube) | Define measures once over the star schema; see [metric views](../../metric-views/index.md) |
| **OBT as a mart** | Build denormalized tables from star schema for specific dashboards |
| **Data Vault + Dim marts** | Data Vault for raw integration, dimensional marts for consumption |
| **Liquid clustering** (Delta OSS 3.1+, Databricks GA DBR 15.4 LTS; `CLUSTER BY AUTO` on UC managed) | Replaces manual partitioning and Z-ORDER on fact tables |
| **Identity columns** (Delta OSS 3.3+ API, SQL DDL 4.4+; Databricks SQL DBR 10.4 LTS+) | Auto-increment surrogate keys — but they **disable concurrent transactions** on the table |
| **Data contracts** | Enforce grain, types, and constraints at write time |

| Constraint on Databricks/Delta | Behavior |
|---|---|
| `NOT NULL`, `CHECK` | Enforced: violating writes fail |
| `PRIMARY KEY`, `FOREIGN KEY` | Informational (GA DBR 15.2+); `RELY` lets the optimizer use them (Photon) — you must keep them true |
| `UNIQUE` | Informational, Public Preview (DBR 18.2+) |

The Kimball bus matrix remains essential for planning which conformed dimensions are shared across facts. Data Mesh teams can own their domain's star schemas while sharing conformed dimensions via a "shared" or "master" domain.

## Related

- [scd-types](scd-types.md)
- [star-schema pattern](../patterns/star-schema.md)
- [one-big-table pattern](../patterns/one-big-table.md)

## Sources

- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-constraint
- https://docs.databricks.com/aws/en/tables/features/generated-columns
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/uc-semantics/metric-views
- https://github.com/delta-io/delta/releases
