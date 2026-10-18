# Python Testing Knowledge Base

> **Purpose**: pytest 9, fixtures, mocking, pytest-asyncio 1.x, Hypothesis, PySpark 4 testing, CI configuration
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Runtime Check First

pytest is at **9.1.1** (2026-06-19) and pytest-asyncio at **1.4.0** (no more `event_loop` fixture).
Databricks Runtime 17.3 LTS preinstalls **pytest 8.3.5** and runs **Spark 4.0.0 with ANSI mode on**:
pytest 9 features (`[tool.pytest]`, `strict`, subtests) need `%pip install "pytest>=9"` there, and
Spark tests must pin `spark.sql.ansi.enabled` to the target runtime.

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/pytest-basics.md](concepts/pytest-basics.md) | Discovery, markers, raises/RaisesGroup, subtests, strict mode, CLI |
| [concepts/fixtures.md](concepts/fixtures.md) | Scope, yield teardown, conftest.py, tmp_path, monkeypatch |
| [concepts/mocking.md](concepts/mocking.md) | unittest.mock, monkeypatch, pytest-mock `mocker`, patch targets |
| [concepts/parametrize.md](concepts/parametrize.md) | Parametrized tests, IDs, data-driven testing |
| [concepts/async-testing.md](concepts/async-testing.md) | pytest-asyncio 1.x strict/auto modes, loop scopes, AsyncMock |
| [concepts/property-based-testing.md](concepts/property-based-testing.md) | Hypothesis strategies, examples, profiles, health checks |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/unit-test-patterns.md](patterns/unit-test-patterns.md) | Unit test structure, AAA pattern, edge cases |
| [patterns/integration-tests.md](patterns/integration-tests.md) | DB rollback, in-process APIs, files, opt-in integration runs |
| [patterns/spark-testing.md](patterns/spark-testing.md) | SparkSession fixture, `pyspark.testing` assertions, ANSI mode |
| [patterns/fixture-factories.md](patterns/fixture-factories.md) | Factory functions for reusable test data |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/test-config.yaml](specs/test-config.yaml) | pytest 9 configuration spec, marker registry, plugins |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **pytest 9** | Native `[tool.pytest]` TOML, `strict = true`, built-in `subtests`; 8.4 added `RaisesGroup` |
| **Fixtures** | Dependency injection with scopes; `tmp_path`/`tmp_path_factory` for files |
| **Mocking** | Patch where the name is looked up; `mocker` from pytest-mock auto-undoes patches |
| **Parametrize** | Same test logic over many inputs with readable IDs |
| **pytest-asyncio 1.x** | Strict mode default; `@pytest_asyncio.fixture`; `loop_scope` instead of `event_loop` |
| **Hypothesis** | Property-based tests; function-scoped fixtures fail a health check |
| **pyspark.testing** | `assertDataFrameEqual` / `assertSchemaEqual` (3.5+; more options in 4.0) |
| **ANSI mode** | On by default since Spark 4.0: bad casts and `/ 0` raise; use `try_*` functions |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/pytest-basics.md, concepts/fixtures.md |
| **Intermediate** | concepts/mocking.md, concepts/parametrize.md, concepts/async-testing.md, patterns/unit-test-patterns.md |
| **Advanced** | concepts/property-based-testing.md, patterns/integration-tests.md, patterns/spark-testing.md, patterns/fixture-factories.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| test-generator | patterns/unit-test-patterns.md, concepts/fixtures.md | Generate pytest tests for Python modules |
| test-generator | patterns/spark-testing.md | Generate PySpark DataFrame test suites |
| test-generator | patterns/integration-tests.md, concepts/async-testing.md | Generate async and integration tests |
| code-reviewer | specs/test-config.yaml | Check test configuration and markers |

## Sources

- https://docs.pytest.org/en/stable/changelog.html
- https://pytest-asyncio.readthedocs.io/en/latest/reference/changelog.html
- https://hypothesis.readthedocs.io/en/latest/changelog.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.testing.html
- https://spark.apache.org/docs/latest/sql-migration-guide.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
