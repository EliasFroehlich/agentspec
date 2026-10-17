# Python Clean Code Quick Reference

> **Purpose**: Fast lookup tables; code examples live in the linked files
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Versions (2026-10-17)

| Version | Status | Note |
|---------|--------|------|
| 3.10 | EOL 2026-10-01 | upgrade |
| 3.12 | security fixes until 2028-10 | Databricks Runtime 17.3 LTS = 3.12.3 |
| 3.14 | bugfix | free-threaded build supported |
| 3.15 | bugfix, released 2026-10-09 | latest |

## Type Hint Patterns

| Pattern | Syntax | Version |
|---------|--------|---------|
| Union / optional | `int \| str`, `str \| None` | 3.10+ |
| Self return | `-> Self` | 3.11+ |
| Generic function / class | `def fn[T](x: T) -> T`, `class Box[T]:` | 3.12+ |
| Type alias | `type Vector = list[float]` | 3.12+ |
| TypeIs (narrow both branches) | `def is_str(v: int \| str) -> TypeIs[str]` | 3.13+ (typing_extensions 4.10+) |
| Read-only TypedDict key | `ReadOnly[str]` | 3.13+ (typing_extensions 4.9+) |
| Deprecation marker | `@warnings.deprecated("...")` | 3.13+ (typing_extensions) |
| Deferred annotations | forward refs without quotes | 3.14+ |
| TypeForm | `def cast[T](tp: TypeForm[T], v: object) -> T` | 3.15+ (typing_extensions 4.13+) |

## New Syntax by Version (not available on 3.12)

| Feature | Version | Example |
|---------|---------|---------|
| Template strings | 3.14 | `t"WHERE id = {id}"` -> `Template` |
| Unparenthesized except | 3.14 | `except TimeoutError, OSError:` (no `as`) |
| Lazy imports | 3.15 | `lazy import json` (module level only) |
| frozendict | 3.15 | `frozendict(a=1)` |

## Dataclass Options

| Parameter | Default | Effect |
|-----------|---------|--------|
| `slots=True` | `False` | `__slots__`, no instance `__dict__` |
| `frozen=True` | `False` | Immutable; hashable with `eq=True` |
| `kw_only=True` | `False` | Keyword-only constructor |
| `order=True` | `False` | Comparison methods |

## uv / ruff Commands

| Command | Purpose |
|---------|---------|
| `uv init --package app --python 3.12` | New src-layout project pinned to 3.12 |
| `uv add pkg` / `uv add --group test pytest` | Runtime / dependency-group (PEP 735) dependency |
| `uv lock` / `uv sync --locked` | Resolve / install exactly the lockfile (CI) |
| `uv run pytest` / `uvx ty check` | Run in project venv / run a tool ad hoc |
| `uv python pin 3.12` | Write `.python-version` |
| `ruff check --select UP,F401 --fix .` | Auto-modernize to `target-version` and drop unused imports |
| `ruff format .` | Format (Black-compatible) |

## Decision Matrix

| Use Case | Choose |
|----------|--------|
| Plain / immutable container | `@dataclass(slots=True)` / `@dataclass(frozen=True, slots=True)` |
| Validation at a boundary | Pydantic `BaseModel` |
| High-volume JSON/MessagePack decoding | `msgspec.Struct` |
| Large file, line by line | Generator with `yield` |
| Several failures to report together | `ExceptionGroup` + `except*` (3.11+) |
| Type checker | mypy 2.x or pyright; ty only experimentally (beta) |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `except:` / deep `except Exception:` | `except (ValueError, TypeError) as e:`; broad catch only at the boundary |
| Mutable default `field=[]` | `field(default_factory=list)` |
| `datetime.utcnow()` | `datetime.now(UTC)` (utcnow deprecated in 3.12, naive) |
| `tempfile.mktemp()` | `tempfile.mkstemp()` / `NamedTemporaryFile` / pytest `tmp_path` |
| `from typing import TypeIs` on a 3.12 cluster | `from typing_extensions import TypeIs` |
| `ignore = ["ANN101", "ANN102"]` in ruff | Delete (rules removed in ruff 0.8.0) |
| uv without a pin | `--python 3.12` / `.python-version` matching the runtime |

## Related Documentation

| Topic | Path |
|-------|------|
| Versions and backports | `concepts/python-versions.md` |
| Type hints guide | `concepts/type-hints.md` |
| Tooling | `patterns/project-tooling.md` |
| Full Index | `index.md` |

## Sources

- https://devguide.python.org/versions/
- https://docs.python.org/3/library/typing.html
- https://docs.python.org/3/library/datetime.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://docs.astral.sh/uv/
- https://docs.astral.sh/ruff/rules/missing-type-self/
