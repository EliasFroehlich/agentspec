# Lakeflow Quick Reference

> **Purpose**: Fast lookup for Lakeflow Spark Declarative Pipelines (formerly DLT), Lakeflow Jobs and DABs
> **Confidence**: High
> **MCP Validated**: 2026-10-17

## Python (`from pyspark import pipelines as dp`)

| API | Creates / does |
|-----|----------------|
| `@dp.table` | Streaming table (streaming DataFrame) |
| `@dp.materialized_view` | Materialized view (batch DataFrame); `refresh_policy` |
| `@dp.temporary_view` | Pipeline-scoped view, not published |
| `dp.create_streaming_table(name)` | Target for flows / AUTO CDC |
| `@dp.append_flow(target, once=False)` | Extra append flow or one-time backfill |
| `dp.create_auto_cdc_flow(...)` | CDC into SCD1/SCD2 (replaces `apply_changes`) |
| `dp.create_auto_cdc_from_snapshot_flow(...)` | CDC from snapshots |
| `dp.create_sink(name, format, options)` | Delta/Kafka/custom sink |
| `@dp.foreach_batch_sink(name)` | Per-micro-batch custom logic |
| `@dp.expect` / `_or_drop` / `_or_fail` / `expect_all*` | Warn / drop / fail |
| `spark.read.table("x")`, `spark.readStream.table("x")` | Read pipeline datasets |

## SQL

| Statement | Purpose |
|-----------|---------|
| `CREATE OR REFRESH [PRIVATE] STREAMING TABLE` | Streaming table |
| `CREATE OR REFRESH [PRIVATE] MATERIALIZED VIEW ... REFRESH POLICY` | MV |
| `FROM STREAM read_files(path, format => ...)` | Auto Loader (no `cloudFiles.` prefix) |
| `CREATE FLOW f AS AUTO CDC INTO t FROM STREAM(s) KEYS(...) SEQUENCE BY ...` | CDC |
| `CREATE FLOW f AS INSERT [ONCE] INTO t BY NAME SELECT ...` | Append / backfill |
| `CONSTRAINT n EXPECT (e) [ON VIOLATION DROP ROW \| FAIL UPDATE]` | Expectation |
| `CLUSTER BY AUTO` | Liquid clustering (preferred over partitioning/Z-order) |
| `:param` / `${key}` | Pipeline parameter (Beta) / configuration value |

## Decision Matrix

| Use case | Choose |
|----------|--------|
| Append-only ingestion, files or Kafka | Streaming table + Auto Loader / Kafka |
| Joins, aggregates, sources with updates | Materialized view (serverless for incremental) |
| Change feed to current state / history | `AUTO CDC` SCD Type 1 / Type 2 |
| Many sources into one table | Append flows |
| Output to Kafka or external table | Sink + append flow |
| Orchestrate notebooks, scripts, SQL, pipelines | Lakeflow Job (bundle YAML) |
| New compute | Serverless (pipelines need `ADVANCED`) |
| Continuous in production | Continuous job wrapping a triggered pipeline |

## Platform Facts

| Item | Value |
|------|-------|
| Pipeline channels | `CURRENT` = DBR 18, `PREVIEW` = DBR 19 |
| Concurrent updates / workspace | 1,000 |
| Datasets / pipeline | 1,000 |
| Source files | 100 files or 50 folder entries (1,000 files) |
| Serverless standard mode startup | ~4-6 min (performance optimized is faster) |
| Databricks CLI | v1.21.0 (2026-10-14); Terraform engine removed in v1.20.0 |
| Minimum CLI for bundles | 0.218.0 (Python bundles 0.275.0) |

## Jobs YAML Essentials

| Key | Values |
|-----|--------|
| Task types | `notebook_task`, `spark_python_task`, `python_wheel_task`, `sql_task`, `pipeline_task`, `run_job_task`, `condition_task`, `for_each_task`, `dbt_task` |
| `run_if` | `ALL_SUCCESS`, `AT_LEAST_ONE_SUCCESS`, `NONE_FAILED`, `ALL_DONE`, `AT_LEAST_ONE_FAILED`, `ALL_FAILED` |
| Triggers | `schedule` (quartz cron), `trigger.file_arrival`, `trigger.table_update`, `continuous`, `triggers` (Beta) |
| Serverless | no cluster keys; `environment_key` for Python script/wheel/dbt; `performance_target` |
| References | `{{job.parameters.x}}`, `{{tasks.t.values.k}}`, `{{input}}`, `{{job.run_id}}` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `import dlt`, `@dlt.table` for MVs (legacy) | `from pyspark import pipelines as dp`, `@dp.materialized_view` |
| `APPLY CHANGES INTO` (legacy) | `CREATE FLOW ... AS AUTO CDC INTO` |
| `input_file_name()` (no longer supported in DBR 17.3 LTS+/UC) | `_metadata.file_path` / `_metadata.file_name` |
| `target:` in pipeline settings (legacy) | `catalog:` + `schema:` |
| `.count()` / `.collect()` in dataset code | Keep definitions lazy |
| `__IS_CURRENT` column | `__END_AT IS NULL` |
| Loop closures over a variable | Default argument `def f(x=item)` |
| `engine: terraform` on CLI 1.20+ | Direct engine (or pin 1.19.x) |

## Related Documentation

| Topic | Path |
|-------|------|
| Getting started | `concepts/getting-started.md` |
| Python API / migration | `patterns/python-decorators.md`, `concepts/python-module-migration.md` |
| SQL syntax | `patterns/sql-tables.md`, `patterns/sql-streaming.md` |
| CDC | `patterns/cdc-apply-changes.md` |
| Data quality | `patterns/expectations.md` |
| Jobs | `reference/lakeflow-jobs.md`, `patterns/jobs-bundle-yaml.md` |
| Bundles / CLI | `reference/bundles-cli.md`, `patterns/dabs-deployment.md` |
| Full index | `index.md` |

## Sources

- https://docs.databricks.com/aws/en/ldp/developer/python-ref ; https://docs.databricks.com/aws/en/ldp/developer/sql-dev ; https://docs.databricks.com/aws/en/ldp/limitations ; https://docs.databricks.com/aws/en/release-notes/dlt/
- https://docs.databricks.com/aws/en/jobs/ ; https://docs.databricks.com/aws/en/dev-tools/bundles/job-task-types ; https://docs.databricks.com/aws/en/dev-tools/bundles/direct ; https://github.com/databricks/cli/releases
