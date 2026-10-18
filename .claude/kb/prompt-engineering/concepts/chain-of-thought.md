# Chain-of-Thought and Model Thinking

> **Purpose**: Get better reasoning from LLMs — built-in thinking controlled by effort on current models, manual chain-of-thought as a fallback, ReAct and self-consistency
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Chain-of-thought (CoT) means letting the model reason before it answers. Current frontier models do this natively:
Claude uses **adaptive thinking**, controlled by `output_config.effort` (`low` … `max`), and OpenAI reasoning models take
`reasoning={"effort": ...}` in the Responses API. With these, prefer general instructions ("think thoroughly about edge
cases") over hand-written step lists, and do not ask the model to write its reasoning into the answer — on Claude Fable 5.1,
Fable 5, Opus 5.5, Opus 5 and Sonnet 5.5 such prompts (for example "reason inside `<thinking>` tags") may be declined.
Manual CoT remains useful when thinking is off or the model has no reasoning mode.

## The Pattern: Claude adaptive thinking

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.create(
    model="claude-opus-5-5",                 # thinking is always on for Opus 5.5 and Fable 5.1
    max_tokens=16000,                        # thinking tokens count toward max_tokens
    thinking={"type": "adaptive", "display": "summarized"},   # default display is "omitted" (empty text)
    output_config={"effort": "high"},        # effort is the depth/cost lever
    messages=[{"role": "user", "content": f"Determine the quarterly growth rate and flag anomalies.\n\n{data}"}],
)
for block in resp.content:                   # select by type: a thinking block can come first
    if block.type == "thinking":
        log.debug(block.thinking)            # summary only; raw reasoning is never returned
    elif block.type == "text":
        print(block.text)
```

## The Pattern: OpenAI reasoning effort

```python
from openai import OpenAI

client = OpenAI()
result = client.responses.create(
    model="gpt-6-astra",
    reasoning={"effort": "high"},
    input=f"Determine the quarterly growth rate and flag anomalies.\n\n{data}",
)
print(result.output_text)
```

## The Pattern: Manual CoT (fallback when thinking is off)

```python
COT_PROMPT = """Analyze the financial data below and determine the quarterly growth rate.
Work through the problem before answering, then put only the final number in <answer> tags.

<data>
{data}
</data>"""

def growth_rate(llm, data: str) -> float:
    text = llm(COT_PROMPT.format(data=data))            # a model or route without native thinking
    return float(text.split("<answer>")[1].split("</answer>")[0])
```

## Quick Reference

| Situation | Approach |
|-----------|----------|
| Claude Opus 5.5 / Fable 5.1 | Thinking always on; tune `effort` (`low` for routine, `high`/`xhigh` for hard work) |
| Claude Sonnet 5.5 / Haiku 5.5 | Thinking on by default; lower `effort` instead of disabling |
| Older Claude (Haiku 4.5, pre-4.6) | `thinking={"type": "enabled", "budget_tokens": N}` still applies there only |
| OpenAI GPT-6 family | `reasoning={"effort": ...}` |
| Model or route without thinking | Manual CoT with the answer isolated in tags |
| Tool use, multi-step research | Agent frameworks implement ReAct-style loops; don't hand-write Thought/Action/Observation |
| Simple lookups, classification | Low effort; no CoT instructions |

## Self-Consistency

Sample several independent answers and take the majority. Current Claude models reject `temperature`, so variety comes
from independent calls (different few-shot orderings or phrasings), not from sampling knobs:

```python
from collections import Counter

def self_consistent(ask, prompt_variants: list[str]) -> str:
    """ask(prompt) -> final answer string; one call per variant."""
    answers = [ask(p) for p in prompt_variants]
    return Counter(answers).most_common(1)[0][0]
```

Use it only for high-stakes, short answers; it multiplies cost by the number of samples.

## Common Mistakes

### Wrong

```python
# Prescriptive steps + reasoning in the output + a thinking budget the model rejects
prompt = """Think step by step inside <thinking> tags:
1. First identify ... 2. Then compute ... 3. Finally ..."""
client.messages.create(model="claude-sonnet-5-5", max_tokens=8000,
                       thinking={"type": "enabled", "budget_tokens": 4000},   # 400 on 5.x models
                       messages=[{"role": "user", "content": prompt}])
```

### Correct

```python
client.messages.create(
    model="claude-sonnet-5-5", max_tokens=16000,
    output_config={"effort": "high"},
    messages=[{"role": "user", "content": "Compute the growth rate. Check the result against the raw totals "
                                          "before you finish.\n\n" + data}],
)
```

## Related

- [Structured Extraction](../concepts/structured-extraction.md)
- [Validation Prompts](../patterns/validation-prompts.md)
- [Claude API Surface](../../genai/concepts/anthropic-api-surface.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking
- https://platform.claude.com/docs/en/build-with-claude/effort
- https://developers.openai.com/api/docs/guides/code-generation
