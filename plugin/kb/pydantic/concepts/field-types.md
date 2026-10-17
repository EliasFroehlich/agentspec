# Field Types

> **Purpose**: Type annotations, optional fields, Annotated, Field and StringConstraints in Pydantic v2
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Pydantic v2 uses Python type annotations to define field types and applies validation
automatically. The `Field()` function adds metadata, constraints, and descriptions.
The `Annotated` type allows attaching validators and constraints directly to types,
making them reusable across models. String normalization (strip, case) is **not** a `Field()`
argument: use `StringConstraints` or `ConfigDict(str_strip_whitespace=True)`.

## The Pattern

```python
from datetime import date
from typing import Annotated, Literal

from pydantic import BaseModel, Field, StringConstraints

# Reusable annotated types (StringConstraints exists in 2.10.6, the DBR 17.3 LTS version)
NonEmptyStr = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]
UpperCode = Annotated[str, StringConstraints(strip_whitespace=True, to_upper=True, min_length=3, max_length=3)]
PositiveFloat = Annotated[float, Field(gt=0)]


class LineItem(BaseModel):
    description: NonEmptyStr = Field(description="Item description")   # "  x " -> "x"; "   " fails
    quantity: Annotated[int, Field(ge=1)] = 1
    unit_price: PositiveFloat
    currency: UpperCode = "USD"                                          # " brl" -> "BRL"
    category: str | None = None


class ExtractionResult(BaseModel):
    confidence: Annotated[float, Field(ge=0.0, le=1.0)]
    source: Literal["invoice", "receipt", "contract"]
    extracted_date: date | None = None
    items: list[LineItem] = Field(default_factory=list)
    tags: set[str] = Field(default_factory=set)
```

## Quick Reference

| Type Annotation | Meaning | Example Value |
|-----------------|---------|---------------|
| `str` | Required string | `"hello"` |
| `str \| None` | String or None (still required unless `= None`) | `None` |
| `int` | Required integer | `42` |
| `float` | Required float (accepts int too) | `3.14` |
| `bool` | Boolean | `True` |
| `list[str]` | List of strings | `["a", "b"]` |
| `dict[str, Any]` | Dictionary | `{"key": "val"}` |
| `set[str]` | Unique set of strings | `{"a", "b"}` |
| `Literal["a", "b"]` | Constrained choices | `"a"` |
| `date` / `datetime` | Date objects | `"2026-01-15"` |
| `Annotated[str, StringConstraints(...)]` | Strip / case / length / pattern | `"ABC"` |

## Field() Constraints

| Parameter | Types | Purpose | Example |
|-----------|-------|---------|---------|
| `min_length` | str, list | Minimum length | `Field(min_length=1)` |
| `max_length` | str, list | Maximum length | `Field(max_length=100)` |
| `pattern` | str | Regex pattern | `Field(pattern=r"^\d+$")` |
| `gt` / `ge` | int, float | Greater than / or equal | `Field(gt=0)` |
| `lt` / `le` | int, float | Less than / or equal | `Field(le=100)` |
| `multiple_of` | int, float | Must be multiple of | `Field(multiple_of=5)` |
| `description` | Any | Field description for schema | `Field(description="...")` |
| `alias` | Any | Alternative name for parsing | `Field(alias="fieldName")` |
| `default` | Any | Default value | `Field(default="USD")` |
| `default_factory` | Any | Factory for mutable defaults | `Field(default_factory=list)` |
| `exclude` | Any | Exclude from serialization | `Field(exclude=True)` |
| `strict` | Any | No coercion for this field | `Field(strict=True)` |

`Field()` has no `strip_whitespace`, `to_lower` or `to_upper`: unknown keywords only raise a
`PydanticDeprecatedSince20` warning ("extra keyword arguments") and are **silently not applied**.
In `StringConstraints`, `pattern` is matched after `strip_whitespace` but **before** `to_upper`/
`to_lower`, so `" brl"` fails `^[A-Z]{3}$` (observed on 2.10.6 and 2.14.1): check length, or the
case in an `AfterValidator`.

## Common Mistakes

### Wrong

```python
class Model(BaseModel):
    name: str | None                                         # REQUIRED: v2 adds no implicit None default
    code: str = Field(min_length=1, strip_whitespace=True)   # warning only, value NOT stripped
```

### Correct

```python
class Model(BaseModel):
    name: str | None = None
    code: Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]
```

## LLM Schema Generation

```python
import json

class Entity(BaseModel):
    name: NonEmptyStr = Field(description="Entity name")
    entity_type: Literal["person", "org", "location"]
    confidence: Annotated[float, Field(ge=0, le=1, description="Extraction confidence")]

# Field descriptions appear in JSON Schema for LLM prompts
schema = Entity.model_json_schema()
# {"properties": {"name": {"description": "Entity name", ...}}}
```

## Related

- [BaseModel](../concepts/base-model.md)
- [Validators](../concepts/validators.md)
- [Extraction Schema](../patterns/extraction-schema.md)

## Sources

- https://docs.pydantic.dev/latest/api/types/
- https://docs.pydantic.dev/latest/api/fields/
- https://docs.pydantic.dev/latest/concepts/fields/
