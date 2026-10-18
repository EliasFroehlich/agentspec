# System Prompts

> **Purpose**: Design system prompts and context for current LLMs — role, constraints, structure with XML tags, caching, and mid-conversation operator instructions
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

The system prompt sets the stable contract for a conversation: who the model is acting as, what it may and may not do,
and how to handle uncertainty. Current models follow explicit, well-explained instructions closely, so say what you want
and why, structure mixed content with XML tags, and keep the prompt stable so it can be cached. With 1M-token context
windows on current Claude and GPT-6 models, curation still matters: put long documents first and the question last.

## The Pattern

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = """You extract and validate financial data from invoices for the accounts-payable team.

<role>
- Extract fields defined in the output schema and flag values that look inconsistent.
- Report a confidence for every field so reviewers know what to double-check.
</role>

<constraints>
- Use only values present in the document; leave a field null when it is absent, because reviewers
  treat a guessed value as an error.
- Dates are ISO 8601 (YYYY-MM-DD); amounts are numbers without currency symbols.
- If the document is unreadable or not an invoice, say so in the notes field instead of extracting.
</constraints>"""

def process_document(document: str) -> str:
    resp = client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=16000,
        system=[{"type": "text", "text": SYSTEM_PROMPT,
                 "cache_control": {"type": "ephemeral"}}],         # frozen prefix, cached across calls
        messages=[{"role": "user", "content": f"<document>\n{document}\n</document>\n\nExtract the data."}],
    )
    return "".join(b.text for b in resp.content if b.type == "text")  # never content[0]: thinking may come first
```

## System Prompt Anatomy

| Section | Purpose | Required |
|---------|---------|----------|
| Role / task | What the model does and for whom | Yes |
| Context scope | Domain, data it operates on | Yes |
| Constraints with reasons | Real business or policy limits, each with its "because" | Yes |
| Output contract | Prefer an API-enforced schema; describe only what the schema can't | Yes |
| Uncertainty handling | What to do when information is missing | Recommended |
| Examples | 3–5 varied examples in `<example>` tags when format or judgment is subtle | Optional |

## Context Engineering Principles

```text
1. CURATE, don't dump -- include what the task needs, exclude noise
2. LONG DATA FIRST -- documents near the top, instructions and the question at the end
3. SEPARATE concerns -- system (stable) vs context vs instructions vs query; XML tags per type
4. KEEP THE PREFIX STABLE -- no timestamps, user IDs or per-request flags in the system prompt
5. STATE THE WHY -- an instruction with its reason generalizes better than a bare rule
6. CALM REGISTER -- reserve emphasis (MUST/NEVER) for the rare instruction that is measurably ignored
```

## Mid-Conversation Operator Instructions

Changing top-level `system` mid-session re-processes the whole cached history. On Claude Opus 5/5.5, Opus 4.8, Fable 5/5.1
and Sonnet 5.5, append a system-role message instead — it keeps the cached prefix and is a channel user-supplied text
cannot spoof (unsupported models return 400; fall back to a tagged note in the user turn):

```python
messages.append({"role": "user", "content": user_text})
messages.append({"role": "system", "content": "Escalation mode: hand off billing disputes to a human."})
resp = client.messages.create(model="claude-opus-5-5", max_tokens=16000,
                              system=SYSTEM_BLOCKS, messages=messages)
```

## Common Mistakes

### Wrong

```python
# Vague role, rules without reasons, dynamic data that breaks caching
system = f"You are a helpful assistant. ALWAYS be accurate!!! Today is {datetime.now()}."
```

### Correct

```python
system = """You classify clinical notes into ICD-10 categories for coding review.
Classify only from explicit text, because coders audit every suggestion against the note.
If the note is ambiguous, set confidence below 0.5 and quote the ambiguous span."""
# Put the date, user and session details in the user turn or a system-role message instead.
```

## Provider-Specific Notes

| Provider | System instructions | Notes |
|----------|--------------------|-------|
| Anthropic Claude (current models) | `system=` parameter; `{"role": "system"}` messages mid-conversation on supported models | 1M context; `cache_control` on the system block |
| OpenAI (Responses API) | `instructions=` or a system/developer message in `input` | GPT-6 family: 1.05M context |
| Google Gemini (`google-genai`) | `types.GenerateContentConfig(system_instruction=...)` | Gemini 2.5 limited to existing users; use 3.x/4 |
| Databricks FMAPI / Unity Gateway | Query by model service name (`system.ai.<model>`) or pay-per-token endpoint | Same prompt design rules as the underlying provider |

## Related

- [Output Formatting](../concepts/output-formatting.md)
- [Few-Shot Prompting](../concepts/few-shot-prompting.md)
- [Prompt Template Pattern](../patterns/prompt-template.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.claude.com/docs/en/build-with-claude/context-windows
- https://platform.openai.com/docs/models ; https://developers.openai.com/api/reference/resources/responses/methods/create
- https://github.com/googleapis/python-genai ; https://ai.google.dev/gemini-api/docs/models
