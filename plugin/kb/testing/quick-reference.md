# Python Testing Quick Reference

> **Purpose**: Fast lookup tables; code examples live in the linked files
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Versions (2026-10-17)

| Tool | Current | On Databricks Runtime 17.3 LTS |
|------|---------|--------------------------------|
| pytest | 9.1.1 | 8.3.5 preinstalled (no `strict`, subtests, `[tool.pytest]`) |
| pytest-asyncio | 1.4.0 (`event_loop` fixture removed in 1.0.0) | install it |
| Hypothesis | 6.170.1 | install it |
| PySpark | 4.2.0 | Spark 4.0.0 (ANSI on); `pyspark.testing` 4.0 API |

## pytest CLI

| Command | Purpose |
|---------|---------|
| `pytest -x` / `--sw` | Stop at first failure / stepwise |
| `pytest --lf` / `--ff` | Last failed only / failures first |
| `pytest -k "expr"` / `-m "marker"` | Filter by name / marker |
| `pytest -ra` | Summary of skips/xfails/errors |
| `pytest --co -q` | List test IDs only |
| `pytest -p no:cacheprovider` | No `.pytest_cache` (read-only filesystems) |
| `pytest --strict` | All strictness options (pytest 9+) |
| `pytest -n auto` | Parallel (pytest-xdist) |

## Fixture Scopes

| Scope | Lifecycle | Use Case |
|-------|-----------|----------|
| `function` | Per test (default) | Mutable state |
| `module` / `package` | Per file / package | Expensive read-only setup |
| `session` | Entire run | SparkSession, DB engine (`tmp_path_factory`) |

## Mock Patterns

| Pattern | Syntax |
|---------|--------|
| Patch where looked up | `@mock.patch("myapp.service.fetch")` |
| Plugin fixture | `mocker.patch(...)`, `mocker.spy(obj, "name")` (pytest-mock) |
| Return / sequence / raise | `m.return_value = X` / `m.side_effect = [1, 2]` / `m.side_effect = ValueError` |
| Async | `AsyncMock()`, `assert_awaited_once()` |
| Env / attribute | `monkeypatch.setenv("KEY", "v")`, `monkeypatch.setattr(obj, "attr", v)` |

## pytest-asyncio 1.x

| Pattern | Syntax |
|---------|--------|
| Async test (strict, default) | `@pytest.mark.asyncio` |
| Async fixture | `@pytest_asyncio.fixture` (plain `@pytest.fixture` only in auto mode) |
| Shared loop | `@pytest_asyncio.fixture(loop_scope="session", scope="session")` + `@pytest.mark.asyncio(loop_scope="session")` |
| Config | `asyncio_mode`, `asyncio_default_fixture_loop_scope`, `asyncio_default_test_loop_scope` |

## Hypothesis

| Pattern | Syntax |
|---------|--------|
| Generate / pin | `@given(st.integers())` / `@example(42)` |
| Tune / profile | `@settings(max_examples=500)` / `settings.register_profile(...)`, `load_profile(...)` |
| Fixtures | module/session scope only (function scope fails `function_scoped_fixture`) |

## Spark Assertions

| Need | Use |
|------|-----|
| Compare DataFrames | `pyspark.testing.assertDataFrameEqual(actual, expected_rows_or_df)` |
| Compare schemas | `pyspark.testing.assertSchemaEqual(actual.schema, expected)` |
| ANSI error check | `pytest.raises(PySparkException)` + `exc.value.getCondition()` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Async fixture with `@pytest.fixture` (strict mode) | `@pytest_asyncio.fixture` |
| Override `event_loop` | `loop_scope=` on marker/fixture |
| `tempfile.mktemp()` / `tmpdir` | `tmp_path` / `tmp_path_factory` |
| `filterwarnings = ["error", "ignore::DeprecationWarning"]` | Keep `DeprecationWarning` visible; ignore specific modules only |
| Local Spark tests with ANSI unset | Pin `spark.sql.ansi.enabled` to the target runtime |
| `pytest_addoption` in a sub-directory conftest | Top-level `tests/conftest.py` |

## Related Documentation

| Topic | Path |
|-------|------|
| pytest basics | `concepts/pytest-basics.md` |
| Async testing | `concepts/async-testing.md` |
| Spark testing | `patterns/spark-testing.md` |
| Full Index | `index.md` |

## Sources

- https://docs.pytest.org/en/stable/changelog.html
- https://pytest-asyncio.readthedocs.io/en/latest/reference/changelog.html
- https://hypothesis.readthedocs.io/en/latest/reference/api.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.testing.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://pytest-mock.readthedocs.io/en/latest/usage.html
