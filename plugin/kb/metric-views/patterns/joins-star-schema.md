# Joins / Star, Snowflake and Multi-Grain Schemas

> **Purpose**: Declarative `joins:` for star and multi-level snowflake schemas, one-to-many fact joins, `rely` optimization, SQL-query join sources and multi-fact bridges
> **MCP Validated**: 2026-10-17

## When to Use

- The metric view's attributes live in dimension tables separate from the fact `source`
- Multi-level hierarchies (customer → nation → region) that dashboards roll up through
- Facts at a different grain than the source (orders per customer, targets per region) — `one_to_many` or a bridge
- Never for query-time joins: join a CTE over the metric view instead

## Implementation

```yaml
version: 1.1
source: main.sales.fact_sales
comment: "Sales analytics with customer and product dimensions"

joins:
  - name: customer                                   # star schema: single level
    source: main.sales.dim_customer
    'on': source.customer_id = customer.customer_id  # quoted: YAML 1.1 tools read bare `on` as boolean
    rely:
      at_most_one_match: true                        # DBR 18.1+; NOT validated at runtime
    joins:                                           # snowflake: nested join (DBR 17.3+)
      - name: region
        source: main.sales.dim_region
        'on': customer.region_id = region.region_id
  - name: product
    source: main.sales.dim_product
    using: [product_id]                              # `using` OR `on`, not both

fields:
  - name: Customer Segment
    expr: customer.segment
  - name: Region
    expr: customer.region.name                       # dot-path through nested joins
  - name: Product Category
    expr: product.category
  - name: Sale Month
    expr: DATE_TRUNC('MONTH', source.sale_date)

measures:
  - name: Total Revenue
    expr: SUM(source.amount)
  - name: Unique Customers
    expr: COUNT(DISTINCT source.customer_id)
```

### One-to-many (fact source joined to a dimensional spine) — DBR 18.1+, spec 1.1

```yaml
version: 1.1
source: main.sales.customers                 # spine: one row per customer
joins:
  - name: orders
    source: main.sales.orders
    'on': orders.customer_id = source.customer_id
    cardinality: one_to_many                 # aggregated independently at the source grain
fields:
  - name: customer_name
    expr: customer_name                      # one_to_many columns can't be fields
measures:
  - name: customer_count
    expr: COUNT(*)
  - name: order_count
    expr: COUNT(orders.order_id)
  - name: orders_per_customer
    expr: MEASURE(order_count) / NULLIF(MEASURE(customer_count), 0)
```

### Join a SQL query (resolve ARRAY/MAP before joining)

```yaml
joins:
  - name: orders
    source: |
      SELECT o_orderkey, o_custkey,
             aggregate(line_items, 0, (acc, x) -> acc + x.quantity) AS total_quantity
      FROM main.sales.orders_with_items
    'on': orders.o_custkey = source.c_custkey
    cardinality: one_to_many
```

### Multi-fact bridge (two facts that share dimensions)

```yaml
version: 1.1
source: SELECT * FROM main.ref.part CROSS JOIN main.ref.supplier   # valid dimension combinations
joins:
  - name: shipments
    source: main.sales.lineitem
    'on': shipments.l_partkey = source.p_partkey AND shipments.l_suppkey = source.s_suppkey
    cardinality: one_to_many
  - name: supply
    source: main.ref.partsupp
    'on': supply.ps_partkey = source.p_partkey AND supply.ps_suppkey = source.s_suppkey
    cardinality: one_to_many
fields:
  - name: part_name
    expr: p_name
measures:
  - name: shipped_qty
    expr: SUM(shipments.l_quantity)
  - name: available_qty
    expr: SUM(supply.ps_availqty)
```

Use `SELECT DISTINCT <keys> FROM <fact>` instead of `CROSS JOIN` when only observed combinations are valid.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `on` / `using` | one required | `on`: boolean expression (unprefixed names default to the joined table); `using`: shared column names |
| `source` (join) | — | Three-part table name or a SQL query |
| `cardinality` | `many_to_one` | `one_to_many` (DBR 18.1+, spec 1.1): descendants must also be `one_to_many`; sibling joins may mix |
| `rely.at_most_one_match` | `false` | Lets the engine skip joins / scan less; wrong results on fan-out |
| Semantics | LEFT OUTER | All source rows kept; unmatched dimension columns are `NULL` |
| Many-to-many | first row | The first matching row is used silently — fix the grain instead |
| ARRAY/MAP columns | disallowed | Joined tables can't have them; flatten (`explode`) or aggregate in the join `source` |
| Materialization | — | Any `one_to_many` join → exact match only (no rollup) |

## Example Usage

```sql
-- Revenue rolls up through the nested join automatically
SELECT `Region`, MEASURE(`Total Revenue`) AS revenue
FROM main.sales.sales_analytics
GROUP BY ALL
ORDER BY revenue DESC
```

## See Also

- [Limitations](../concepts/limitations.md)
- [Materialization](../concepts/materialization.md)
- [Fundamentals](../concepts/fundamentals.md)
- [Create and Query](create-and-query.md)

## Sources

- https://docs.databricks.com/aws/en/uc-semantics/metric-views/joins (cardinality, `rely`, nested and sibling one-to-many, bridge)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/yaml-reference (join fields, snowflake, one-to-many rules)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/basic-modeling (ARRAY/MAP, SQL-query join source)
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/feature-availability
- https://docs.databricks.com/aws/en/uc-semantics/metric-views/materialization (one-to-many → exact match)
- https://yaml.org/type/bool.html (`on` is a YAML 1.1 boolean)
