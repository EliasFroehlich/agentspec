# Output Formatting

> **Purpose**: Get schema-valid structured output from LLMs with provider-native structured outputs (Claude, OpenAI, Gemini), Instructor, and validation
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

All three major APIs now constrain output to a JSON Schema natively: Claude structured outputs (`output_config.format`,
`client.messages.parse`; GA), OpenAI Structured Outputs (Responses API `text.format` / `client.responses.parse`), and Gemini
`response_schema`. Use them instead of "return only JSON" prompts, JSON mode, assistant prefill, or forcing a tool call —
on Claude Opus 5.5, Sonnet 5.5 and Fable 5.1 a forced `tool_choice` returns 400, and prefill returns 400 on every 4.6+
model. Pydantic stays as the type definition and for business-rule validation after parsing.

## The Pattern: Claude

```python
import anthropic
from pydantic import BaseModel, Field

class TaskResult(BaseModel):
    status: str = Field(description="completed | failed | partial")
    summary: str
    confidence: float = Field(ge=0.0, le=1.0)   # SDK moves ge/le into the description and re-validates
    errors: list[str] = Field(default_factory=list)

client = anthropic.Anthropic()
resp = client.messages.parse(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": f"Analyze:\n\n{data}"}],
    output_format=TaskResult,
)
if resp.stop_reason in ("refusal", "max_tokens"):     # no valid object in these cases
    raise RuntimeError(resp.stop_reason)
result: TaskResult = resp.parsed_output
```

Raw JSON Schema instead of Pydantic: `client.messages.create(..., output_config={"format": {"type": "json_schema",
"schema": SCHEMA}})`, then `json.loads` the first `text` block. Objects need `"additionalProperties": false`.

## The Pattern: OpenAI (Responses API)

```python
from openai import OpenAI

client = OpenAI()
resp = client.responses.parse(
    model="gpt-6-astra",
    input=[{"role": "system", "content": "Analyze the data."},
           {"role": "user", "content": data}],
    text_format=TaskResult,
)
result = resp.output_parsed
```

Raw schema: `client.responses.create(..., text={"format": {"type": "json_schema", "name": "task_result",
"schema": SCHEMA, "strict": True}})`. OpenAI recommends Structured Outputs over JSON mode whenever possible.

## The Pattern: Gemini (`google-genai`)

```python
from google import genai
from google.genai import types

client = genai.Client()
resp = client.models.generate_content(
    model="gemini-3.8-flash",
    contents=f"Analyze:\n\n{data}",
    config=types.GenerateContentConfig(response_mime_type="application/json", response_schema=TaskResult),
)
result = resp.parsed
```

## The Pattern: Instructor (one interface, many providers)

```python
import instructor

client = instructor.from_provider("anthropic/claude-sonnet-5", mode=instructor.Mode.TOOLS)
result = client.create(
    max_tokens=1024, response_model=TaskResult,
    messages=[{"role": "user", "content": f"Analyze:\n\n{data}"}],
)
```

`Mode.TOOLS` relies on tool calling. Before using it with models that reject forced `tool_choice` (Opus 5.5, Sonnet 5.5,
Fable 5.1), confirm the Instructor version supports them, or use the native `messages.parse` path.

## Structured Output Methods

| Method | Provider | Schema enforced by | Use when |
|--------|----------|--------------------|----------|
| `messages.parse` / `output_config.format` | Anthropic | API (GA) | Default for Claude |
| `strict: true` on tool definitions | Anthropic, OpenAI | API | Arguments of tools the model chooses to call |
| `responses.parse` / `text.format` | OpenAI | API | Default for OpenAI |
| `response_schema` | Google Gemini | API | Default for Gemini |
| Instructor (`from_provider`) | Many | Library (tools/JSON) + Pydantic retries | One code path across providers |
| `ai_extract` / `ai_query(..., responseFormat => ...)` | Databricks SQL | Platform | Batch extraction inside Spark/SQL |
| JSON mode / prompt-only | Any | Nothing | Prototypes only |

Notes: Claude citations cannot be combined with `output_config.format` (400). JSON Schema support is a subset on each
provider — every object needs explicit `properties` (`additionalProperties: false`), so model key/value data as a list
of `{name, value}` objects instead of `dict`/map fields, and check each provider's limitations page.

## Common Mistakes

### Wrong

```python
# Prompt-only JSON + manual repair loop
text = llm("Return the data as JSON only, no markdown")
data = json.loads(text.strip("```json").strip("```"))
```

### Correct

```python
resp = client.messages.parse(model="claude-sonnet-5-5", max_tokens=16000,
                             messages=[{"role": "user", "content": prompt}], output_format=MySchema)
data = resp.parsed_output
```

## Related

- [Structured Extraction](../concepts/structured-extraction.md)
- [Document Extraction Pattern](../patterns/document-extraction.md)
- [Claude API Surface](../../genai/concepts/anthropic-api-surface.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.openai.com/docs/guides/structured-outputs
- https://github.com/googleapis/python-genai
- https://python.useinstructor.com/integrations/anthropic/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_extract
