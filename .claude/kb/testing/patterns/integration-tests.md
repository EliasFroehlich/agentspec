# Integration Test Patterns

> **Purpose**: Verify interactions across boundaries -- databases, HTTP APIs, files, async services -- with isolated, self-cleaning fixtures
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## When to Use

- Verifying that components work together correctly
- Testing real database queries and transactions
- Validating API request/response cycles in-process
- Verifying file I/O end to end
- Gating slow tests behind an opt-in flag in CI

## Implementation

```python
import json

import pytest


# --- Database: one engine per session, one rolled-back transaction per test ---

@pytest.fixture(scope="session")
def db_engine(tmp_path_factory):
    from sqlalchemy import create_engine
    db_file = tmp_path_factory.mktemp("db") / "integration.db"   # under pytest basetemp (last 3 runs kept)
    engine = create_engine(f"sqlite:///{db_file}")
    yield engine
    engine.dispose()


@pytest.fixture
def db_session(db_engine):
    from sqlalchemy.orm import Session
    with db_engine.connect() as connection:
        transaction = connection.begin()
        session = Session(bind=connection)
        yield session
        session.close()
        transaction.rollback()


@pytest.mark.integration
class TestInvoiceRepository:
    def test_save_and_retrieve(self, db_session):
        repo = InvoiceRepository(db_session)
        repo.save(Invoice(number="INV-001", vendor="Acme", amount=100.0))

        result = repo.find_by_number("INV-001")

        assert result is not None
        assert result.vendor == "Acme"

    def test_find_nonexistent_returns_none(self, db_session):
        assert InvoiceRepository(db_session).find_by_number("NOPE") is None


# --- HTTP API: in-process client, no network ---

@pytest.fixture(scope="module")
def api_client():
    from fastapi.testclient import TestClient
    from myapp.main import app
    with TestClient(app) as client:     # runs startup/shutdown events
        yield client


@pytest.mark.integration
class TestInvoiceAPI:
    def test_create_invoice_returns_201(self, api_client):
        payload = {"invoice_number": "INV-100", "vendor": "Test Corp", "amount": 500.00}
        response = api_client.post("/invoices", json=payload)
        assert response.status_code == 201
        assert response.json()["invoice_number"] == "INV-100"

    def test_missing_fields_return_422(self, api_client):
        assert api_client.post("/invoices", json={"vendor": "X"}).status_code == 422


# --- Files: tmp_path gives a unique directory per test ---

@pytest.mark.integration
def test_process_files_end_to_end(tmp_path):
    input_path = tmp_path / "invoices.json"
    output_path = tmp_path / "output.json"
    input_path.write_text(json.dumps([{"id": 1, "amount": 100}, {"id": 2, "amount": 200}]))

    process_invoice_file(input_path, output_path)

    results = json.loads(output_path.read_text())
    assert len(results) == 2
    assert all(r["processed"] for r in results)
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `@pytest.mark.integration` | -- | Register in `markers`; strict mode rejects unknown markers |
| `scope="session"` / `"module"` | -- | Share expensive, read-only resources |
| `--run-integration` | disabled | Custom flag (conftest below) to opt in |

## Isolation Strategies

| Strategy | When to Use | Trade-off |
|----------|-------------|-----------|
| Transaction rollback | Database tests | Fast, no residue |
| `tmp_path` / `tmp_path_factory` | File I/O, SQLite files | Unique dirs; pytest keeps only the last 3 runs |
| Containers | Full service tests | Slow, realistic |
| Mock only the external edge | Third-party APIs | Balanced isolation |

## conftest.py for Opt-In Integration Tests

`pytest_addoption` is only called for *initial* conftests, so put it in the top-level
`tests/conftest.py` (with `testpaths = ["tests"]`); in `tests/integration/conftest.py` the run
fails with "unrecognized arguments: --run-integration".

```python
# tests/conftest.py
import pytest


def pytest_addoption(parser):
    parser.addoption("--run-integration", action="store_true", default=False,
                     help="Run integration tests")


def pytest_collection_modifyitems(config, items):
    if config.getoption("--run-integration"):
        return
    skip = pytest.mark.skip(reason="Need --run-integration to run")
    for item in items:
        if "integration" in item.keywords:
            item.add_marker(skip)
```

## Async Integration Tests (pytest-asyncio 1.x)

```python
import httpx
import pytest
import pytest_asyncio


@pytest_asyncio.fixture                  # strict mode: @pytest.fixture would error under pytest 9
async def async_api_client():
    from myapp.main import app
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as client:
        yield client


@pytest.mark.asyncio
@pytest.mark.integration
async def test_async_health_check(async_api_client):
    response = await async_api_client.get("/health")
    assert response.json()["status"] == "ok"
```

## Property-Based Integration Tests (Hypothesis)

```python
from hypothesis import given, settings, strategies as st


@pytest.mark.integration
@settings(max_examples=50, deadline=None)          # network-free but slower than unit tests
@given(
    vendor=st.text(min_size=1, max_size=100),
    amount=st.floats(min_value=0.01, max_value=1_000_000, allow_nan=False),
)
def test_invoice_roundtrip(api_client, vendor, amount):   # api_client is module-scoped (see above)
    payload = {"invoice_number": "INV-HYP", "vendor": vendor, "amount": round(amount, 2)}
    create = api_client.post("/invoices", json=payload)
    if create.status_code == 201:
        fetched = api_client.get(f"/invoices/{create.json()['id']}")
        assert fetched.json()["vendor"] == vendor
```

A function-scoped fixture here fails Hypothesis' `function_scoped_fixture` health check; see
[Property-Based Testing](../concepts/property-based-testing.md).

## See Also

- [Fixtures](../concepts/fixtures.md)
- [Async Testing](../concepts/async-testing.md)
- [Spark Testing](../patterns/spark-testing.md)

## Sources

- https://docs.pytest.org/en/stable/how-to/fixtures.html
- https://docs.pytest.org/en/stable/how-to/tmp_path.html
- https://docs.pytest.org/en/stable/reference/reference.html
- https://pytest-asyncio.readthedocs.io/en/latest/reference/changelog.html
- https://hypothesis.readthedocs.io/en/latest/reference/api.html
- https://fastapi.tiangolo.com/advanced/testing-events/
