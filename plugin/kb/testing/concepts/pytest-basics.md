# pytest Basics

> **Purpose**: Core pytest 9 conventions -- discovery, markers, expected exceptions, subtests, strict mode and CLI
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

pytest discovers `test_*.py` / `*_test.py` files, `Test*` classes and `test_*` functions, and
rewrites plain `assert` statements to show rich failure diffs. Current release is **pytest 9.1.1**
(2026-06-19); 9.0 dropped Python 3.9 and made `PytestRemovedIn9Warning` an error. Check the runtime:
Databricks Runtime 17.3 LTS preinstalls **pytest 8.3.5**, which has none of the 8.4/9.x features
below unless you `%pip install "pytest>=9"`.

## The Pattern

```python
import pytest


def test_parse_invoice_number():
    raw = "  inv-001  "
    assert raw.strip().upper() == "INV-001"


class TestStringProcessor:          # grouping only, no base class
    def test_strip_whitespace(self):
        assert "  hello  ".strip() == "hello"


def test_raises_value_error():
    with pytest.raises(ValueError, match="invalid literal"):
        int("not a number")


def test_each_row(subtests):        # pytest 9.0+: built-in subtests fixture
    for i, raw in enumerate(["1", "2", "3"]):
        with subtests.test(msg="parse", i=i):   # every failing row is reported, not just the first
            assert int(raw) == i + 1


def test_batch_errors():            # pytest 8.4+: match ExceptionGroup contents
    with pytest.RaisesGroup(ValueError, TypeError):
        raise ExceptionGroup("batch", [ValueError("a"), TypeError("b")])
```

## Markers

```python
import sys

import pytest


@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature(): ...


@pytest.mark.skipif(sys.platform == "win32", reason="POSIX only")
def test_posix_only(): ...


@pytest.mark.xfail(reason="Known bug #123", strict=True)   # XPASS fails the run
def test_known_failure():
    assert 1 == 2


@pytest.mark.slow                    # custom: register it, or strict mode rejects it
def test_large_dataset_processing(): ...
```

## Configuration (pytest 9 native TOML)

```toml
[tool.pytest]                     # pytest 9+; cannot coexist with [tool.pytest.ini_options]
minversion = "9.0"
testpaths = ["tests"]
addopts = ["-ra"]
strict = true                     # strict_config + strict_markers + strict_parametrization_ids + strict_xfail
markers = ["slow: long-running tests", "integration: needs external services"]
```

`strict = true` also turns on strictness options added in future versions, so use it with a
pinned pytest. On pytest 8.x keep `[tool.pytest.ini_options]` with `addopts = "--strict-markers"`.

## New Since pytest 8.4

| Feature | Version | Notes |
|---------|---------|-------|
| `pytest.RaisesGroup` / `pytest.RaisesExc` | 8.4 | Assert on `ExceptionGroup` members |
| `pytest.HIDDEN_PARAM` | 8.4 | Hide a parameter from the test ID |
| Built-in `subtests` fixture | 9.0 | Formerly the pytest-subtests plugin; marked experimental (reporting may change) |
| Native TOML config (`[tool.pytest]`, `pytest.toml`) | 9.0 | Real TOML types instead of INI strings |
| `strict` option / `--strict` | 9.0 | Enables all strictness options |
| `PytestRemovedIn9Warning` is an error | 9.0 | e.g. a sync test that requests an async fixture now errors |

## Common Mistakes

### Wrong

```python
import unittest

class TestBad(unittest.TestCase):          # works, but loses fixtures and assert rewriting
    def test_value(self):
        self.assertEqual(1 + 1, 2)

def test_rows():
    for raw in ["1", "x", "3"]:
        assert raw.isdigit()               # stops at the first failure; "3" never checked
```

### Correct

```python
def test_value():
    assert 1 + 1 == 2

@pytest.mark.parametrize("raw", ["1", "x", "3"])   # or the subtests fixture
def test_rows(raw):
    assert raw.isdigit()
```

## CLI Quick Reference

| Flag | Purpose |
|------|---------|
| `-x` / `--sw` | Stop at first failure / stepwise (resume next run) |
| `--lf` / `--ff` | Only last-failed / failures first |
| `-k "expr"` / `-m "marker"` | Filter by name / marker expression |
| `-ra` | Summary of skips, xfails and errors |
| `--co -q` | List test IDs without running |
| `-o key=value` | Override a config option for this run |
| `-p no:cacheprovider` | Disable `.pytest_cache` (read-only filesystems, e.g. Databricks workspace) |
| `-n auto` | Parallel run (pytest-xdist plugin) |

## Related

- [Fixtures](../concepts/fixtures.md)
- [Async Testing](../concepts/async-testing.md)
- [Property-Based Testing](../concepts/property-based-testing.md)

## Sources

- https://docs.pytest.org/en/stable/changelog.html
- https://docs.pytest.org/en/stable/reference/reference.html
- https://docs.pytest.org/en/stable/how-to/plugins.html
- https://docs.pytest.org/en/stable/deprecations.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://docs.databricks.com/aws/en/notebooks/testing
