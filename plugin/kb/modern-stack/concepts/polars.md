# Polars

> **Purpose**: Rust DataFrame library with lazy evaluation, a streaming engine (default in 2.0), Arrow memory, plus Polars Cloud and Polars Distributed
> **Confidence**: HIGH
> **MCP Validated**: 2026-10-17

## Overview

Polars is a DataFrame library written in Rust, with Python, R and Node bindings. It uses Apache Arrow as its in-memory format. The **lazy API** (`LazyFrame`) builds a query plan that an optimizer rewrites before execution. The **eager API** (`DataFrame`) runs each step immediately. The current release is **Python Polars 2.0.0 (2026-10-06)**. The last 1.x release is 1.44.2 (2026-09-09).

## Polars 2.0 in Brief (2026-10-06)

- `LazyFrame.collect()` uses the **streaming engine by default** (`engine="auto"` resolves to streaming). The upgrade guide warns this "may silently impact the results of your pipelines".
- **Row order is no longer guaranteed** for `join`, `group_by` and `unpivot`. Use `maintain_order` (`"left"` on joins, `True` on `group_by`) or sort explicitly.
- **Out-of-core (spill to disk) is on by default**. Spilling starts near 80% of RAM with a 64 GB default disk budget. It covers sort, window functions and many expressions; joins and group-bys are planned.
- New native **`Map`** dtype. Arrow/Parquet/Iceberg maps load as `Map` instead of `List(Struct)`.
- Removed: `collect(streaming=...)` (deprecated in 1.25), `LazyFrame.profile()`, `LazyFrame.fetch()`, `melt()` (use `unpivot`), `with_row_count()` (use `with_row_index`), and the DataFrame Interchange Protocol.
- `pl.read_csv` / `pl.read_ipc` now dispatch to `scan_*(...).collect()`. Several casts are stricter: string-to-temporal casts are removed (use `str.to_date()` / `str.to_datetime()`).

## The Concept

```python
import polars as pl

lf = (
    pl.scan_parquet("data/orders/*.parquet")           # lazy: nothing is read yet
    .filter(pl.col("status") == "active")              # predicate pushdown
    .select("customer_id", "region", "net_amount")     # projection pushdown
    .group_by("region", maintain_order=True)           # 2.0: order is opt-in
    .agg(pl.col("net_amount").sum().alias("revenue"))
)

df = lf.collect()                         # 2.0: streaming engine by default
df = lf.collect(engine="streaming")       # explicit (1.x and 2.0)
df = lf.collect(engine="in-memory")       # pin the in-memory engine

# Stream results without materializing everything (1.34+)
for batch in lf.collect_batches():
    handle(batch)

lf.sink_parquet("out/revenue.parquet")    # streaming sink to file
print(lf.explain())                       # optimized plan
```

## Quick Reference

| Input | Output | Notes |
|-------|--------|-------|
| `pl.scan_parquet(...)` / `scan_csv` / `scan_ndjson` / `scan_iceberg` | `LazyFrame` | Prefer `scan_*` in pipelines |
| `lf.collect()` | `DataFrame` | Streaming engine by default in 2.0 |
| `lf.collect(engine="gpu")` | `DataFrame` | GPU engine; marked unstable |
| `lf.sink_parquet/csv/ipc(...)` | files | Streaming writes |
| `lf.collect_batches()` / `lf.sink_batches(...)` | batches | Added in 1.34.0 (2025-10-02) |
| `lf.sink_parquet(pl.PartitionBy("out", key="region"))` | Hive-style partitioned files | `PartitionBy` added in 1.37.0 (2026-01-10) |
| `pl.Decimal` | exact decimals | Stabilized in 1.35.0 (2025-10-26) |
| `df.to_arrow()` / `pl.from_arrow(...)` | Arrow table / DataFrame | Zero-copy where possible; DuckDB reads Polars frames directly |

## Version Notes (1.x, from GitHub release notes)

| Version | Date | Highlights |
|---------|------|------------|
| 1.34.0 | 2025-10-02 | `LazyFrame.{sink,collect}_batches`; skip files in `scan_iceberg` using metadata statistics |
| 1.35.0 | 2025-10-26 | Decimal stabilized; more native streaming group-by aggregations |
| 1.37.0 | 2026-01-10 | `pl.PartitionBy` API; `sink_delta`; new streaming CSV/NDJSON sink pipelines; Python 3.9 dropped |
| 1.44.2 | 2026-09-09 | Last 1.x release before 2.0 |
| 2.0.0 | 2026-10-06 | Streaming default, ordering opt-in, `Map`, out-of-core default |

## Polars Cloud and Polars Distributed

- **Polars Cloud**: GA on AWS since 2025-09-03. Its distributed engine was announced as an open/public beta, with workers running the OSS streaming engine. It runs the same LazyFrame code remotely:

```python
import polars as pl
import polars_cloud as pc

ctx = pc.ComputeContext(workspace="your-workspace", cpus=16, memory=64)
query = pl.scan_parquet("s3://my-dataset/").group_by("region").agg(pl.len())
query.remote(context=ctx).sink_parquet("s3://my-dst/")
```

- **Polars Distributed** (on-premises, 2026-06-03): the distributed engine deployed in your own infrastructure, mainly Kubernetes via a Helm chart. It includes query profiling and OpenLineage events. The 2.0 upgrade guide points to the Polars Cloud / On-Prem Query Profiler as the replacement for the removed `LazyFrame.profile()`.

## Common Mistakes

### Wrong

```python
lf.collect(streaming=True)            # deprecated in 1.25, removed in 2.0
left.join(right, on="k").collect()    # 2.0: output order is not guaranteed
pl.col("ts_str").cast(pl.Datetime)    # 2.0: string-to-temporal casts removed
```

### Correct

```python
lf.collect()                                   # 2.0 (streaming by default)
lf.collect(engine="streaming")                 # 1.x
left.join(right, on="k", maintain_order="left").collect()
pl.col("ts_str").str.to_datetime()
```

## Trade-offs

| Strength | Limitation |
|----------|------------|
| Lazy optimizer (predicate/projection/slice pushdown, CSE) | Expression API differs from pandas; not a drop-in replacement |
| Multi-threaded by default; Arrow memory; zero-copy with DuckDB/PyArrow | Single node in OSS; distributed needs Polars Cloud or Polars Distributed |
| Streaming engine + spill to disk for larger-than-RAM data | 2.0 changed defaults (engine, ordering, casts); test before upgrading |
| Fast release cadence | Pin the major version in production |

## Related

- [Polars Patterns](../patterns/polars-patterns.md)
- [DuckDB](duckdb.md)
- [Local-First Analytics](../patterns/local-first-analytics.md)

## Sources

- https://pola.rs/posts/release-polars-2/
- https://docs.pola.rs/releases/upgrade/2/
- https://github.com/pola-rs/polars/releases
- https://github.com/pola-rs/polars/releases/tag/py-1.34.0
- https://github.com/pola-rs/polars/releases/tag/py-1.35.0
- https://github.com/pola-rs/polars/releases/tag/py-1.37.0
- https://docs.pola.rs/api/python/version/1/reference/lazyframe/api/polars.LazyFrame.collect.html
- https://pola.rs/posts/polars-cloud-launch/
- https://docs.cloud.pola.rs/polars-cloud/
- https://pola.rs/posts/polars-distributed-available-on-kubernetes/
