# Multi-Pass Extraction Pattern

> **Purpose**: Raise extraction accuracy with progressive passes (extract → review → verify), each pass schema-enforced, stopping early when confidence is high
> **MCP Validated**: 2026-10-17

## When to Use

- Dense documents (tables, nested structures, ambiguous layouts) where one pass misses or misreads fields
- High-value extraction where the cost of an error exceeds the cost of extra calls
- Not needed when a single structured-output call already meets your accuracy target — measure first

## Implementation

```python
from typing import Optional, TypeVar

import anthropic
from pydantic import BaseModel, Field

T = TypeVar("T", bound=BaseModel)
client = anthropic.Anthropic()

def ask(schema: type[T], prompt: str, model: str = "claude-sonnet-5-5") -> T:
    """One schema-enforced call. Swap this function to change provider."""
    resp = client.messages.parse(
        model=model, max_tokens=16000,
        messages=[{"role": "user", "content": prompt}],
        output_format=schema,
    )
    if resp.stop_reason != "end_turn":
        raise RuntimeError(f"pass stopped: {resp.stop_reason}")
    return resp.parsed_output

# --- Schemas per pass ---

class FieldValue(BaseModel):
    name: str
    value: Optional[str] = None
    confidence: float = Field(ge=0.0, le=1.0)

class Pass1(BaseModel):
    fields: list[FieldValue]

class Pass2(BaseModel):
    fields: list[FieldValue]
    corrections: list[str] = Field(default_factory=list)
    missed_fields: list[str] = Field(default_factory=list)

class Pass3(BaseModel):
    fields: list[FieldValue]
    discrepancies: list[str] = Field(default_factory=list)

class MultiPassResult(BaseModel):
    final: dict[str, Optional[str]]
    confidence: float
    passes: int
    issues: list[str]

# --- Prompts (document first, task last) ---

PASS_1 = """<document>
{document}
</document>

Extract every field present in the document with a confidence for each. Leave value null when absent."""

PASS_2 = """<document>
{document}
</document>

<previous_extraction>
{previous}
</previous_extraction>

Review each extracted field against the document. Fix errors, add missed fields, re-score confidence,
and list each correction you made."""

PASS_3 = """<document>
{document}
</document>

<extraction>
{previous}
</extraction>

Verify consistency: totals against line items, dates against each other, names and references across
the document. Return the verified fields and every discrepancy you found."""

def _avg(fields: list[FieldValue]) -> float:
    return sum(f.confidence for f in fields) / len(fields) if fields else 0.0

def multi_pass_extract(document: str, target: float = 0.95, max_passes: int = 3) -> MultiPassResult:
    issues: list[str] = []
    p1 = ask(Pass1, PASS_1.format(document=document))
    fields, passes = p1.fields, 1

    if _avg(fields) < target and max_passes >= 2:
        p2 = ask(Pass2, PASS_2.format(document=document, previous=p1.model_dump_json()))
        fields, passes = p2.fields, 2
        issues += p2.corrections

    if _avg(fields) < target and max_passes >= 3:
        p3 = ask(Pass3, PASS_3.format(document=document,
                                      previous=Pass1(fields=fields).model_dump_json()),
                 model="claude-opus-5-5")                 # strongest model for the final check
        fields, passes = p3.fields, 3
        issues += p3.discrepancies

    return MultiPassResult(final={f.name: f.value for f in fields},
                           confidence=_avg(fields), passes=passes, issues=issues)
```

Model-reported confidence is a heuristic, not a calibrated probability: calibrate the stop threshold on a labeled
sample before relying on it. OpenAI equivalent of `ask`: `client.responses.parse(model=..., input=prompt,
text_format=schema).output_parsed`.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `target` | `0.95` | Stop early when average confidence reaches it (calibrate first) |
| `max_passes` | `3` | Upper bound on calls per document |
| Pass 1–2 model | `claude-sonnet-5-5` | Extraction and review |
| Pass 3 model | `claude-opus-5-5` | Cross-field verification |
| Sampling params | none | Current Claude models reject `temperature`; consistency comes from schema + instructions |

## Example Usage

```python
result = multi_pass_extract(open("invoice.txt").read(), target=0.95)
print(result.passes, f"{result.confidence:.2f}", result.issues[:3])
```

## See Also

- [Document Extraction](../patterns/document-extraction.md)
- [Validation Prompts](../patterns/validation-prompts.md)
- [Chain-of-Thought and Thinking](../concepts/chain-of-thought.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.openai.com/docs/guides/structured-outputs
