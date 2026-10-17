# LLMOps Patterns

> **Purpose**: Prompt versioning, per-provider token counting and cost tracking, input guardrails, evaluation, and tracing (MLflow 3, Langfuse v5, OpenTelemetry GenAI conventions)
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

LLMOps applies MLOps discipline to LLM applications: version prompts like code, count tokens with the provider's own
counter, track cost per request and per completed task, guard inputs and outputs, evaluate on fixed datasets, and trace
production traffic. On Databricks the native stack is MLflow 3 (Tracing, `mlflow.genai.evaluate`, Prompt Registry,
production monitoring) plus Unity Gateway usage tracking; outside it, Langfuse (Python SDK v5, OpenTelemetry-based),
LangSmith, Arize Phoenix or Braintrust fill the same roles. OpenTelemetry GenAI semantic conventions (now in the
`semantic-conventions-genai` repository) are the common denominator between them.

## The Concept

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone

import anthropic

client = anthropic.Anthropic()

# USD per million tokens (input, output) as of 2026-10-17 -- re-check the provider pricing pages
PRICES = {
    "claude-fable-5-1": (10.00, 50.00), "claude-opus-5-5": (4.00, 20.00),
    "claude-sonnet-5-5": (2.00, 10.00), "claude-haiku-5-5": (0.10, 0.50),   # Haiku 5.5: prompt <= 100K
    "gpt-6-astra": (10.00, 50.00), "gpt-6.1-sol": (2.00, 10.00), "gpt-6-luna": (0.10, 0.50),
}

@dataclass(frozen=True)
class VersionedPrompt:
    name: str
    version: str
    system: str
    model: str = "claude-sonnet-5-5"
    max_input_tokens: int = 50_000
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())

    def count_input_tokens(self, user_text: str) -> int:
        """Provider-side count; tiktoken undercounts Claude tokens and must not be used."""
        return client.messages.count_tokens(
            model=self.model, system=self.system,
            messages=[{"role": "user", "content": user_text}],
        ).input_tokens

    def cost_usd(self, usage) -> float:
        """usage: response.usage (thinking tokens are billed as output; cache reads are cheaper)."""
        price_in, price_out = PRICES[self.model]
        return (usage.input_tokens * price_in + usage.output_tokens * price_out) / 1_000_000

    def validate_input(self, user_text: str) -> tuple[bool, str]:
        if (n := self.count_input_tokens(user_text)) > self.max_input_tokens:
            return False, f"input too long: {n} tokens"
        if any(p in user_text.lower() for p in ("ignore previous", "you are now", "disregard")):
            return False, "possible prompt injection"
        return True, "ok"
```

OpenAI equivalent for counting: `OpenAI().responses.input_tokens.count(model=..., input=..., tools=...)`.

## Tracing

```python
# MLflow 3 (default on Databricks): one line per library, plus @mlflow.trace for your own steps
import mlflow

mlflow.anthropic.autolog()          # also mlflow.openai / mlflow.langchain autolog

@mlflow.trace
def rag_pipeline(query: str) -> str:
    chunks = retrieve(query)        # decorate retrieve with @mlflow.trace(span_type="RETRIEVER")
    return generate(query, chunks)
```

```python
# Langfuse Python SDK v5 (OpenTelemetry-based): get_client(), @observe, propagate_attributes
from langfuse import get_client, observe, propagate_attributes

langfuse = get_client()

@observe()
def answer(query: str, user_id: str) -> str:
    with propagate_attributes(user_id=user_id, metadata={"prompt_version": "2.1.0"}):
        chunks = retrieve(query)
        result = generate(query, chunks)
    langfuse.score_current_trace(name="relevancy", value=score_relevancy(query, result))
    return result

langfuse.flush()                    # short-lived scripts only
```

`langfuse.decorators` / `langfuse_context` belong to the v2 SDK; v4 replaced `update_current_trace()` with
`propagate_attributes()`.

## Quick Reference

| Area | Databricks-native | Elsewhere |
|------|-------------------|-----------|
| Prompt versioning | MLflow Prompt Registry (`register_prompt`, aliases) | Git, Langfuse prompts, LangSmith |
| Token counting | Provider counters (`messages.count_tokens`, `responses.input_tokens.count`) | Same |
| Cost tracking | Unity Gateway usage, `system.billing` tables | Langfuse, provider usage APIs, LiteLLM |
| Guardrails | Unity Gateway (PII masking, rate limits), `ai_mask` | Llama Guard 4, NeMo Guardrails, Guardrails AI |
| Evaluation | `mlflow.genai.evaluate` + scorers/judges | RAGAS, DeepEval, Braintrust |
| Tracing | MLflow Tracing (OTel-compatible), traces in Unity Catalog | Langfuse, LangSmith, Arize Phoenix |
| Production monitoring | MLflow scorers on sampled traces | Langfuse evaluators, Datadog LLM Observability |

| RAG metric | Measures | Typical target |
|------------|----------|----------------|
| Groundedness / faithfulness | Answer supported by context | > 0.90 |
| Relevance | Answer addresses the question | > 0.85 |
| Context precision / relevance | Retrieved chunks are relevant | > 0.80 |
| Context recall / sufficiency | Needed facts were retrieved | > 0.80 |

## Common Mistakes

### Wrong

```python
import tiktoken
tokens = len(tiktoken.encoding_for_model("gpt-4o").encode(prompt))   # wrong tokenizer for Claude
cost = tokens * 15 / 1e6                                              # stale price table
```

### Correct

```python
p = VersionedPrompt(name="rag-answer", version="2.1.0", system=SYSTEM)
resp = client.messages.create(model=p.model, max_tokens=16000, system=p.system,
                              messages=[{"role": "user", "content": question}])
log_metrics(prompt=f"{p.name}@{p.version}", cost_usd=p.cost_usd(resp.usage),
            cache_read_tokens=resp.usage.cache_read_input_tokens)
```

## Related

- [rag-pipelines](../concepts/rag-pipelines.md)
- [Evaluation Framework](../../genai/patterns/evaluation-framework.md)
- [Prompt Template Pattern](../../prompt-engineering/patterns/prompt-template.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/token-counting
- https://platform.claude.com/docs/en/about-claude/pricing ; https://platform.openai.com/docs/models
- https://mlflow.org/docs/latest/genai/tracing/ ; https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/
- https://langfuse.com/docs/observability/sdk/python/overview
- https://langfuse.com/docs/observability/sdk/upgrade-path/python-v3-to-v4
- https://opentelemetry.io/docs/specs/semconv/gen-ai/ (moved to github.com/open-telemetry/semantic-conventions-genai)
