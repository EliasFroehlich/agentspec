# Async Testing (pytest-asyncio 1.x)

> **Purpose**: Test coroutines with pytest-asyncio 1.x -- strict mode, async fixtures, event-loop scopes, AsyncMock
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

pytest does not run coroutines by itself; the pytest-asyncio plugin does. Version **1.0.0**
(2025-05-26) removed the `event_loop` fixture, and the default mode is **strict**: async tests need
`@pytest.mark.asyncio` and async fixtures need `@pytest_asyncio.fixture`. Under pytest 9 a test that
requests an async fixture no plugin handled **errors** ("pytest does not natively support it").
Current release: pytest-asyncio 1.4.0 (2026-05-26).

## The Concept

```python
# pyproject.toml (pytest 9 native table)
# [tool.pytest]
# asyncio_default_fixture_loop_scope = "function"
# asyncio_default_test_loop_scope = "function"
import asyncio

import pytest
import pytest_asyncio


@pytest_asyncio.fixture                               # strict mode: not @pytest.fixture
async def resource():
    await asyncio.sleep(0)
    yield {"ok": True}                                # code after yield = async teardown


@pytest_asyncio.fixture(loop_scope="session", scope="session")
async def shared_pool():
    pool = object()                                   # e.g. await create_pool(...)
    yield pool


@pytest.mark.asyncio
async def test_resource(resource):
    assert resource["ok"] is True


@pytest.mark.asyncio(loop_scope="session")            # same loop as the session fixture
async def test_uses_pool(shared_pool):
    assert shared_pool is not None
```

## Quick Reference

| Need | 1.x way | Removed / deprecated |
|------|---------|----------------------|
| Run an async test | `@pytest.mark.asyncio` (strict) or `asyncio_mode = "auto"` | -- |
| Async fixture | `@pytest_asyncio.fixture` (any mode); plain `@pytest.fixture` only in auto mode | `@pytest.fixture` async in strict mode (warned since 0.25) |
| Share a loop across tests | `loop_scope="module"`/`"session"` on marker and fixture | overriding `event_loop` (removed in 1.0.0) |
| Default fixture loop scope | `asyncio_default_fixture_loop_scope` (unset = fixture scope) | -- |
| Default test loop scope | `asyncio_default_test_loop_scope` (unset = `function`) | -- |
| Custom loop / policy | `pytest_asyncio_loop_factories` hook (1.4.0) | overriding `event_loop_policy` (deprecated in 1.4.0) |
| Mock a coroutine | `unittest.mock.AsyncMock`, `assert_awaited_once()` | -- |

## Auto Mode

```python
# [tool.pytest]
# asyncio_mode = "auto"        # every async test and async fixture is handled by pytest-asyncio
from unittest.mock import AsyncMock

import pytest


@pytest.fixture
async def balance():           # allowed in auto mode
    yield 41


async def test_total(balance):  # no marker needed in auto mode
    fetch = AsyncMock(return_value=1)
    assert balance + await fetch() == 42
    fetch.assert_awaited_once()
```

Auto mode is convenient in pure-asyncio projects; keep strict mode when other async plugins
(anyio, trio) share the suite.

## Common Mistakes

### Wrong

```python
@pytest.fixture                       # strict mode + pytest 9: test errors at setup
async def client():
    async with httpx.AsyncClient() as c:
        yield c


@pytest.fixture(scope="session")
def event_loop():                     # fixture removed in pytest-asyncio 1.0.0
    ...
```

### Correct

```python
@pytest_asyncio.fixture
async def client():
    transport = httpx.ASGITransport(app=app)          # in-process, no network
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

## Related

- [Fixtures](../concepts/fixtures.md)
- [Mocking](../concepts/mocking.md)
- [Integration Tests](../patterns/integration-tests.md)

## Sources

- https://pytest-asyncio.readthedocs.io/en/latest/reference/changelog.html
- https://pytest-asyncio.readthedocs.io/en/latest/reference/configuration.html
- https://docs.pytest.org/en/stable/deprecations.html#sync-test-depending-on-async-fixture
- https://docs.python.org/3/library/unittest.mock.html
