# Validation Prompts Pattern

> **Purpose**: Verify LLM extractions against the source before returning them — a schema-enforced audit pass with evidence quotes, plus consistency checks across independent runs
> **MCP Validated**: 2026-10-17

## When to Use

- High-stakes extraction (financial, medical, legal) where errors have real cost
- You need auditable evidence for every extracted value
- Reducing unsupported values before data reaches downstream systems
- Offline quality measurement belongs in an evaluation framework (MLflow scorers, RAGAS); this pattern runs inline

## Implementation

```python
from typing import Optional

import anthropic
from pydantic import BaseModel, Field

client = anthropic.Anthropic()

class NamedValue(BaseModel):
    name: str
    value: Optional[str] = None

class Extracted(BaseModel):          # lists of objects, not dict/map fields: schema-enforced
    fields: list[NamedValue]         # outputs need explicit properties on every object

class FieldCheck(BaseModel):
    field_name: str
    extracted_value: Optional[str]
    is_supported: bool
    evidence: Optional[str] = Field(None, description="Exact quote from the document")
    correction: Optional[str] = Field(None, description="Corrected value when the extraction is wrong")
    confidence: float = Field(ge=0.0, le=1.0)

class Validation(BaseModel):
    fields: list[FieldCheck]
    flagged_issues: list[str] = Field(default_factory=list)

def _parse(schema, prompt: str, model: str):
    resp = client.messages.parse(model=model, max_tokens=16000,
                                 messages=[{"role": "user", "content": prompt}], output_format=schema)
    if resp.stop_reason != "end_turn":
        raise RuntimeError(resp.stop_reason)
    return resp.parsed_output

EXTRACT = """<document>
{document}
</document>

Extract these fields; use null when a field is not in the document:
{field_list}"""

VALIDATE = """<document>
{document}
</document>

<extracted>
{extracted}
</extracted>

Audit each extracted field against the document. For each one, quote the exact supporting text,
mark whether it is supported, and give the corrected value when it is wrong. Flag anything that
could be confused with another value in the document."""

def extract_and_validate(document: str, fields: list[str], threshold: float = 0.85) -> Validation:
    extracted = _parse(Extracted, EXTRACT.format(document=document,
                                                 field_list="\n".join(f"- {f}" for f in fields)),
                       model="claude-sonnet-5-5")
    result = _parse(Validation, VALIDATE.format(document=document,
                                                extracted=extracted.model_dump_json()),
                    model="claude-opus-5-5")                    # independent, stronger auditor
    for f in result.fields:
        if not f.is_supported and f.correction and f.confidence >= threshold:
            f.extracted_value, f.is_supported = f.correction, True
        elif f.confidence < threshold:
            result.flagged_issues.append(f"{f.field_name}: needs human review")
    return result
```

The extract and audit passes use different models (or at least separate calls without shared context) so the auditor does
not just agree with its own earlier output.

## Consistency Check Across Runs

Current Claude models reject `temperature`, so variance comes from independent calls and prompt variants rather than
sampling knobs:

```python
from collections import Counter

def consistency(document: str, fields: list[str], variants: list[str]) -> dict[str, dict]:
    """variants: alternative phrasings of the extraction instruction."""
    runs = [{nv.name: nv.value for nv in
             _parse(Extracted, f"<document>\n{document}\n</document>\n\n{v}\n" + "\n".join(fields),
                    model="claude-haiku-5-5").fields} for v in variants]
    report = {}
    for name in fields:
        values = [r.get(name) for r in runs]
        top, count = Counter(values).most_common(1)[0]
        report[name] = {"value": top, "agreement": count / len(values), "all_values": values}
    return report
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `threshold` | `0.85` | Below this, route the field to human review instead of auto-correcting |
| Extract model | `claude-sonnet-5-5` | First pass |
| Audit model | `claude-opus-5-5` | Independent verification pass |
| Consistency runs | 3 prompt variants | Agreement < 1.0 marks a field as unstable |

## Example Usage

```python
doc = "Invoice #1234, dated 2024-03-15, from Acme Corp, total $5,250.00 USD"
result = extract_and_validate(doc, ["invoice_number", "date", "vendor", "total_amount", "currency"])
for f in result.fields:
    print(f"[{'OK' if f.is_supported else 'FLAG'}] {f.field_name}={f.extracted_value} "
          f"({f.confidence:.2f}) evidence={f.evidence!r}")
```

## See Also

- [Chain-of-Thought and Thinking](../concepts/chain-of-thought.md)
- [Multi-Pass Extraction](../patterns/multi-pass-extraction.md)
- [Evaluation Framework](../../genai/patterns/evaluation-framework.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/about-claude/pricing
