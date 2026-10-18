# Prompt Engineering Knowledge Base

> **Purpose**: Techniques and patterns for effective LLM prompting -- context engineering, provider-native structured outputs, model thinking and effort, few-shot examples, extraction and validation
> **MCP Validated:** 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/chain-of-thought.md](concepts/chain-of-thought.md) | Built-in thinking (Claude effort, OpenAI reasoning effort), manual CoT as fallback, self-consistency |
| [concepts/structured-extraction.md](concepts/structured-extraction.md) | Typed extraction with native structured outputs, Instructor, and Databricks `ai_extract` |
| [concepts/few-shot-prompting.md](concepts/few-shot-prompting.md) | Varied examples in `<example>` tags, dynamic selection |
| [concepts/system-prompts.md](concepts/system-prompts.md) | System prompt design, XML structure, caching, mid-conversation system messages |
| [concepts/output-formatting.md](concepts/output-formatting.md) | Claude `messages.parse`, OpenAI `responses.parse`, Gemini `response_schema`, Instructor |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/document-extraction.md](patterns/document-extraction.md) | Claude native PDF + structured outputs, OpenAI variant, Databricks SQL batch |
| [patterns/validation-prompts.md](patterns/validation-prompts.md) | Independent audit pass with evidence quotes; consistency across runs |
| [patterns/multi-pass-extraction.md](patterns/multi-pass-extraction.md) | Schema-enforced extract → review → verify passes with early stop |
| [patterns/prompt-template.md](patterns/prompt-template.md) | Provider-neutral templates rendering to Claude/OpenAI; MLflow Prompt Registry |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/prompt-formats.yaml](specs/prompt-formats.yaml) | Prompt anatomy, reasoning controls per provider, model recommendations (as of 2026-10-17) |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Context Engineering** | Curated context, long data first, XML tags, stable cacheable prefix |
| **Model Thinking** | Effort controls reasoning depth on current models; prescriptive CoT is a fallback |
| **Structured Extraction** | Pydantic schemas enforced by the provider API, plus business-rule validation |
| **Few-Shot Prompting** | 3–5 varied examples to pin format and judgment |
| **System Prompts** | Reasoned constraints instead of shouted rules; operator updates via system-role messages |
| **Output Formatting** | Native structured outputs replace JSON mode, prefill and tool-forcing |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/system-prompts.md, concepts/output-formatting.md |
| **Intermediate** | concepts/chain-of-thought.md, concepts/few-shot-prompting.md |
| **Advanced** | concepts/structured-extraction.md, patterns/multi-pass-extraction.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| ai-prompt-specialist | patterns/document-extraction.md, patterns/prompt-template.md | Design extraction and structured output prompts |
| llm-specialist | concepts/output-formatting.md, concepts/chain-of-thought.md | Optimize prompts and request settings |
