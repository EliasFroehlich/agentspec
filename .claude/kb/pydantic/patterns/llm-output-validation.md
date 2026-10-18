# LLM Output Validation

> **Purpose**: Validate and parse LLM JSON responses into typed Pydantic models
> **MCP Validated**: 2026-10-17

## When to Use

- Parsing structured JSON output from any LLM (Gemini, GPT, Claude)
- Enforcing schema compliance on non-deterministic LLM responses
- Building reliable extraction pipelines that fail gracefully on malformed output
- Generating format instructions from Pydantic schemas to embed in prompts

## Implementation

```python
import json
import logging

from pydantic import BaseModel, ValidationError

logger = logging.getLogger(__name__)


def validate_llm_output[T: BaseModel](   # PEP 695 (Python 3.12+)
    raw_response: str,
    model_class: type[T],
    strict: bool = False,
) -> T:
    """Parse and validate LLM JSON output against a Pydantic model.

    Args:
        raw_response: Raw string from LLM (may contain markdown fences).
        model_class: Pydantic model class to validate against.
        strict: If True, disallow type coercion.

    Returns:
        Validated model instance.

    Raises:
        ValidationError: If the output does not match the schema, including
            malformed JSON (error type "json_invalid"; ValidationError subclasses ValueError).
    """
    cleaned = _extract_json(raw_response)
    return model_class.model_validate_json(cleaned, strict=strict)


def validate_llm_output_safe[T: BaseModel](
    raw_response: str,
    model_class: type[T],
) -> tuple[T | None, list[dict] | None]:
    """Non-raising version that returns (result, errors)."""
    try:
        return validate_llm_output(raw_response, model_class), None
    except ValidationError as e:   # covers schema errors and json_invalid
        logger.warning("Validation failed: %s errors", e.error_count())
        return None, e.errors()


def _extract_json(text: str) -> str:
    """Strip markdown code fences and whitespace from LLM output."""
    text = text.strip()
    if text.startswith("```"):
        lines = text.split("\n")
        # Remove first line (```json) and last line (```)
        lines = [l for l in lines[1:] if l.strip() != "```"]
        text = "\n".join(lines)
    return text.strip()


def build_format_instruction(model_class: type[BaseModel]) -> str:
    """Generate LLM prompt instructions from a Pydantic model schema."""
    schema = model_class.model_json_schema()
    return (
        "You must respond with valid JSON matching this schema. "
        "Do not include any text outside the JSON object.\n\n"
        f"```json\n{json.dumps(schema, indent=2)}\n```"
    )
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `strict` | `False` | When True, disables type coercion (string "5" won't become int 5) |
| `strip_fences` | `True` | Remove markdown code fences from LLM output |
| `temperature` | `0.0` | Recommended LLM temperature for structured output |

## Example Usage

```python
from typing import Literal

from pydantic import BaseModel, Field


class ExtractedEntity(BaseModel):
    name: str = Field(description="Entity name")
    entity_type: Literal["person", "org", "location"] = Field(description="Entity category")
    confidence: float = Field(ge=0, le=1, description="Extraction confidence")
    context: str | None = Field(default=None, description="Surrounding text snippet")


# 1. Generate prompt instructions
instructions = build_format_instruction(ExtractedEntity)

# 2. Send to any LLM client
prompt = f"Extract entities from: 'John works at Acme in NYC'\n\n{instructions}"
llm_response = call_llm(prompt)  # returns JSON string

# 3. Validate the response
entity, errors = validate_llm_output_safe(llm_response, ExtractedEntity)
if entity:
    print(f"Found: {entity.name} ({entity.entity_type})")
else:
    print(f"Validation errors: {errors}")


# 4. Batch validation for list outputs
class EntityList(BaseModel):
    entities: list[ExtractedEntity]

batch_result, errors = validate_llm_output_safe(llm_response, EntityList)
```

## Partial JSON Parsing (Pydantic v2.7+)

```python
from pydantic_core import from_json

# Useful for streaming LLM responses
partial_json = '{"name": "John", "entity_type": "pers'
from_json(partial_json, allow_partial=True)                # {'name': 'John'} -- unfinished string dropped
from_json(partial_json, allow_partial="trailing-strings")  # {'name': 'John', 'entity_type': 'pers'}
# Validate with model_validate() only when the stream is complete, or give late-arriving fields defaults
```

## TypeAdapter for List Validation

```python
from pydantic import TypeAdapter

# Validate a list of entities without wrapping in a BaseModel
EntityListAdapter = TypeAdapter(list[ExtractedEntity])

# Parse LLM output that returns a JSON array directly
entities = EntityListAdapter.validate_json(llm_response)

# Generate JSON Schema for arrays
schema = EntityListAdapter.json_schema()
```

## Discriminated Unions for Multi-Type LLM Output

```python
from typing import Annotated, Literal

from pydantic import BaseModel, Field


class PersonEntity(BaseModel):
    entity_type: Literal["person"]
    name: str
    role: str | None = None


class OrgEntity(BaseModel):
    entity_type: Literal["org"]
    name: str
    industry: str | None = None


# The discriminator goes on the union *inside* the list. Putting Field(discriminator=...)
# on the list field itself raises TypeError when the class is defined.
Entity = Annotated[PersonEntity | OrgEntity, Field(discriminator="entity_type")]


class EntityResult(BaseModel):
    entities: list[Entity]


result = EntityResult.model_validate_json(llm_response)   # unknown tag -> union_tag_invalid
```

## See Also

- [Error Handling](../patterns/error-handling.md)
- [Extraction Schema](../patterns/extraction-schema.md)
- [BaseModel](../concepts/base-model.md)

## Sources

- https://docs.pydantic.dev/latest/concepts/json/
- https://docs.pydantic.dev/latest/api/pydantic_core/
- https://docs.pydantic.dev/latest/concepts/unions/
- https://docs.pydantic.dev/latest/concepts/type_adapter/
- https://docs.pydantic.dev/latest/errors/validation_errors/
