# Property-Based Testing (Hypothesis)

> **Purpose**: Generate test inputs with Hypothesis -- strategies, explicit examples, reproducibility, settings profiles and health checks
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Instead of hand-picking inputs, `@given` describes the *kind* of input and Hypothesis searches for a
failing case, then shrinks it to a minimal example. Runs are reproducible: failures are replayed from
the example database, `@example` pins known cases, and `derandomize`/`@seed` fix generation. Current
release: Hypothesis 6.170.1 (2026-10-16); 6.169.0 dropped Python 3.10.

## The Concept

```python
from hypothesis import example, given, settings, strategies as st


def normalize_code(raw: str) -> str:
    return raw.strip().upper()


@given(st.text())
@example("  inv-001 ")            # always run this case, in addition to generated ones
def test_normalize_is_idempotent(raw):
    once = normalize_code(raw)
    assert normalize_code(once) == once


@settings(max_examples=200)
@given(st.lists(st.integers()))
def test_sort_keeps_length(xs):
    assert len(sorted(xs)) == len(xs)
```

## Quick Reference

| Need | API |
|------|-----|
| Generate values | `st.integers(min_value=0)`, `st.text(min_size=1)`, `st.lists(...)`, `st.dictionaries(...)` |
| Build objects | `st.builds(Invoice, amount=st.floats(min_value=0, allow_nan=False))` |
| Custom strategy | `@st.composite` |
| Pin a known case | `@example(...)` |
| Reject invalid input | `assume(cond)` (prefer narrower strategies) |
| Reproduce | failure database (automatic), `@seed(n)`, `settings(derandomize=True)` |
| Tune run | `@settings(max_examples=..., deadline=...)` |
| Profiles | `settings.register_profile("ci", ...)` + `settings.load_profile("ci")`; built-in `ci` profile is active when `CI` is set |

## Settings Profiles

```python
# tests/conftest.py
import os

from hypothesis import settings

settings.register_profile("dev", max_examples=50)
settings.register_profile("nightly", max_examples=2_000)
settings.load_profile(os.getenv("HYPOTHESIS_PROFILE", "dev"))
```

## Common Mistakes

### Wrong

```python
@pytest.fixture
def api_client():                       # function-scoped
    return make_client()


@given(vendor=st.text(min_size=1))
def test_roundtrip(api_client, vendor):  # FailedHealthCheck: function_scoped_fixture
    ...
```

Function-scoped fixtures are **not** reset between generated inputs, so Hypothesis fails the test
with `HealthCheck.function_scoped_fixture`.

### Correct

```python
@pytest.fixture(scope="module")         # shared, read-only resource is fine
def api_client():
    return make_client()


@given(vendor=st.text(min_size=1))
def test_roundtrip(api_client, vendor):
    ...
```

If per-input state is needed, create it inside the test (or a context manager there). Suppress
the check (`@settings(suppress_health_check=[HealthCheck.function_scoped_fixture])`) only when the
fixture is truly stateless.

## Related

- [pytest Basics](../concepts/pytest-basics.md)
- [Parametrize](../concepts/parametrize.md)
- [Integration Tests](../patterns/integration-tests.md)

## Sources

- https://hypothesis.readthedocs.io/en/latest/reference/api.html
- https://hypothesis.readthedocs.io/en/latest/changelog.html
- https://hypothesis.readthedocs.io/en/latest/reference/strategies.html
