# Star Schema

> **Purpose**: Complete star schema DDL — facts, dimensions, surrogate keys, degenerate dims, junk dims
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

The star schema is the foundation of dimensional modeling: a central fact table surrounded by denormalized dimension tables. Each dimension joins to the fact on a surrogate key. This pattern optimizes for read-heavy analytical queries with minimal joins. The first DDL block is ANSI-style; the Databricks/Delta version (informational keys, liquid clustering, identity caveat) is in the last section.

## The Pattern

```sql
-- ============================================================
-- Star Schema: E-Commerce Analytics
-- Grain: one row = one order line item
-- ============================================================

-- Conformed Date Dimension (shared across all facts)
CREATE TABLE dim_date (
    date_sk         INT PRIMARY KEY,           -- YYYYMMDD format
    full_date       DATE NOT NULL UNIQUE,
    day_of_week     VARCHAR(10) NOT NULL,
    day_of_month    INT NOT NULL,
    month_number    INT NOT NULL,
    month_name      VARCHAR(10) NOT NULL,
    quarter         INT NOT NULL,
    year            INT NOT NULL,
    is_weekend      BOOLEAN NOT NULL,
    is_holiday      BOOLEAN NOT NULL DEFAULT FALSE,
    fiscal_quarter  INT,
    fiscal_year     INT
);

-- Customer Dimension (SCD Type 2)
CREATE TABLE dim_customer (
    customer_sk     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id     VARCHAR(36) NOT NULL,
    customer_name   VARCHAR(200) NOT NULL,
    email           VARCHAR(200),
    segment         VARCHAR(50) NOT NULL,
    region          VARCHAR(100) NOT NULL,
    country         VARCHAR(100) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE NOT NULL DEFAULT '9999-12-31',
    is_current      BOOLEAN NOT NULL DEFAULT TRUE
);

-- Product Dimension (SCD Type 1 — overwrite)
CREATE TABLE dim_product (
    product_sk      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id      VARCHAR(36) NOT NULL UNIQUE,
    product_name    VARCHAR(300) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    subcategory     VARCHAR(100),
    brand           VARCHAR(100),
    unit_cost       DECIMAL(10,2)
);

-- Junk Dimension (low-cardinality flags combined)
CREATE TABLE dim_order_flags (
    flag_sk         INT PRIMARY KEY,
    is_gift         BOOLEAN NOT NULL,
    is_expedited    BOOLEAN NOT NULL,
    is_subscription BOOLEAN NOT NULL,
    payment_method  VARCHAR(20) NOT NULL   -- 'credit_card','paypal','bank_transfer'
);

-- Fact Table
CREATE TABLE fact_order_items (
    order_item_sk   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        VARCHAR(36) NOT NULL,       -- degenerate dimension
    date_sk         INT NOT NULL REFERENCES dim_date(date_sk),
    customer_sk     BIGINT NOT NULL REFERENCES dim_customer(customer_sk),
    product_sk      BIGINT NOT NULL REFERENCES dim_product(product_sk),
    flag_sk         INT NOT NULL REFERENCES dim_order_flags(flag_sk),
    quantity        INT NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    discount_pct    DECIMAL(5,2) NOT NULL DEFAULT 0,
    net_amount      DECIMAL(12,2) NOT NULL,     -- additive
    tax_amount      DECIMAL(12,2) NOT NULL,     -- additive
    shipping_cost   DECIMAL(12,2) NOT NULL DEFAULT 0
);
```

## Quick Reference

| Element | Purpose | Example |
|---------|---------|---------|
| Surrogate key | Stable PK, handles SCD | `customer_sk IDENTITY` |
| Natural key | Business identifier | `customer_id VARCHAR(36)` |
| Degenerate dimension | Dimension in fact table | `order_id` (no dim table needed) |
| Junk dimension | Low-cardinality flags | `is_gift`, `payment_method` combined |
| Conformed dimension | Shared across facts | `dim_date`, `dim_customer` |
| Bridge table | Many-to-many relationships | `bridge_order_promotions` |

## Common Mistakes

### Wrong

```sql
-- Natural key as FK in fact table — breaks when source changes IDs
CREATE TABLE fact_orders (
    customer_email VARCHAR(200) REFERENCES dim_customer(email)
);
```

### Correct

```sql
-- Surrogate key as FK — stable, immutable, performant
CREATE TABLE fact_orders (
    customer_sk BIGINT REFERENCES dim_customer(customer_sk)
);
```

## Star Schema in the Modern Stack (2025+)

Star schemas remain the gold standard for enterprise analytics, but modern tooling changes how they are built and consumed:

| Modern Tool | How It Enhances Star Schema |
|------------|---------------------------|
| **Semantic layer** (Unity Catalog metric views, dbt Semantic Layer, Cube, AtScale) | Measures defined once over the star; metric views declare star/snowflake joins in YAML — see [joins-star-schema](../../metric-views/patterns/joins-star-schema.md) |
| **Liquid clustering** (Delta OSS 3.1+; Databricks GA DBR 15.4 LTS, `CLUSTER BY AUTO` on UC managed tables) | Replaces Z-ORDER/manual partitioning on fact tables; up to 4 keys |
| **Identity columns** (Delta OSS 3.3+; Databricks SQL DBR 10.4 LTS+) | Auto-increment surrogate keys; **disable concurrent transactions** on the table |
| **Informational constraints** (Databricks, PK/FK GA DBR 15.2+) | Document relationships for BI/AI tools; `RELY` enables optimizer rewrites |
| **Hidden partitioning** (Iceberg) | Partition facts by date without exposing layout |
| **Materialized views** (Databricks, Snowflake) | Refresh pre-joined OBT marts from the star schema |
| **Data contracts** | Enforce dim/fact schemas at write time |

```sql
-- Star schema on Databricks (UC managed Delta tables)
-- Dimension loaded by a single writer: an identity surrogate key is acceptable
CREATE TABLE main.gold.dim_customer (
    customer_sk   BIGINT GENERATED ALWAYS AS IDENTITY,  -- disables concurrent transactions
    customer_id   STRING NOT NULL,
    customer_name STRING NOT NULL,
    segment       STRING NOT NULL,
    region        STRING NOT NULL,
    is_current    BOOLEAN NOT NULL,
    CONSTRAINT dim_customer_pk PRIMARY KEY (customer_sk) RELY   -- informational
) CLUSTER BY AUTO;

-- Fact written by parallel jobs: no identity column; deterministic key instead
CREATE TABLE main.gold.fact_orders (
    order_item_key STRING NOT NULL,          -- e.g. sha2(concat_ws('|', order_id, line_no), 256)
    order_id       STRING NOT NULL,
    customer_sk    BIGINT NOT NULL,
    product_sk     BIGINT NOT NULL,
    date_sk        INT    NOT NULL,
    quantity       INT    NOT NULL,
    net_amount     DECIMAL(12,2) NOT NULL,
    CONSTRAINT fact_orders_pk PRIMARY KEY (order_item_key),
    CONSTRAINT fact_orders_customer_fk FOREIGN KEY (customer_sk) REFERENCES main.gold.dim_customer
) CLUSTER BY (date_sk, customer_sk);          -- or CLUSTER BY AUTO
```

## Related

- [dimensional-modeling](../concepts/dimensional-modeling.md)
- [data-vault](data-vault.md)
- [one-big-table](one-big-table.md)
- [metric views](../../metric-views/index.md)

## Sources

- https://docs.databricks.com/aws/en/tables/constraints
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-table-constraint
- https://docs.databricks.com/aws/en/tables/features/generated-columns
- https://docs.databricks.com/aws/en/tables/clustering
- https://docs.databricks.com/aws/en/uc-semantics/metric-views
- https://docs.delta.io/latest/versioning.html
