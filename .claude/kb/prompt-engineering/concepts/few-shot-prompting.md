# Few-Shot Prompting

> **Purpose**: Teach LLMs the expected output format, tone and edge-case behavior through in-context examples
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Few-shot (multishot) prompting puts a few input → output examples in the prompt so the model copies their structure,
tone and decisions. Examples are one of the most reliable ways to steer format; Anthropic recommends 3–5 relevant,
diverse examples wrapped in `<example>` tags (inside `<examples>`) so they are not confused with instructions. Examples
also work with thinking models: they shape how the model approaches similar problems. When the API enforces a schema
(structured outputs), examples are for judgment and edge cases, not for teaching JSON syntax.

## The Pattern

```python
from typing import Literal

import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()

class EmailLabel(BaseModel):
    category: Literal["order_update", "security_alert", "meeting", "other"]
    priority: Literal["low", "medium", "high", "critical"]
    action: Literal["none", "respond", "escalate"]

FEW_SHOT = """Classify the email.

<examples>
<example>
<input>Your order #12345 has shipped and will arrive by Friday.</input>
<output>{"category": "order_update", "priority": "low", "action": "none"}</output>
</example>
<example>
<input>URGENT: unusual sign-in detected, reset your password now.</input>
<output>{"category": "security_alert", "priority": "critical", "action": "escalate"}</output>
</example>
<example>
<input>Can we move Tuesday's sync to 3pm?</input>
<output>{"category": "meeting", "priority": "medium", "action": "respond"}</output>
</example>
</examples>

<email>
{email_text}
</email>"""

def classify_email(email_text: str) -> EmailLabel:
    resp = client.messages.parse(
        model="claude-haiku-5-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": FEW_SHOT.replace("{email_text}", email_text)}],
        output_format=EmailLabel,          # schema enforced by the API; examples teach the judgment
    )
    return resp.parsed_output
```

## Quick Reference

| Shot count | When to use | Token cost |
|------------|-------------|------------|
| Zero-shot | Simple, well-defined tasks with an enforced schema | Lowest |
| One-shot | Show one format or tone | Low |
| Few-shot (3–5) | Classification boundaries, tone, edge cases | Medium |
| Dynamic few-shot | Large example bank; select by similarity per request | Medium + retrieval |

## Example Selection Rules

1. **Cover edge cases** — include at least one tricky example
2. **Show every output class** — represent each category you expect
3. **Consistent format** — all examples follow the same structure
4. **Diverse inputs** — vary length, style and difficulty so the model does not copy one surface pattern
5. **Label them as illustrative** — the model mirrors examples closely; avoid a single "gold" output it will clone
6. **Do not duplicate the schema** — with structured outputs the API carries the schema (Google's Gen AI SDK docs warn that repeating it in the prompt can lower output quality)

## Common Mistakes

### Wrong

```python
# Inconsistent formats teach the model nothing reliable
examples = """
Example 1: The sentiment is positive
Example 2: {"sentiment": "negative"}
Example 3: Neutral sentiment detected
"""
```

### Correct

```python
examples = """<examples>
<example><input>I love this product!</input><output>positive</output></example>
<example><input>Terrible experience, never again</input><output>negative</output></example>
<example><input>The package arrived on time</input><output>neutral</output></example>
</examples>"""
```

## Dynamic Few-Shot Selection

```python
import numpy as np

def select_examples(query_vec: np.ndarray, bank: list[dict], k: int = 3) -> list[dict]:
    """bank items: {"input": str, "output": str, "vec": np.ndarray (normalized)}."""
    scores = [float(query_vec @ ex["vec"]) for ex in bank]
    top = np.argsort(scores)[::-1][:k]
    return [bank[i] for i in top]

def render_examples(examples: list[dict]) -> str:
    body = "\n".join(f"<example><input>{e['input']}</input><output>{e['output']}</output></example>"
                     for e in examples)
    return f"<examples>\n{body}\n</examples>"
```

Keep the selected examples in a stable order (for example by ID) so the cached prompt prefix stays reusable.

## Related

- [System Prompts](../concepts/system-prompts.md)
- [Output Formatting](../concepts/output-formatting.md)
- [Prompt Template Pattern](../patterns/prompt-template.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://github.com/googleapis/python-genai (README: "don't duplicate [the schema] in your input prompt")
