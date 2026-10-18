# BaseModel

> **Purpose**: Core building block for validated data schemas in Pydantic v2 (2.10+), with serialization, computed fields, TypeAdapter and discriminated unions
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Overview

`BaseModel` validates on instantiation, serializes to dict/JSON and generates JSON Schema (core in
Rust, `pydantic-core`). Examples run on 2.10.6 (preinstalled on Databricks Runtime 17.3 LTS) through
2.14.x; 2.11+ options are in [Strict Mode and Config](strict-and-config.md).

## The Pattern

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class Invoice(BaseModel):
    """Schema for extracted invoice data."""
    model_config = ConfigDict(str_strip_whitespace=True, validate_default=True)

    invoice_number: str = Field(description="Unique invoice identifier")   # required: no default
    vendor_name: str = Field(min_length=1, description="Name of the vendor")
    total_amount: float = Field(gt=0, description="Total invoice amount")
    currency: str = Field(default="USD", pattern=r"^[A-Z]{3}$")
    issue_date: datetime = Field(description="Date the invoice was issued")
    line_items: list[str] = Field(default_factory=list)
    notes: str | None = None
```

Omit the default to make a field required; the docs call `Field(..., description=...)` confusing.

## Quick Reference

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| `Invoice(**data)` | kwargs | Invoice | Validates on creation |
| `Invoice.model_validate(d)` | dict | Invoice | `strict=True` per call is allowed |
| `Invoice.model_validate_json(s)` | JSON str/bytes | Invoice | Invalid JSON -> `ValidationError` type `json_invalid` |
| `inv.model_dump(exclude_none=True)` | -- | dict | Skip `None` fields |
| `Invoice.model_json_schema()` | -- | dict | JSON Schema (use in LLM prompts) |
| `inv.model_copy(update={"currency": "EUR"})` | dict | Invoice | Clone; `update` is **not** validated |

## Common Mistakes

### Wrong (Pydantic v1 syntax)

```python
class MyModel(BaseModel):
    class Config:
        orm_mode = True

    def dict(self, **kwargs):  # v1 method
        return super().dict(**kwargs)
```

### Correct (Pydantic v2 syntax)

```python
class MyModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # v2: model_dump() replaces .dict()
```

## Computed Fields

```python
from pydantic import BaseModel, computed_field


class Order(BaseModel):
    items: list[dict]
    tax_rate: float = 0.1

    @computed_field
    @property
    def total(self) -> float:
        subtotal = sum(item["price"] * item["quantity"] for item in self.items)
        return round(subtotal * (1 + self.tax_rate), 2)


Order(items=[{"price": 10, "quantity": 2}]).model_dump()
# {'items': [...], 'tax_rate': 0.1, 'total': 22.0}
```

Computed fields appear in `model_dump()` and the serialization JSON Schema (marked `readOnly`);
`model_dump(exclude_computed_fields=True)` (2.12+) drops them.

## TypeAdapter (Non-BaseModel Types)

```python
from typing import Annotated

from pydantic import Field, TypeAdapter

adapter = TypeAdapter(list[int])
adapter.validate_python(["1", "2", "3"])   # [1, 2, 3]
adapter.json_schema()                      # {'items': {'type': 'integer'}, 'type': 'array'}
TypeAdapter(list[Annotated[int, Field(gt=0)]]).validate_json("[1, 2]")
```

## Discriminated Unions

```python
from typing import Literal

from pydantic import BaseModel, Field


class S3Source(BaseModel):
    source_type: Literal["s3"]
    bucket: str


class GCSSource(BaseModel):
    source_type: Literal["gcs"]
    bucket: str


class Pipeline(BaseModel):
    name: str
    source: S3Source | GCSSource = Field(discriminator="source_type")


p = Pipeline(name="ingest", source={"source_type": "s3", "bucket": "b"})
type(p.source).__name__   # 'S3Source' -- picked by tag, no trial-and-error
```

For a **list** of variants put the discriminator on the item type:
`list[Annotated[S3Source | GCSSource, Field(discriminator="source_type")]]`.

## Related

- [Field Types](../concepts/field-types.md)
- [Strict Mode and Config](../concepts/strict-and-config.md)
- [LLM Output Validation](../patterns/llm-output-validation.md)

## Sources

- https://docs.pydantic.dev/latest/concepts/models/
- https://docs.pydantic.dev/latest/api/base_model/
- https://docs.pydantic.dev/latest/concepts/fields/
- https://docs.pydantic.dev/latest/concepts/type_adapter/
- https://docs.pydantic.dev/latest/concepts/unions/
- https://docs.pydantic.dev/latest/errors/validation_errors/
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
