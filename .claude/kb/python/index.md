# Python Clean Code Knowledge Base

> **Purpose**: Clean, typed Python 3.12-3.15 for data engineering -- dataclasses, type hints, generators, error handling, uv/ruff tooling
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Runtime Target First

Code for the interpreter that runs it. Databricks Runtime 17.3 LTS ships **Python 3.12.3**; Python
3.15.0 was released on 2026-10-09. Anything this KB marks "3.13+", "3.14+" or "3.15+" fails on a 3.12
runtime unless a backport is named. Start with [concepts/python-versions.md](concepts/python-versions.md).

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/python-versions.md](concepts/python-versions.md) | Support lifecycle, 3.13/3.14/3.15 changes, runtime targets and backports |
| [concepts/dataclasses.md](concepts/dataclasses.md) | @dataclass with slots, frozen, kw_only; vs attrs, Pydantic, msgspec |
| [concepts/type-hints.md](concepts/type-hints.md) | Unions, Self, PEP 695 generics, TypedDict, Protocol, TypeIs |
| [concepts/generators.md](concepts/generators.md) | Generator functions, yield, send, generator expressions |
| [concepts/context-managers.md](concepts/context-managers.md) | with statement, __enter__/__exit__, contextlib |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/file-parser.md](patterns/file-parser.md) | File parsing with generators and context managers |
| [patterns/clean-architecture.md](patterns/clean-architecture.md) | Structure, naming, module organization |
| [patterns/project-tooling.md](patterns/project-tooling.md) | uv projects, PEP 735 dependency groups, ruff, mypy/pyright/ty |
| [patterns/error-handling.md](patterns/error-handling.md) | Exception hierarchy, chaining, retry, exception groups |
| [patterns/functional-patterns.md](patterns/functional-patterns.md) | Comprehensions, map, filter, reduce, functools, itertools |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/python-standards.yaml](specs/python-standards.yaml) | Versions, runtime targets, linting rules, conventions |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Runtime target** | `requires-python` and ruff `target-version` follow the deployment interpreter (DBR 17.3 LTS = 3.12) |
| **Dataclasses** | Declarative containers; `frozen=True, slots=True` for value objects; no runtime validation |
| **Type Hints** | `X \| None`, `Self`, PEP 695 generics (3.12); `TypeIs`/`ReadOnly` (3.13); `TypeForm` (3.15) |
| **Generators** | Lazy evaluation for memory-efficient data processing |
| **Context Managers** | Resource lifecycle management with guaranteed cleanup |
| **Python 3.13** | `TypeIs`, `ReadOnly`, `@warnings.deprecated`; free-threading and JIT experimental |
| **Python 3.14** | Deferred annotations, t-strings, `except A, B:`; free-threaded build officially supported |
| **Python 3.15** | `lazy import`, `frozendict`, `typing.TypeForm`, UTF-8 default encoding |
| **uv + ruff** | uv: Pythons, venvs, lockfiles, dependency groups; ruff: lint + format; ty is still beta |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/python-versions.md, concepts/dataclasses.md, concepts/type-hints.md |
| **Intermediate** | concepts/generators.md, patterns/clean-architecture.md, patterns/project-tooling.md |
| **Advanced** | patterns/file-parser.md, patterns/error-handling.md, patterns/functional-patterns.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| python-developer | All files | Write clean, idiomatic Python for the target runtime |
| code-cleaner | concepts/type-hints.md, patterns/project-tooling.md | Modernize safely (ruff `UP` with the right target) |
| code-reviewer | patterns/clean-architecture.md, specs/python-standards.yaml | Review code quality |
| test-generator | patterns/error-handling.md, concepts/dataclasses.md | Generate typed test fixtures |

## Sources

- https://devguide.python.org/versions/
- https://docs.python.org/3.15/whatsnew/3.15.html
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
- https://docs.astral.sh/uv/
- https://docs.astral.sh/ruff/
- https://astral.sh/blog/ty
