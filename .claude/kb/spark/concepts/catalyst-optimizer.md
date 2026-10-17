# Catalyst Optimizer

> **Purpose**: How Spark plans queries — logical/physical plans, `explain` modes, predicate pushdown, codegen, AQE re-optimization — plus Spark 4 SQL features (pipe syntax, SQL UDFs, SQL scripting) with version notes
> **Confidence**: 0.92
> **MCP Validated**: 2026-10-17

## Overview

Catalyst turns a query into a parsed → analyzed → optimized logical plan and then a physical plan,
applying rule- and cost-based optimizations (predicate pushdown, column pruning, constant folding,
join selection, whole-stage codegen). **AQE** then re-optimizes the physical plan at runtime at each
shuffle/broadcast boundary using real statistics (join strategy switch, partition coalescing, skew
splitting, empty-relation propagation). AQE does **not** reorder joins — collect table statistics so
the cost-based optimizer can. On Databricks, Photon executes supported physical operators but
Catalyst still plans the query.

## The Concept

```python
df = (spark.read.table("main.sales.orders")
    .filter(F.col("order_date") >= "2026-01-01")
    .select("order_id", "customer_id", "amount")
    .join(spark.read.table("main.sales.customers"), "customer_id"))

df.explain()                    # = mode="simple": physical plan only
df.explain(mode="extended")     # parsed, analyzed, optimized logical plans + physical plan
df.explain(mode="formatted")    # physical plan outline + per-node details
df.explain(mode="cost")         # optimized logical plan with statistics (if available)
df.explain(mode="codegen")      # physical plan + generated code
```

With AQE, `DataFrame.explain()` shows the **initial** plan and the **current** plan under each
`AdaptiveSparkPlan` node (`isFinalPlan=true` once the query finishes). SQL `EXPLAIN` does not run
the query, so it only shows the initial plan — compare final plans in the Spark UI / query profile.

## Quick Reference

| Optimization | What it does | When it helps |
|--------------|--------------|---------------|
| Predicate pushdown | Moves filters into the scan | Parquet/Delta/Iceberg skip files and row groups |
| Column pruning | Reads only needed columns | Columnar formats |
| Data skipping (Delta) | Uses file-level min/max stats and clustering | Filters on clustering keys |
| Broadcast hash join | Ships the small side to every executor | Static: `autoBroadcastJoinThreshold` (OSS 10 MB); runtime on Databricks: AQE up to 30 MB |
| Whole-stage codegen | Fuses operators into one generated function | CPU-bound stages (JVM engine) |
| AQE coalesce | Merges small post-shuffle partitions | Many tiny tasks |
| AQE skew join | Splits skewed partitions (sort-merge / shuffle-hash joins only) | Straggler tasks |
| Dynamic partition pruning | Prunes partitions of the fact side from a dimension filter (`spark.sql.optimizer.dynamicPartitionPruning.enabled`, default `true`; not an AQE feature) | Star joins on partitioned tables |

## Common Mistakes

### Wrong

```python
# Filtering on a derived expression of the column hides the predicate from the scan
df = spark.read.table("main.sales.orders")
df = df.withColumn("year", F.year("order_date")).filter(F.col("year") == 2026)
```

### Correct

```python
# Range filter on the raw column — pushed down to the scan / data skipping
df = (spark.read.table("main.sales.orders")
    .filter((F.col("order_date") >= "2026-01-01") & (F.col("order_date") < "2027-01-01"))
    .withColumn("year", F.year("order_date")))
```

## Spark 4 SQL Features

### Pipe syntax (Spark 4.0+, Databricks DBR 16.2+)

```sql
FROM main.sales.orders
|> WHERE amount > 100
|> AGGREGATE SUM(amount) AS total GROUP BY region
|> ORDER BY total DESC;
-- Spark 4.2 / DBR 18.0+: `|` is accepted as an alias of `|>`, and aggregates are allowed in |> SELECT
```

### SQL user-defined functions (Spark 4.0+)

```sql
CREATE FUNCTION discount_price(price DECIMAL(12,2), pct DECIMAL(5,2))
RETURNS DECIMAL(12,2)
RETURN price * (1 - pct / 100);

SELECT order_id, discount_price(amount, 10) AS discounted FROM main.sales.orders;
```

### SQL scripting (OSS: flag in 4.0, GA in 4.1; Databricks DBR 16.3+)

```sql
BEGIN
  DECLARE threshold INT DEFAULT 1000;
  IF (SELECT COUNT(*) FROM main.staging.orders) > threshold THEN
    INSERT INTO main.sales.orders SELECT * FROM main.staging.orders;
  ELSE
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Below threshold';
  END IF;
END;
```

On DBR 17.3 LTS (Spark 4.0.0) scripting works because Databricks ships it from DBR 16.3; on OSS
Spark 4.0 it is behind a configuration flag. Databricks adds `CREATE PROCEDURE` in DBR 17.0+ and
cursors / `CONTINUE` handlers in DBR 18.1+.

## Related

- [partitioning](../concepts/partitioning.md)
- [photon](../concepts/photon.md)
- [performance-tuning](../patterns/performance-tuning.md)
- [version-matrix](../reference/version-matrix.md)

## Sources

- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.explain.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-explain
- https://docs.databricks.com/aws/en/optimizations/aqe
- https://spark.apache.org/docs/latest/sql-performance-tuning.html
- https://spark.apache.org/docs/latest/configuration.html
- https://spark.apache.org/docs/latest/sql-pipe-syntax.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-qry-pipeline
- https://spark.apache.org/docs/latest/sql-ref-syntax-ddl-create-sql-function.html
- https://spark.apache.org/docs/latest/control-flow/compound-stmt.html
- https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-scripting
- https://spark.apache.org/docs/4.0.0/sql-error-conditions.html
- https://spark.apache.org/releases/spark-release-4.1.0.html
