# Type Hints

> **Purpose**: Type annotations that run on Python 3.12+ -- unions, Self, PEP 695 generics, TypedDict, Protocol, narrowing
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Type hints let a type checker (mypy, pyright, ty) catch bugs before runtime. Everything here runs on
**3.12** (Databricks Runtime 17.3 LTS); 3.13+ typing features (`TypeIs`, `ReadOnly`, deferred
annotations, `TypeForm`, TypedDict `extra_items`) and their backports are in [Python Versions](python-versions.md).

## The Concept

```python
from dataclasses import dataclass, field
from typing import Self


@dataclass(slots=True)
class TreeNode:
    value: int
    children: list[Self] = field(default_factory=list)

    def add_child(self, value: int) -> Self:
        child = type(self)(value=value)   # not TreeNode(...): keeps subclasses typed correctly
        self.children.append(child)
        return child
```

## Modern Syntax Reference

| Old | Modern | Since |
|-----|--------|-------|
| `Optional[str]` | `str \| None` | 3.10 |
| `Union[int, str]` | `int \| str` | 3.10 |
| `List[int]`, `Dict[str, int]`, `Tuple[int, ...]` | `list[int]`, `dict[str, int]`, `tuple[int, ...]` | 3.9 |
| `T = TypeVar("T")` + `Generic[T]` | `def f[T](x: T) -> T` / `class Box[T]:` | 3.12 |
| `Vector: TypeAlias = list[float]` | `type Vector = list[float]` | 3.12 |
| `from typing import Callable, Iterator` | `from collections.abc import Callable, Iterator` | 3.9 |

## Python 3.11 Features

```python
from typing import LiteralString, Never, Self


class Builder:
    def set_name(self, name: str) -> Self:   # subclasses keep their own type
        self.name = name
        return self


def run_query(sql: LiteralString) -> list[dict]: ...   # rejects f-strings built from user input
def fail(msg: str) -> Never:                             # function never returns
    raise RuntimeError(msg)
```

## Python 3.12 Generics (PEP 695)

```python
from collections.abc import Callable


def first[T](items: list[T]) -> T:
    return items[0]


class Box[T]:
    def __init__(self, item: T) -> None:
        self.item = item


type Vector = list[float]
type Callback[T] = Callable[[T], None]   # alias body is evaluated lazily
```

## TypedDict and Protocol

```python
from typing import NotRequired, Protocol, TypedDict, runtime_checkable


class APIResponse(TypedDict):
    status: int
    data: dict[str, object]
    error: NotRequired[str]          # key may be absent


@runtime_checkable
class Serializable(Protocol):        # structural typing: any class with to_dict() matches
    def to_dict(self) -> dict[str, object]: ...
```

## Type Narrowing with TypeIs (3.13+, typing_extensions on 3.12)

```python
from typing_extensions import TypeIs   # `from typing import TypeIs` on 3.13+


def is_str(val: int | str) -> TypeIs[str]:   # narrowed type must be a subtype of the input
    return isinstance(val, str)


def describe(val: int | str) -> str:
    if is_str(val):
        return val.upper()           # narrowed to str
    return str(val + 1)              # narrowed to int in the else branch (TypeGuard cannot)
```

## Common Mistakes

### Wrong

```python
from typing import Dict, List, Optional, Union

def process(items: Optional[List[Dict[str, Union[int, str]]]]) -> None: ...
```

### Correct

```python
def process(items: list[dict[str, int | str]] | None) -> None: ...
```

## Quick Reference

| Type | Use Case | Since |
|------|----------|-------|
| `str \| None` | Optional value | 3.10 |
| `Self` / `Never` / `LiteralString` | Fluent APIs / no-return / SQL safety | 3.11 |
| `def fn[T](x: T)`, `type X = ...` | Generics and aliases | 3.12 |
| `Annotated[int, Gt(0)]` / `Protocol` | Metadata for validators / structural subtyping | 3.9 / 3.8 |
| `TypeIs` / `ReadOnly` | Narrowing / read-only TypedDict keys | 3.13 (typing_extensions on 3.12) |
| `TypeForm` | Annotate type expressions passed as values | 3.15 (typing_extensions 4.13+) |

## Related

- [Python Versions](../concepts/python-versions.md)
- [Dataclasses](../concepts/dataclasses.md)
- [Error Handling](../patterns/error-handling.md)

## Sources

- https://docs.python.org/3/library/typing.html
- https://peps.python.org/pep-0695/
- https://typing-extensions.readthedocs.io/en/latest/
- https://docs.python.org/3.13/whatsnew/3.13.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
