# Python Versions and Runtime Targets

> **Purpose**: Support lifecycle, what changed in 3.13/3.14/3.15, and how to code for the interpreter you deploy on
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Pick the language level from the **runtime that executes the code**, not the newest release. Python
3.15.0 shipped on 2026-10-09, but Databricks Runtime 17.3 LTS runs **Python 3.12.3** (with pydantic
2.10.6, pytest 8.3.5, typing_extensions 4.12.2), so any 3.13+ syntax or stdlib name fails there.

## Support Lifecycle (devguide, 2026-10-17)

| Version | Status | First release | End of life |
|---------|--------|---------------|-------------|
| 3.10 | end-of-life | 2021-10-04 | 2026-10-01 |
| 3.11 | security fixes only | 2022-10-24 | 2027-10 |
| 3.12 | security fixes only | 2023-10-02 | 2028-10 |
| 3.13 | security fixes only | 2024-10-07 | 2029-10 |
| 3.14 | bugfix | 2025-10-07 | 2030-10 |
| 3.15 | bugfix (latest; `main` is 3.16) | 2026-10-09 | 2031-10 |

## The Concept

```python
import sys

if sys.version_info >= (3, 13):
    from typing import ReadOnly, TypedDict, TypeIs
    from warnings import deprecated
else:  # Python 3.12, e.g. Databricks Runtime 17.3 LTS
    from typing_extensions import ReadOnly, TypedDict, TypeIs, deprecated


class RunConfig(TypedDict):
    job_id: ReadOnly[str]   # type checkers reject writes to this key
    retries: int


def is_str(value: int | str) -> TypeIs[str]:
    return isinstance(value, str)


@deprecated("Use load_config_v2()")  # emits DeprecationWarning at call time
def load_config() -> RunConfig:
    return {"job_id": "daily", "retries": 3}
```

Backports: `ReadOnly` typing_extensions 4.9.0+, `TypeIs` 4.10.0+, `TypeForm` 4.13.0+ (not in DBR 17.3's 4.12.2).

## What Changed, and Can I Use It on 3.12?

| Feature | Version | On a 3.12 runtime |
|---------|---------|-------------------|
| `TypeIs` (PEP 742), `ReadOnly` (PEP 705), TypeVar defaults (PEP 696) | 3.13 | via `typing_extensions` |
| `@warnings.deprecated` (PEP 702) | 3.13 | `typing_extensions.deprecated` |
| `itertools.batched(..., strict=True)` | 3.13 | no (`batched` itself is 3.12) |
| Free-threaded build (PEP 703) | 3.13 experimental, 3.14 supported (separate `python3.14t` build) | no |
| JIT compiler | 3.13 experimental/off by default; 3.14 experimental; 3.15 "significantly upgraded", still experimental | no |
| Deferred annotations (PEP 649/749), `annotationlib` | 3.14 | no (`from __future__ import annotations` is the 3.12 tool) |
| Template strings `t"..."` (PEP 750) | 3.14 | no (SyntaxError) |
| `except A, B:` without parentheses (PEP 758, only without `as`) | 3.14 | no (SyntaxError) |
| `Union[X, Y]` and `X \| Y` are the same runtime class | 3.14 | no |
| `lazy import` (PEP 810), `frozendict` (PEP 814), `sentinel` (PEP 661) | 3.15 | no |
| `typing.TypeForm` (PEP 747), TypedDict extra items (PEP 728) | 3.15 | `TypeForm` via typing_extensions 4.13+ |
| UTF-8 default encoding (PEP 686) | 3.15 | pass `encoding="utf-8"` explicitly |

## Python 3.14 Syntax

```python
from string.templatelib import Interpolation, Template

try:
    connect()
except TimeoutError, ConnectionRefusedError:   # PEP 758; needs parentheses when using `as`
    schedule_retry()


def to_sql(query: Template) -> tuple[str, list[object]]:  # parameterize, never concatenate
    sql: list[str] = []
    params: list[object] = []
    for part in query:
        if isinstance(part, Interpolation):
            sql.append("?")
            params.append(part.value)
        else:
            sql.append(part)
    return "".join(sql), params


name = "O'Brien"
to_sql(t"SELECT id FROM users WHERE name = {name}")  # ('... WHERE name = ?', ["O'Brien"])
```

## Python 3.15 Syntax

```python
lazy import json                     # PEP 810: module-level only, loaded on first use
from typing import TypeForm          # PEP 747


def parse_as[T](raw: str, tp: TypeForm[T]) -> T:
    value = json.loads(raw)
    if not isinstance(value, tp):    # runtime isinstance only works for plain classes
        raise TypeError(f"expected {tp}")
    return value


defaults = frozendict(region="us-east-1", retries=3)   # PEP 814: immutable and hashable
# `lazy import` is a SyntaxError inside functions, class bodies and `try` blocks
```

## Common Mistakes

### Wrong

```python
from typing import TypeIs          # built on a 3.14 laptop: ImportError on a 3.12 cluster
except ValueError, KeyError:       # SyntaxError on 3.12 and 3.13
```

### Correct

```python
from typing_extensions import TypeIs   # works on 3.12 and later
except (ValueError, KeyError):          # works on every supported version
```

Pin `requires-python = ">=3.12"` and ruff `target-version = "py312"` so tooling rejects newer syntax.

## Related

- [Type Hints](../concepts/type-hints.md)
- [Error Handling](../patterns/error-handling.md)
- [Project Tooling](../patterns/project-tooling.md)

## Sources

- https://devguide.python.org/versions/
- https://docs.python.org/3.13/whatsnew/3.13.html
- https://docs.python.org/3.14/whatsnew/3.14.html
- https://docs.python.org/3.15/whatsnew/3.15.html
- https://docs.python.org/3/library/typing.html
- https://docs.python.org/3/library/warnings.html
- https://docs.python.org/3/library/itertools.html
- https://peps.python.org/pep-0810/
- https://peps.python.org/pep-0747/
- https://typing-extensions.readthedocs.io/en/latest/
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
