# Spark Knowledge Base

> **Purpose**: PySpark and Spark SQL for Apache Spark 4.x and Databricks — DataFrames, ANSI mode, Spark Connect, AQE/Photon tuning, Delta liquid clustering, streaming triggers
> **Version Coverage**: Apache Spark 4.0–4.2 (4.2.0 released 2026-07-14); Databricks Runtime 17.3 LTS (Spark 4.0.0), 18 LTS (Spark 4.1.0), 19 (Spark 4.2.0); Delta Lake 4.x (OSS 4.4.1). Features from Spark 4.1/4.2 are flagged as not available on DBR 17.3.
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/dataframe-api.md](concepts/dataframe-api.md) | Core transformations, Column expressions, VARIANT, Python data sources, plotting |
| [concepts/ansi-mode.md](concepts/ansi-mode.md) | ANSI default (Spark 4.0 / DBR 17.0+), `try_*` functions, coercion traps |
| [concepts/spark-connect.md](concepts/spark-connect.md) | Where Connect runs on Databricks and what changes (lazy analysis, no RDD/JVM handles) |
| [concepts/partitioning.md](concepts/partitioning.md) | repartition vs coalesce, AQE, `shuffle.partitions=auto`, clustering vs partitionBy |
| [concepts/catalyst-optimizer.md](concepts/catalyst-optimizer.md) | Plans, `explain` modes, AQE re-optimization, pipe syntax, SQL UDFs, scripting |
| [concepts/photon.md](concepts/photon.md) | Databricks vectorized engine: defaults, fallbacks, when to turn it off |
| [concepts/real-time-mode.md](concepts/real-time-mode.md) | Streaming triggers incl. real-time mode prerequisites (DBR 16.4 LTS+, OSS 4.1+) |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/read-write-patterns.md](patterns/read-write-patterns.md) | UC/Delta table I/O, replaceWhere, Parquet files, Iceberg transforms, `_metadata`, VARIANT ingest |
| [patterns/delta-integration.md](patterns/delta-integration.md) | Liquid clustering, `CLUSTER BY AUTO`, MERGE / SCD2, CDF, UniForm, predictive optimization |
| [patterns/performance-tuning.md](patterns/performance-tuning.md) | AQE, broadcast thresholds, skew, caching rules, write sizing, OSS vs Databricks settings |
| [patterns/python-udfs.md](patterns/python-udfs.md) | Built-ins vs `udf(useArrow)` vs `pandas_udf` vs `arrow_udf` (4.1+) |
| [patterns/window-functions.md](patterns/window-functions.md) | ROW_NUMBER, RANK, LAG/LEAD, running totals, sessionization |
| [patterns/transform-with-state.md](patterns/transform-with-state.md) | `StatefulProcessor` + `transformWithStateInPandas` with timers and TTL |

### Reference

| File | Purpose |
|------|---------|
| [reference/version-matrix.md](reference/version-matrix.md) | Feature × Spark release × DBR 17.3 LTS / 18 LTS / 19 availability |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **ANSI mode** | On by default since Spark 4.0 / DBR 17.0: invalid casts, overflow, ÷0 and bad indexes fail; use `try_*` |
| **Spark Connect** | gRPC client/server; used by Databricks standard access mode, serverless and Databricks Connect |
| **AQE + auto shuffle** | Runtime re-optimization; on Databricks set `spark.sql.shuffle.partitions=auto` |
| **Photon** | Databricks native engine; UDFs/RDD/stateful streaming fall back to the JVM |
| **Liquid clustering** | Replaces partitioning and ZORDER for Delta; `CLUSTER BY AUTO` with predictive optimization |
| **VARIANT** | Semi-structured type (Spark 4.0, GA 4.1; Databricks DBR 15.4+) read with `variant_get` |
| **SQL enhancements** | Pipe syntax (4.0), SQL UDFs (4.0), scripting (GA 4.1; DBR 16.3+), recursive CTE (4.1; DBR 17.0+), QUALIFY (4.2; DBR 10.4+) |
| **Python UDF flavours** | Arrow-optimized `udf` (3.5+), `pandas_udf`, `arrow_udf` (4.1+) |
| **transformWithState** | Arbitrary stateful operator (4.0+) replacing `mapGroupsWithState` / `applyInPandasWithState` |
| **Real-time mode** | Explicit low-latency trigger with strict compute rules (not automatic) |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/dataframe-api.md, concepts/ansi-mode.md |
| **Intermediate** | patterns/read-write-patterns.md, patterns/window-functions.md, concepts/spark-connect.md |
| **Advanced** | patterns/performance-tuning.md, concepts/photon.md, patterns/delta-integration.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| spark-engineer | All files | DataFrame transforms, I/O, ANSI-safe code |
| spark-specialist | patterns/performance-tuning.md, concepts/photon.md, reference/version-matrix.md | Architecture, tuning, version decisions |
| spark-performance-analyzer | patterns/performance-tuning.md, concepts/partitioning.md, concepts/photon.md | Cost and runtime optimization |
| spark-troubleshooter | concepts/ansi-mode.md, concepts/spark-connect.md, patterns/performance-tuning.md | Failure diagnosis |
| build-agent | patterns/read-write-patterns.md | Verification via a notebook/job task or Databricks Connect (spark-submit tasks are not supported on standard access mode) |

## Sources

- https://spark.apache.org/downloads.html
- https://docs.databricks.com/aws/en/release-notes/runtime/
- https://github.com/delta-io/delta/releases/tag/v4.4.1
- https://docs.databricks.com/aws/en/compute/standard-limitations
