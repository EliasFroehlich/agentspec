# Claude API Surface (current models)

> **Purpose**: Request shapes for the current Claude models — structured outputs, adaptive thinking/effort, parameters the current models reject, caching, and the agent surfaces
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Everything goes through `POST /v1/messages` (Python SDK `anthropic` 1.15.0 on PyPI as of 2026-10-17). The current family —
Claude Fable 5.1, Opus 5.5, Sonnet 5.5, Haiku 5.5 — rejects several shapes that older examples rely on: `temperature`/`top_p`/`top_k`,
assistant prefill, `budget_tokens`, and forced `tool_choice`. Structured outputs are GA, thinking is adaptive and steered with
`effort`, and a response can begin with a `thinking` block, so always select content blocks by `type`.

## The Concept

```python
import anthropic
from pydantic import BaseModel

class Invoice(BaseModel):
    invoice_number: str
    total_amount: float

client = anthropic.Anthropic()

# Structured output: the API enforces the schema (GA, no beta header)
resp = client.messages.parse(
    model="claude-opus-5-5",
    max_tokens=16000,                       # thinking counts toward max_tokens
    messages=[{"role": "user", "content": f"Extract the invoice:\n\n{doc}"}],
    output_format=Invoice,                  # SDK converts it to output_config.format
)
if resp.stop_reason == "refusal":            # HTTP 200 with a refusal: check first
    raise RuntimeError("request declined")
invoice = resp.parsed_output                # validated Invoice instance

# Free text: adaptive thinking + effort; read blocks by type, never content[0]
resp = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # display defaults to "omitted"
    output_config={"effort": "high"},       # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Explain the anomaly in these numbers: ..."}],
)
answer = "".join(b.text for b in resp.content if b.type == "text")
```

## Quick Reference

Models as of 2026-10-17 (prices per million tokens, input/output; [pricing](https://platform.claude.com/docs/en/about-claude/pricing)):

| Model | ID | Context | Price | Thinking |
|-------|----|---------|-------|----------|
| Claude Fable 5.1 | `claude-fable-5-1` | 1M | $10 / $50 | always on |
| Claude Opus 5.5 | `claude-opus-5-5` | 1M | $4 / $20 | always on; effort default `medium` |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | 1M | $2 / $10 | on by default; off = `{"type": "between_tools"}` |
| Claude Haiku 5.5 | `claude-haiku-5-5` | 1M | $0.10 / $0.50 (prompt ≤ 100K) | on by default |

Still served (previous generation): Opus 5, Opus 4.8/4.7/4.6, Sonnet 5, Sonnet 4.6, Haiku 4.5.
Retired: Claude 3.x models, Sonnet 4 and Opus 4 (2026-06-15), Opus 4.1 (2026-08-05). Sonnet 4.5 is deprecated and retires 2026-11-30.

| Request shape | Rejected on | Use instead |
|---------------|-------------|-------------|
| `temperature` / `top_p` / `top_k` | 400 on Opus 4.7+, Opus 5/5.5, Fable; non-default values 400 on Sonnet 5/5.5, Haiku 5.5 | Prompt instructions, structured outputs, `effort` |
| Trailing assistant prefill | 400 on every 4.6+ model, including Haiku 5.5 | `output_config.format`; "respond directly" in the system prompt |
| `thinking: {type: "enabled", budget_tokens}` | 400 on 4.7+ and the 5.x family; deprecated on 4.6 | `thinking: {type: "adaptive"}` + `output_config.effort` |
| `thinking: {type: "disabled"}` | 400 on Opus 5.5 (any effort) and Sonnet 5.5 | Lower `effort`; Sonnet 5.5: `between_tools` at effort ≤ `high` |
| `tool_choice` `any` / `tool` | 400 on Fable 5.1, Opus 5.5, Sonnet 5.5 (also `count_tokens`, Batches) | `auto` + `strict: true` + a prompt that names the tool, or `output_config.format` |
| `response.content[0].text` | Breaks when the first block is `thinking` | `b.type == "text"` |
| `tiktoken` for Claude token counts | Different tokenizer | `client.messages.count_tokens(...)` |

| Feature | Shape | Status |
|---------|-------|--------|
| Structured outputs | `output_config={"format": {"type": "json_schema", "schema": ...}}`, `messages.parse(output_format=Model)`, `strict: true` on tools | GA (top-level `output_format` on `create()` is deprecated) |
| Prompt caching | `cache_control={"type": "ephemeral"}` (optional `"ttl": "1h"`); write 1.25x (5 min) or 2x (1 h); read 0.1x, 0.05x on Opus/Sonnet 5.5, 0.025x on Fable 5.1 | GA |
| Message Batches | `client.messages.batches.create(...)`, 50% discount | GA |
| Mid-conversation `{"role": "system"}` message | Append to `messages` instead of editing `system`; keeps the cached prefix and is the operator channel user text cannot spoof (Opus 5/5.5, Opus 4.8, Fable 5/5.1, Sonnet 5.5; unsupported models return 400) | GA |
| Compaction | Server-side summarization, beta `compact-2026-01-12`; append `response.content`, not only text | Beta |
| Context editing | `context_management.edits` with `clear_tool_uses_20250919`, beta `context-management-2025-06-27` | Beta |
| MCP connector | `mcp_servers=[{"type": "url", ...}]` **and** `tools=[{"type": "mcp_toolset", ...}]`, beta `mcp-client-2025-11-20`; remote HTTP servers only | Beta |
| Tool search | `tool_search_tool_regex_20251119` / `_bm25_20251119` + `defer_loading: true` on large catalogs | GA |
| Files API, Skills | `client.files.*`, `client.skills.*` | GA |
| Refusal fallback | `stop_reason == "refusal"`; server-side `fallbacks` (beta) on the Claude API | Beta |

| Agent approach | Package / surface | Who hosts |
|----------------|-------------------|-----------|
| Manual tool loop | `anthropic` (`while stop_reason == "tool_use"`) | You |
| Tool Runner | `client.beta.messages.tool_runner` + `@beta_tool` | You (SDK runs the loop) |
| Claude Agent SDK | `claude-agent-sdk` 0.2.168 / `@anthropic-ai/claude-agent-sdk` — the Claude Code harness as a library | You |
| Managed Agents | REST, beta `managed-agents-2026-04-01` | Anthropic runs the loop and a per-session sandbox |

## Common Mistakes

### Wrong

```python
# Forced tool call to get JSON + temperature: both return 400 on Opus 5.5 / Sonnet 5.5
resp = client.messages.create(
    model="claude-sonnet-5-5", max_tokens=1024, temperature=0,
    tools=[{"name": "out", "input_schema": Invoice.model_json_schema()}],
    tool_choice={"type": "tool", "name": "out"},
    messages=[{"role": "user", "content": doc}],
)
data = resp.content[0].input
```

### Correct

```python
resp = client.messages.parse(
    model="claude-sonnet-5-5", max_tokens=16000,
    messages=[{"role": "user", "content": doc}],
    output_format=Invoice,
)
data = resp.parsed_output
```

## Related

- [Tool Calling](../concepts/tool-calling.md)
- [Multi-Agent Systems](../concepts/multi-agent-systems.md)
- [Output Formatting](../../prompt-engineering/concepts/output-formatting.md)
- [Chain-of-Thought and Thinking](../../prompt-engineering/concepts/chain-of-thought.md)

## Sources

- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/about-claude/model-deprecations
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking
- https://platform.claude.com/docs/en/build-with-claude/effort
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.claude.com/docs/en/build-with-claude/token-counting
- https://platform.claude.com/docs/en/agents-and-tools/mcp-connector
- https://code.claude.com/docs/en/agent-sdk/python
- https://pypi.org/project/anthropic/ ; https://pypi.org/project/claude-agent-sdk/
