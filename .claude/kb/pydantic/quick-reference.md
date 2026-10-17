# Pydantic Quick Reference

> **Purpose**: Fast lookup tables; code examples live in the linked files
> **Confidence**: 0.95
> **MCP Validated**: 2026-10-17

## Core Model Methods

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| `Model(**data)` / `model_validate(obj)` | kwargs / dict | instance | `strict=True` allowed per call |
| `model_validate_json(s)` | str/bytes | instance | Bad JSON -> `ValidationError` (`json_invalid`) |
| `model_dump()` / `model_dump_json()` | -- | dict / str | `exclude_none=True`, `by_alias=True` |
| `model_json_schema()` | -- | dict | Feed to LLM prompts |
| `model_copy(update={})` | dict | instance | `update` values are not validated |

## Field Type Patterns

| Pattern | Syntax | Use Case |
|---------|--------|----------|
| Required | `name: str` | Must be provided |
| Optional | `name: str \| None = None` | May be missing (needs the `= None`) |
| Constrained | `Annotated[int, Field(ge=1)]` | Numeric/length/pattern limits |
| Normalized string | `Annotated[str, StringConstraints(strip_whitespace=True, min_length=1)]` | Not possible via `Field()` |
| Literal / enum | `Literal["ok", "error"]` / `class C(StrEnum)` | Closed choice set |
| Description | `Field(description="...")` | Avoid `Field(..., ...)` for required |

## Validators

| Decorator | Mode | Receives | Use Case |
|-----------|------|----------|----------|
| `@field_validator` + `@classmethod` | `"after"` (default) / `"before"` / `"wrap"` | value | Field rules / coercion |
| `@model_validator` | `"before"` | raw input | Pre-process payload |
| `@model_validator` | `"after"` | instance, return `self` (`-> Self`) | Cross-field rules |
| `Annotated[T, AfterValidator(fn)]` | -- | value | Reusable rule |

## Version-Gated Features

| Feature | Min version | On DBR 17.3 LTS (2.10.6) |
|---------|-------------|--------------------------|
| `from_json(..., allow_partial=True)` | 2.7 | yes |
| `validate_by_name` / `validate_by_alias` / `serialize_by_alias` | 2.11 | no -> `populate_by_name=True` |
| `model_dump(exclude_computed_fields=True)` | 2.12 | no |
| `MISSING` sentinel | 2.12 experimental, 2.14 stable | no |
| Python 3.15 / `TypeForm` support | 2.14 | no |

## Unions and TypeAdapter

| Pattern | Syntax |
|---------|--------|
| Tagged union | `source: A \| B = Field(discriminator="kind")` |
| List of tagged variants | `list[Annotated[A \| B, Field(discriminator="kind")]]` |
| Non-model validation | `TypeAdapter(list[int]).validate_json("[1, 2]")` |

## Pydantic Settings (`pydantic-settings`)

| Feature | Syntax | Notes |
|---------|--------|-------|
| Env vars + prefix | `SettingsConfigDict(env_prefix="APP_")` | `APP_TIMEOUT` -> `timeout` |
| .env files | `env_file=(".env", ".env.prod")` | Later files override |
| Nested models | `env_nested_delimiter="__"` | `APP_DB__HOST` -> `db.host`; `APP_DB_HOST` is ignored |
| TOML / YAML | `toml_file=` / `yaml_file=` | Also override `settings_customise_sources` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| `Field(strip_whitespace=True)` | `StringConstraints(strip_whitespace=True)` (Field only warns, never strips) |
| `name: str \| None` with no default | `name: str \| None = None` |
| `Field(discriminator=...)` on a `list[...]` field | Discriminator on the union inside the list |
| `-> "MyModel"` in after-validators | `-> Self` |
| `populate_by_name=True` on 2.11+ | `validate_by_name=True` (+ `validate_by_alias=True`) |
| `@validator`, `.dict()`, inner `class Config`, `parse_obj_as()` (v1) | `@field_validator`, `.model_dump()`, `model_config`, `TypeAdapter` |
| Catch `json.JSONDecodeError` around `model_validate_json` | Catch `ValidationError` (type `json_invalid`) |
| `assert` in validators | `raise ValueError(...)` (`assert` is skipped under `-O`) |

## Related Documentation

| Topic | Path |
|-------|------|
| Strict mode and config | `concepts/strict-and-config.md` |
| Validators deep dive | `concepts/validators.md` |
| LLM output parsing | `patterns/llm-output-validation.md` |
| Full Index | `index.md` |

## Sources

- https://docs.pydantic.dev/latest/api/types/
- https://docs.pydantic.dev/latest/concepts/unions/
- https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- https://pydantic.dev/articles/pydantic-v2-14-release
