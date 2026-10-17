# Pydantic Knowledge Base

> **Purpose**: Pydantic v2 data validation -- models, fields, validators, strict mode, TypeAdapter, discriminated unions, settings, LLM output
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Version Check First

Latest is **2.14.1** (2026-10-11; 2.14 drops Python 3.9 and supports 3.15). Databricks Runtime 17.3
LTS preinstalls **2.10.6**, so the alias API (2.11), `exclude_computed_fields` (2.12) and the stable
`MISSING` sentinel (2.14) are unavailable there unless you install a newer version. Every example in
this KB runs on 2.10.6 unless a minimum version is given. V3 has no release date; deprecations are
only removed in V3.

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/base-model.md](concepts/base-model.md) | BaseModel methods, serialization, computed fields, TypeAdapter, unions |
| [concepts/field-types.md](concepts/field-types.md) | Field types, `X \| None`, Annotated, Field and StringConstraints |
| [concepts/validators.md](concepts/validators.md) | field_validator, model_validator, modes, `-> Self` |
| [concepts/nested-models.md](concepts/nested-models.md) | Nested model composition, error paths |
| [concepts/strict-and-config.md](concepts/strict-and-config.md) | ConfigDict, strict mode, alias API 2.11+, validate_call, MISSING, version gates |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/llm-output-validation.md](patterns/llm-output-validation.md) | Validate and parse LLM JSON responses, partial JSON |
| [patterns/extraction-schema.md](patterns/extraction-schema.md) | Build schemas for document data extraction |
| [patterns/error-handling.md](patterns/error-handling.md) | Handle ValidationError, retries, fallbacks |
| [patterns/custom-validators.md](patterns/custom-validators.md) | Reusable custom validation logic |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/invoice-schema.yaml](specs/invoice-schema.yaml) | Invoice extraction Pydantic schema spec |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **BaseModel** | Core class for data schemas with automatic validation (lax by default) |
| **Field Types** | Annotations with `X \| None`, `Annotated`, `Field()` constraints, `StringConstraints` |
| **Validators** | `@field_validator` / `@model_validator`; after-validators return `self` (`-> Self`) |
| **Strict mode** | `ConfigDict(strict=True)`, `Field(strict=True)` or `model_validate(..., strict=True)` |
| **Alias API (2.11+)** | `validate_by_name`, `validate_by_alias`, `serialize_by_alias`; `populate_by_name` on 2.10 |
| **Computed Fields** | `@computed_field` for derived values included in serialization/schema |
| **TypeAdapter** | Validate non-BaseModel types (lists, dicts, unions) |
| **Discriminated Unions** | `Field(discriminator=...)` on the union (inside `list[...]` for lists) |
| **validate_call** | Validate function arguments with type hints |
| **Pydantic Settings** | `BaseSettings` for env vars, .env, TOML, YAML; nested via `env_nested_delimiter` |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/base-model.md, concepts/field-types.md |
| **Intermediate** | concepts/validators.md, concepts/strict-and-config.md, patterns/llm-output-validation.md |
| **Advanced** | patterns/extraction-schema.md, patterns/custom-validators.md, patterns/error-handling.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| ai-prompt-specialist-gcp | patterns/llm-output-validation.md, patterns/extraction-schema.md | Define Pydantic schemas for LLM structured output |
| python-developer | concepts/base-model.md, concepts/validators.md, concepts/strict-and-config.md | Build validated data models |
| data-quality-analyst | patterns/custom-validators.md | Define data quality checks with Pydantic |

## Sources

- https://docs.pydantic.dev/latest/changelog/
- https://pydantic.dev/articles/pydantic-v2-14-release
- https://pydantic.dev/articles/pydantic-v2-11-release
- https://pydantic.dev/docs/validation/dev/get-started/version-policy
- https://docs.databricks.com/aws/en/release-notes/runtime/17.3lts
