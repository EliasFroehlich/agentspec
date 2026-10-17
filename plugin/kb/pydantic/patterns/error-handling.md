# Error Handling

> **Purpose**: Handle Pydantic ValidationError gracefully in LLM extraction pipelines
> **MCP Validated**: 2026-10-17

## When to Use

- LLM returns malformed or partial JSON that fails validation
- Building retry logic when extraction fails schema validation
- Logging structured error information for monitoring
- Providing actionable feedback to retry with corrected prompts

## Implementation

```python
import logging
from collections.abc import Callable

from pydantic import BaseModel, ValidationError

logger = logging.getLogger(__name__)


class ExtractionResult(BaseModel):
    """Wrapper for extraction results with error metadata."""
    success: bool
    data: dict | None = None
    errors: list[dict] | None = None
    error_summary: str | None = None
    attempts: int = 1


def parse_validation_errors(error: ValidationError) -> list[dict]:
    """Convert ValidationError to structured error list."""
    return [
        {
            "field": " -> ".join(str(loc) for loc in err["loc"]) or "root",
            "message": err["msg"],
            "type": err["type"],
            "input": err.get("input"),
        }
        for err in error.errors()
    ]


def format_error_for_retry(error: ValidationError) -> str:
    """Format validation errors as LLM retry instructions."""
    if any(err["type"] == "json_invalid" for err in error.errors()):
        return "Your previous response was not valid JSON. Return ONLY a JSON object."
    lines = ["The previous response had validation errors. Fix these issues:"]
    for err in error.errors():
        field_path = " -> ".join(str(loc) for loc in err["loc"])
        lines.append(f"  - Field '{field_path}': {err['msg']}")
    lines.append("\nReturn corrected JSON only.")
    return "\n".join(lines)


def validate_with_retry[T: BaseModel](   # PEP 695 bound type parameter (Python 3.12+)
    call_llm_fn: Callable[[str, float], str],
    prompt: str,
    model_class: type[T],
    max_retries: int = 2,
    temperature: float = 0.0,
) -> ExtractionResult:
    """Validate LLM output with automatic retry on failure.

    model_validate_json raises ValidationError for both schema violations and
    malformed JSON (error type "json_invalid"), so one except clause covers both.
    """
    current_prompt = prompt
    attempts = max_retries + 1
    for attempt in range(1, attempts + 1):
        raw_response = call_llm_fn(current_prompt, temperature)
        try:
            result = model_class.model_validate_json(_extract_json(raw_response))
        except ValidationError as e:
            logger.warning("Attempt %d/%d failed: %d errors", attempt, attempts, e.error_count())
            if attempt == attempts:
                return ExtractionResult(
                    success=False,
                    errors=parse_validation_errors(e),
                    error_summary=f"{e.error_count()} validation errors after {attempt} attempts",
                    attempts=attempt,
                )
            current_prompt = f"{prompt}\n\n{format_error_for_retry(e)}"
        else:
            logger.info("Validation succeeded on attempt %d", attempt)
            return ExtractionResult(success=True, data=result.model_dump(), attempts=attempt)
    raise AssertionError("unreachable")


def _extract_json(text: str) -> str:
    """Strip markdown code fences from LLM response."""
    text = text.strip()
    if text.startswith("```"):
        lines = [line for line in text.split("\n")[1:] if line.strip() != "```"]
        text = "\n".join(lines)
    return text.strip()
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `max_retries` | `2` | Number of retry attempts after first failure |
| `temperature` | `0.0` | Lower temperature for more deterministic retries |
| `log_level` | `WARNING` | Log level for validation failures |

## Example Usage

```python
from pydantic import BaseModel, Field


class Receipt(BaseModel):
    merchant: str = Field(min_length=1)
    total: float = Field(gt=0)
    date: str = Field(pattern=r"^\d{4}-\d{2}-\d{2}$")


# With retry logic
result = validate_with_retry(
    call_llm_fn=my_llm_client.generate,
    prompt="Extract receipt data from: ...",
    model_class=Receipt,
    max_retries=2,
)

if result.success:
    print(f"Extracted: {result.data}")
else:
    print(f"Failed: {result.error_summary}")
    for err in result.errors or []:
        print(f"  {err['field']}: {err['message']}")
```

## Error Type Reference

| Error Type | Cause | Fix Strategy |
|------------|-------|-------------|
| `missing` | Required field not in JSON | Retry with explicit field list |
| `string_type` | Wrong type for string field | Retry with type instruction |
| `greater_than` | Number below minimum | Retry with constraint reminder |
| `json_invalid` | Not valid JSON at all (still a `ValidationError`) | Retry asking for JSON only |
| `string_pattern_mismatch` | Regex pattern failed | Retry with format example |

## See Also

- [LLM Output Validation](../patterns/llm-output-validation.md)
- [Validators](../concepts/validators.md)
- [Custom Validators](../patterns/custom-validators.md)

## Sources

- https://docs.pydantic.dev/latest/errors/validation_errors/
- https://docs.pydantic.dev/latest/concepts/models/
- https://peps.python.org/pep-0695/
