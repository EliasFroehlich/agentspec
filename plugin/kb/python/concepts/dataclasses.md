# Dataclasses

> **Purpose**: @dataclass patterns with slots, frozen, kw_only and field factories (Python 3.12+), and when to pick attrs, Pydantic or msgspec instead
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Dataclasses generate `__init__`, `__repr__` and `__eq__` from annotated fields. `slots=True`,
`kw_only=True` and `match_args` exist since 3.10. `frozen=True` plus `slots=True` gives immutable,
hashable value objects with no per-instance `__dict__`, which saves memory and speeds up attribute
access. Dataclasses do **not** validate types at runtime. Use `datetime.now(UTC)` for timestamps:
`datetime.utcnow()` is deprecated since 3.12 and returns a *naive* datetime.

## The Concept

```python
from dataclasses import dataclass, field, replace
from datetime import UTC, datetime   # datetime.UTC exists since 3.11
from typing import Self


@dataclass(frozen=True, slots=True)
class Metric:
    name: str
    value: float
    timestamp: datetime = field(default_factory=lambda: datetime.now(UTC))   # timezone-aware
    tags: tuple[str, ...] = ()

    def with_tag(self, tag: str) -> Self:
        return replace(self, tags=(*self.tags, tag))   # copy with changes, keeps the class
```

## Decorator Parameters

| Parameter | Default | Effect |
|-----------|---------|--------|
| `slots=True` | `False` | Generates `__slots__`: no instance `__dict__`, less memory, faster attribute access |
| `frozen=True` | `False` | Assignment raises `FrozenInstanceError`; with `eq=True` the instance is hashable |
| `kw_only=True` | `False` | All fields become keyword-only constructor arguments |
| `order=True` | `False` | Generates `__lt__`, `__le__`, `__gt__`, `__ge__` |
| `match_args=True` | `True` | Generates `__match_args__` for positional pattern matching |

## Field Options

```python
from dataclasses import dataclass, field


@dataclass(slots=True)
class Pipeline:
    name: str
    steps: list[str] = field(default_factory=list)
    _run_id: str = field(default="", init=False, compare=False)
```

| Field parameter | Purpose |
|-----------------|---------|
| `default_factory` | Callable for mutable defaults (list, dict, set) |
| `repr=False` / `compare=False` / `hash=False` | Exclude from repr / equality / hash |
| `init=False` | Exclude from `__init__`; set it in `__post_init__` |

## Common Mistakes

### Wrong

```python
@dataclass
class Config:
    items: list[str] = []                                  # ValueError: mutable default
    created: datetime = field(default_factory=datetime.utcnow)   # deprecated, naive time
```

### Correct

```python
@dataclass(slots=True)
class Config:
    items: list[str] = field(default_factory=list)
    created: datetime = field(default_factory=lambda: datetime.now(UTC))
```

## Pattern Matching and Post-Init

```python
from dataclasses import dataclass, field


@dataclass(slots=True)
class Point:
    x: float
    y: float


@dataclass(slots=True)
class FilePath:
    raw: str
    resolved: str = field(init=False)

    def __post_init__(self) -> None:
        self.resolved = self.raw.strip().replace("\\", "/")


def describe(shape: object) -> str:
    match shape:
        case Point(0, 0):
            return "origin"
        case Point(x, y) if x == y:
            return f"diagonal point {x}"
        case _:
            return "unknown"
```

## Dataclass vs attrs vs Pydantic vs msgspec

| Feature | dataclass | attrs (`@define`) | Pydantic `BaseModel` | msgspec `Struct` |
|---------|-----------|-------------------|----------------------|------------------|
| Install | stdlib | package | package | package |
| Slots | `slots=True` | default | no (`__dict__`) | -- |
| Runtime validation | no | opt-in validators | always, on construction | on decode only |
| Coercion (`"1"` -> `1`) | no | opt-in converters | yes (lax mode) | no |
| Serialization | `asdict()` only | via `cattrs` | JSON + JSON Schema | JSON, MessagePack, YAML, TOML |
| Best for | internal DTOs | rich internal models | API/LLM boundaries, config | high-volume message decoding |

```python
import msgspec


class Event(msgspec.Struct):
    name: str
    count: int = 0


msgspec.json.decode(b'{"name": "click", "count": 3}', type=Event)   # validates while decoding
Event(name=123)   # accepted: the constructor does not validate
```

## Related

- [Type Hints](../concepts/type-hints.md)
- [Clean Architecture](../patterns/clean-architecture.md)
- [Pydantic BaseModel](../../pydantic/concepts/base-model.md)

## Sources

- https://docs.python.org/3/library/dataclasses.html
- https://docs.python.org/3/reference/datamodel.html#slots
- https://docs.python.org/3/library/datetime.html
- https://www.attrs.org/en/stable/api.html
- https://github.com/msgspec/msgspec
