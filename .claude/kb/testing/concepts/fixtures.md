# Fixtures

> **Purpose**: pytest fixture patterns -- scope, yield teardown, conftest.py, built-in tmp_path/monkeypatch, async fixtures
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Fixtures are pytest's dependency injection: a test names a fixture as a parameter, pytest builds it,
caches it for its **scope**, and runs the code after `yield` as teardown. Fixtures compose (a fixture
can request other fixtures) and are shared through `conftest.py` files.

## The Pattern

```python
import pytest


@pytest.fixture
def sample_invoice() -> dict:
    """Function scope (default): fresh object for every test."""
    return {"invoice_number": "INV-001", "vendor_name": "Acme Corp", "total_amount": 1500.0}


@pytest.fixture(scope="session")
def db_engine(tmp_path_factory):
    """One engine per run; tmp_path_factory is the session-scoped temp-dir helper."""
    from sqlalchemy import create_engine
    engine = create_engine(f"sqlite:///{tmp_path_factory.mktemp('db') / 'test.db'}")
    yield engine
    engine.dispose()                 # teardown runs even if tests failed


@pytest.fixture
def db_conn(db_engine):
    """Composes db_engine; rolls back so tests never see each other's writes."""
    with db_engine.connect() as conn:
        tx = conn.begin()
        yield conn
        tx.rollback()
```

## Fixture Scopes

| Scope | Created | Destroyed | Use Case |
|-------|---------|-----------|----------|
| `function` | Per test (default) | After the test | Mutable, isolated state |
| `class` | Per test class | After the class | Shared across methods |
| `module` | Per .py file | After the module | Expensive, read-only setup |
| `package` | Per package | After the package | Package-level resources |
| `session` | Once per run | At the end | SparkSession, DB engines |

## Built-in Fixtures

| Fixture | Purpose |
|---------|---------|
| `tmp_path` / `tmp_path_factory` | Unique `pathlib.Path` temp dir per test / per session (`tmpdir` is deprecated) |
| `monkeypatch` | `setattr`, `setenv`, `delenv`, `setitem`, `chdir`, undone after the test |
| `capsys` / `caplog` | Captured stdout/stderr / log records |
| `request` | Fixture introspection; `request.param` for parametrized fixtures |
| `subtests` | Independent sub-assertions in one test (pytest 9.0+) |

## conftest.py

```python
# tests/conftest.py -- fixtures here are visible to every test below this directory
import pytest


@pytest.fixture(autouse=True)
def test_environment(monkeypatch):
    """Autouse: applied to every test without being requested."""
    monkeypatch.setenv("ENV", "test")
    monkeypatch.setenv("LOG_LEVEL", "DEBUG")
```

## Files Without Leaks

```python
def test_reads_csv(tmp_path):
    path = tmp_path / "data.csv"                       # unique per test; pytest keeps the last 3 runs
    path.write_text("id,name,amount\n1,Acme,100.00\n")
    assert load_csv(path)[0]["name"] == "Acme"
```

`tempfile.mktemp()` is deprecated and unsafe (another process can create the file between
naming and opening); use `tmp_path`, or `tempfile.mkstemp()` outside pytest.

## Common Mistakes

### Wrong

```python
@pytest.fixture(scope="module")          # one list shared by every test in the module
def shared_list():
    return []

def test_a(shared_list):
    shared_list.append(1)
    assert len(shared_list) == 1

def test_b(shared_list):
    assert len(shared_list) == 0         # FAILS after test_a; order-dependent
```

### Correct

```python
@pytest.fixture                          # function scope: a new list per test
def fresh_list():
    return []

def test_b(fresh_list):
    assert len(fresh_list) == 0          # always passes
```

## Async Fixtures

Async fixtures need pytest-asyncio and, in its default strict mode, `@pytest_asyncio.fixture`
(not `@pytest.fixture`); session-wide ones also need `loop_scope="session"`. See
[Async Testing](async-testing.md).

```python
import pytest_asyncio


@pytest_asyncio.fixture(loop_scope="session", scope="session")
async def db_pool():
    pool = await create_pool("postgresql://localhost/test")
    yield pool
    await pool.close()
```

## Related

- [Fixture Factories](../patterns/fixture-factories.md)
- [Async Testing](../concepts/async-testing.md)
- [Integration Tests](../patterns/integration-tests.md)

## Sources

- https://docs.pytest.org/en/stable/how-to/fixtures.html
- https://docs.pytest.org/en/stable/how-to/tmp_path.html
- https://docs.pytest.org/en/stable/reference/reference.html
- https://docs.python.org/3/library/tempfile.html
- https://pytest-asyncio.readthedocs.io/en/latest/reference/changelog.html
