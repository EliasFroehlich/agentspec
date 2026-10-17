# Strict Mode, ConfigDict and Version-Gated Features

> **Purpose**: Model configuration, strict vs lax validation, the 2.11 alias API, `validate_call`, `MISSING`, and which Pydantic version each needs
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

Pydantic validates in **lax** mode by default (`"5"` becomes `5`). `ConfigDict` sets model-wide
behavior; strictness can also be set per field or per call. Several options are recent: check the
version you deploy. Pydantic is at **2.14.1** (2026-10-11), but Databricks Runtime 17.3 LTS
preinstalls **2.10.6**, which lacks everything marked 2.11+ below unless you `%pip install` a newer one.

## The Concept

```python
from pydantic import BaseModel, ConfigDict, Field, ValidationError


class Order(BaseModel):
    model_config = ConfigDict(
        strict=True,                 # no coercion: "5" is rejected for int
        str_strip_whitespace=True,   # strip every str field
        extra="forbid",              # unknown keys raise instead of being ignored
        validate_by_name=True,       # 2.11+: accept the field name...
        validate_by_alias=True,      # 2.11+: ...and the alias (the default)
        serialize_by_alias=True,     # 2.11+: model_dump() emits aliases
    )

    order_id: int = Field(alias="orderId")
    customer: str


Order(order_id=1, customer="  Acme ")          # by name -> customer == "Acme"
Order.model_validate({"orderId": 1, "customer": "Acme"}).model_dump()   # {'orderId': 1, ...}
try:
    Order(order_id="1", customer="Acme")
except ValidationError as e:
    print(e.errors()[0]["type"])                # 'int_type'
```

## Quick Reference

| Option | Default | Since | Notes |
|--------|---------|-------|-------|
| `strict` | `False` | 2.0 | Also `Field(strict=True)` and `model_validate(data, strict=True)` |
| `str_strip_whitespace` | `False` | 2.0 | Model-wide; per field use `StringConstraints(strip_whitespace=True)` |
| `extra` | `"ignore"` | 2.0 | `"forbid"` / `"allow"` |
| `validate_default` | `False` | 2.0 | Validate default values too |
| `from_attributes` | `False` | 2.0 | Build from ORM objects / attributes |
| `validate_by_alias` | `True` | 2.11 | Accept the alias when validating |
| `validate_by_name` | `False` | 2.11 | Accept the field name when validating |
| `serialize_by_alias` | `False` | 2.11 | Dump with aliases by default |
| `populate_by_name` | `False` | 2.0 | Only option on 2.10; "not recommended" from 2.11, slated for deprecation in V3 |

## Version-Gated Features

| Feature | Since | Example |
|---------|-------|---------|
| Partial JSON parsing | 2.7 | `pydantic_core.from_json(raw, allow_partial=True)` |
| Alias API above | 2.11 | `ConfigDict(validate_by_name=True)` |
| `exclude_computed_fields` | 2.12 | `m.model_dump(exclude_computed_fields=True)` |
| `MISSING` sentinel | 2.12 experimental, 2.14 stable | `from pydantic import MISSING` (2.14+) |
| `TypeForm` support, Python 3.15 | 2.14 | 2.14 also drops Python 3.9 |

## Validating Function Arguments

```python
from pydantic import validate_call


@validate_call
def scale(x: int, factor: float = 1.0) -> float:
    return x * factor


scale("3", factor="2")   # 6.0 -- arguments are coerced (lax) and validated
scale("abc")             # ValidationError: int_parsing
```

## Common Mistakes

### Wrong

```python
# Works on 2.11+; on DBR 17.3 LTS (pydantic 2.10.6) the unknown key is silently ignored,
# so Order(order_id=1) still fails with "missing" -- no warning at class definition.
model_config = ConfigDict(validate_by_name=True)
```

### Correct

```python
import pydantic

if tuple(int(p) for p in pydantic.VERSION.split(".")[:2]) >= (2, 11):
    config = ConfigDict(validate_by_name=True, validate_by_alias=True)
else:
    config = ConfigDict(populate_by_name=True)   # 2.10 and earlier
```

Simpler: pin one version everywhere (`pydantic==2.10.6` locally to match the cluster, or install
the same newer version on the cluster) instead of branching.

## Related

- [BaseModel](../concepts/base-model.md)
- [Field Types](../concepts/field-types.md)
- [Validators](../concepts/validators.md)

## Sources

- https://docs.pydantic.dev/latest/api/config/
- https://docs.pydantic.dev/latest/concepts/strict_mode/
- https://docs.pydantic.dev/latest/concepts/validation_decorator/
- https://pydantic.dev/docs/validation/2.11/api/pydantic/config/
- https://pydantic.dev/articles/pydantic-v2-11-release
- https://pydantic.dev/articles/pydantic-v2-14-release
- https://docs.pydantic.dev/latest/concepts/json/
- https://docs.pydantic.dev/latest/changelog/
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
