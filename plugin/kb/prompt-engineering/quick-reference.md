# Prompt Engineering Quick Reference

> Fast lookup tables. For code examples, see linked files.
> **MCP Validated:** 2026-10-17 — model IDs as of 2026-10-17.

## Context Engineering

```text
[System]   Role + scope + constraints with reasons (stable → cacheable)
[Context]  Long documents and data first, each in XML tags (<document>, <source>)
[Examples] 3–5 varied examples in <example> tags, only where format or judgment is subtle
[Task]     Outcome to achieve and how to verify it — not a prescribed step list
[Query]    The per-request input, last
Output     Enforced by an API schema whenever it is structured
```

## Technique Selection

| Technique | Best for | Notes |
|-----------|----------|-------|
| Zero-shot + schema | Clear extraction/classification | Default starting point |
| Few-shot | Tone, format, decision boundaries | Vary examples; the model mirrors them closely |
| Built-in thinking (Claude effort / OpenAI reasoning effort) | Analysis, math, code | Prefer over hand-written chain-of-thought |
| Manual CoT with `<answer>` tags | Models or routes without thinking | Fallback only |
| Agent loop (ReAct-style) | Tool use, research | Use a framework or SDK tool runner |
| Self-consistency | High-stakes short answers | Independent calls; multiplies cost |
| Multi-pass + audit | Dense documents | Measure single-pass accuracy first |

## Reasoning and Sampling Controls

| Provider / model | Control | Not supported |
|------------------|---------|---------------|
| Claude Opus 5.5, Fable 5.1 | Thinking always on; `output_config.effort` (Opus 5.5 default `medium`) | `temperature`/`top_p`/`top_k`, prefill, `budget_tokens`, disabling thinking |
| Claude Sonnet 5.5, Haiku 5.5 | Thinking on by default; `effort`; Sonnet 5.5 off = `between_tools` | Non-default sampling values, prefill, `budget_tokens` |
| OpenAI GPT-6 family | `reasoning={"effort": ...}` (Responses API) | — (check per model) |
| Gemini | `GenerateContentConfig` (`temperature`, `top_p`, …) | — |

## Structured Output Methods

| Method | Provider | Call |
|--------|----------|------|
| Native structured outputs | Anthropic (GA) | `client.messages.parse(..., output_format=Model)` → `.parsed_output` |
| Structured Outputs | OpenAI | `client.responses.parse(..., text_format=Model)` → `.output_parsed` |
| Response schema | Gemini | `GenerateContentConfig(response_mime_type="application/json", response_schema=Model)` → `.parsed` |
| Strict tools | Anthropic, OpenAI | `strict: true` on tool definitions |
| Instructor | Many | `instructor.from_provider("anthropic/...")` |
| `ai_extract` / `ai_classify` | Databricks SQL | Per-row batch extraction |
| JSON mode, prefill, forced tool | — | Superseded; prefill and forced tool 400 on current Claude models |

## Cost and Latency Levers

| Lever | How |
|-------|-----|
| Prompt caching | Frozen system/tools prefix + `cache_control`; reads 0.1x (0.05x Opus/Sonnet 5.5) |
| Effort | `low` for routing/classification, `high`+ only where quality measurably improves |
| Batches | Message Batches API: 50% off for non-interactive work |
| Mid-conversation changes | Append a `{"role": "system"}` message instead of editing `system` (supported models) |
| Token counting | `client.messages.count_tokens` (Claude); `client.responses.input_tokens.count` (OpenAI) |

## Decision Matrix

| Use Case | Choose |
|----------|--------|
| Extract fields from an invoice | `patterns/document-extraction.md` |
| Better reasoning | `concepts/chain-of-thought.md` |
| Reliable JSON | `concepts/output-formatting.md` |
| Teach format by example | `concepts/few-shot-prompting.md` |
| Verify extracted values | `patterns/validation-prompts.md` |
| High-accuracy extraction | `patterns/multi-pass-extraction.md` |
| Reusable, versioned prompts | `patterns/prompt-template.md` |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| "Return ONLY valid JSON" + regex repair | API-enforced schema |
| Prescribe "step 1… step 4" or ask for `<thinking>` in the output | General instructions + effort; reasoning stays in thinking blocks |
| Set `temperature=0` on Claude Opus/Sonnet/Haiku 5.5 | Omit it; constrain with schema and instructions |
| Read `response.content[0].text` | Select blocks by `type` |
| Put timestamps or user IDs in the system prompt | Put them in the user turn |
| Use `dict` fields in output schemas | Lists of `{name, value}` objects |
| Shout rules in caps | State the rule once, with its reason |

## Related Documentation

| Topic | Path |
|-------|------|
| Claude API surface | `../genai/concepts/anthropic-api-surface.md` |
| System Prompts | `concepts/system-prompts.md` |
| Full Index | `index.md` |

## Sources

- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices · https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.openai.com/docs/guides/structured-outputs · https://github.com/googleapis/python-genai · https://docs.databricks.com/aws/en/large-language-models/ai-functions
