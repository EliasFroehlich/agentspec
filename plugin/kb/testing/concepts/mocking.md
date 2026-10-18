# Mocking

> **Purpose**: unittest.mock, monkeypatch, and patching strategies for test isolation
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Mocking replaces real dependencies with controlled doubles so tests run in isolation. Python
provides `unittest.mock` (Mock, MagicMock, patch) in the standard library, and pytest adds
`monkeypatch` for simpler attribute/env patching; the pytest-mock plugin adds a `mocker` fixture.
The critical rule: always patch where the name is **looked up**, not where it is **defined**.

## The Pattern

```python
from unittest.mock import Mock, MagicMock, patch, AsyncMock
import pytest


# --- unittest.mock.patch ---

@patch("myapp.service.external_api_call")
def test_service_with_mock(mock_api):
    """Patch where imported, not where defined."""
    mock_api.return_value = {"status": "ok", "data": [1, 2, 3]}

    from myapp.service import process_data
    result = process_data()

    assert result == [1, 2, 3]
    mock_api.assert_called_once()


# --- pytest monkeypatch ---

def test_with_monkeypatch(monkeypatch):
    """Simpler patching for attributes and env vars."""
    monkeypatch.setenv("API_KEY", "test-key-123")
    monkeypatch.setattr("myapp.config.TIMEOUT", 5)

    from myapp.config import get_settings
    settings = get_settings()
    assert settings.api_key == "test-key-123"
```

## Patch Target Rule

```text
CRITICAL: Patch where the name is LOOKED UP, not where it is DEFINED.

# myapp/utils.py
def fetch_data():
    return requests.get(...)

# myapp/service.py
from myapp.utils import fetch_data  # <-- imported here

# test_service.py
@patch("myapp.service.fetch_data")    # CORRECT: patch in service
@patch("myapp.utils.fetch_data")      # WRONG: patch at definition
```

## Mock Object Quick Reference

| Feature | Syntax | Purpose |
|---------|--------|---------|
| Basic mock | `Mock()` | Accepts any attribute/call |
| Magic methods | `MagicMock()` | Supports `__len__`, `__iter__`, etc. |
| Return value | `m.return_value = X` | `m()` returns X |
| Side effect list | `m.side_effect = [1, 2]` | Sequential returns |
| Side effect error | `m.side_effect = ValueError("msg")` | Raise on call |
| Side effect fn | `m.side_effect = lambda x: x * 2` | Custom logic |
| Async mock | `AsyncMock()` | For async functions |
| Spec mock | `Mock(spec=MyClass)` | Only allows real attrs |

## Assertion Methods

```python
mock_fn = Mock()
mock_fn("arg1", key="val")

mock_fn.assert_called_once_with("arg1", key="val")   # also: assert_called(), assert_called_with()
assert mock_fn.call_count == 1
assert mock_fn.call_args == call("arg1", key="val")     # from unittest.mock import call
```

## monkeypatch Patterns

```python
def test_env_vars(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "sqlite:///test.db")
    monkeypatch.delenv("SECRET_KEY", raising=False)

def test_setattr(monkeypatch):
    monkeypatch.setattr("os.path.exists", lambda p: True)

def test_dict_item(monkeypatch):
    monkeypatch.setitem(config_dict, "retries", 0)
```

## pytest-mock (`mocker` fixture)

```python
def test_service_with_mocker(mocker):
    """Same API as mock.patch; every patch is undone automatically after the test."""
    fake_api = mocker.patch("myapp.service.external_api_call", return_value={"data": [1]})
    spy = mocker.spy(myapp.service, "transform")       # real call, but recorded
    assert myapp.service.process_data() == [1]
    fake_api.assert_called_once()
    assert spy.call_count == 1
```

Install with `uv add --group test pytest-mock`; do not use `mocker.patch` as a context manager (it warns).

## Common Mistakes

### Wrong

```python
# Patching at the definition site
@patch("external_lib.client.make_request")  # WRONG
def test_bad(mock_req):
    from myapp.api import call_api
    call_api()
```

### Correct

```python
# Patching at the import site
@patch("myapp.api.make_request")  # CORRECT
def test_good(mock_req):
    mock_req.return_value = {"ok": True}
    from myapp.api import call_api
    result = call_api()
    assert result["ok"] is True
```

## Related

- [Fixtures](../concepts/fixtures.md)
- [Unit Test Patterns](../patterns/unit-test-patterns.md)
- [Integration Tests](../patterns/integration-tests.md)

## Sources

- https://docs.python.org/3/library/unittest.mock.html
- https://docs.pytest.org/en/stable/how-to/monkeypatch.html
- https://pytest-mock.readthedocs.io/en/latest/usage.html
