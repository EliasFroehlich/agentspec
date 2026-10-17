# Reusable Prompt Templates

> **Purpose**: Versioned, provider-neutral prompt templates that render to the Claude Messages API or the OpenAI Responses API, plus managed versioning with MLflow Prompt Registry
> **MCP Validated**: 2026-10-17

## When to Use

- Building a shared prompt library for a team
- Keeping prompts version-controlled, testable and comparable across releases
- Running the same prompt on more than one provider
- Not as a place to hide per-request data: dynamic values go in the user turn, not the system text

## Implementation

```python
from dataclasses import dataclass, field
from typing import Any, Optional

from pydantic import BaseModel

@dataclass(frozen=True)
class PromptTemplate:
    """Provider-neutral template. Sampling parameters are deliberately absent: current Claude
    models reject temperature/top_p/top_k, so behavior is controlled by the prompt, the output
    schema and effort/reasoning settings."""
    name: str
    version: str
    system: str                                  # stable text, safe to cache
    user: str                                    # str.format placeholders for per-request data
    output_schema: Optional[type[BaseModel]] = None
    effort: str = "medium"                       # Claude output_config.effort / OpenAI reasoning.effort
    metadata: dict[str, Any] = field(default_factory=dict)

    def render_anthropic(self, model: str, **kw) -> dict[str, Any]:
        """kwargs for client.messages.parse (schema) or client.messages.create (free text)."""
        params: dict[str, Any] = {
            "model": model, "max_tokens": 16000,
            "system": [{"type": "text", "text": self.system, "cache_control": {"type": "ephemeral"}}],
            "messages": [{"role": "user", "content": self.user.format(**kw)}],
        }
        if self.output_schema:
            params["output_format"] = self.output_schema          # use with messages.parse
        else:
            params["output_config"] = {"effort": self.effort}
        return params

    def render_openai(self, model: str, **kw) -> dict[str, Any]:
        """kwargs for client.responses.parse (schema) or client.responses.create (free text)."""
        params: dict[str, Any] = {
            "model": model, "instructions": self.system,
            "input": self.user.format(**kw), "reasoning": {"effort": self.effort},
        }
        if self.output_schema:
            params["text_format"] = self.output_schema           # use with responses.parse
        return params

# --- Pre-built templates ---

class Classification(BaseModel):
    category: str
    confidence: float
    evidence: str                 # quote from the text that supports the label

EXTRACTION = PromptTemplate(
    name="document-extraction", version="2.0.0",
    system=("You extract data for an automated ledger. Use only values present in the document and "
            "leave fields null when absent, because guessed values break reconciliation."),
    user="<document>\n{document}\n</document>\n\nExtract the fields.",
)

CLASSIFICATION = PromptTemplate(
    name="text-classification", version="2.0.0",
    system="You route support messages to exactly one queue.",
    user="<categories>\n{categories}\n</categories>\n\n<text>\n{text}\n</text>",
    output_schema=Classification, effort="low",
)

QA = PromptTemplate(
    name="question-answering", version="2.0.0",
    system=("Answer only from the provided context. If the context does not contain the answer, "
            "say so — users act on these answers."),
    user="<context>\n{context}\n</context>\n\n<question>{question}</question>",
)

# --- Local registry (use MLflow Prompt Registry when available) ---

class PromptRegistry:
    def __init__(self) -> None:
        self._templates: dict[tuple[str, str], PromptTemplate] = {}

    def register(self, t: PromptTemplate) -> None:
        self._templates[(t.name, t.version)] = t

    def get(self, name: str, version: str = "latest") -> PromptTemplate:
        versions = sorted((v for n, v in self._templates if n == name),
                          key=lambda v: tuple(int(x) for x in v.split(".")))
        if not versions:
            raise KeyError(name)
        return self._templates[(name, versions[-1] if version == "latest" else version)]

registry = PromptRegistry()
for t in (EXTRACTION, CLASSIFICATION, QA):
    registry.register(t)
```

## Managed Versioning: MLflow Prompt Registry

```python
import mlflow

prompt = mlflow.genai.register_prompt(          # template variables use double braces
    name="support-qa",
    template="Answer only from the context.\n<context>{{context}}</context>\n<question>{{question}}</question>",
    commit_message="v2: cite context only",
)
mlflow.genai.set_prompt_alias(name="support-qa", alias="production", version=prompt.version)

loaded = mlflow.genai.load_prompt("prompts:/support-qa@production")
user_text = loaded.format(context=ctx, question=q)
```

Databricks exposes the same API as a managed service (MLflow Prompt Registry); see its docs for naming and permissions.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `version` | semantic version | Bump on any wording change; evaluate before promoting |
| `effort` | `medium` | Per template; `low` for routing/classification |
| `output_schema` | `None` | Pydantic model enforced by the API when set |
| Sampling | none | Not sent; OpenAI `temperature` can be added per call where the model supports it |

## Example Usage

```python
import anthropic

t = registry.get("text-classification")
resp = anthropic.Anthropic().messages.parse(
    **t.render_anthropic("claude-haiku-5-5", categories="billing\ntechnical\nother", text=msg))
print(resp.parsed_output)

from openai import OpenAI
resp = OpenAI().responses.parse(**t.render_openai("gpt-6-luna", categories="billing\ntechnical\nother", text=msg))
print(resp.output_parsed)
```

## See Also

- [System Prompts](../concepts/system-prompts.md)
- [Few-Shot Prompting](../concepts/few-shot-prompting.md)
- [Document Extraction](../patterns/document-extraction.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.openai.com/docs/guides/structured-outputs
- https://developers.openai.com/api/docs/guides/code-generation
- https://mlflow.org/docs/latest/genai/prompt-registry/
- https://docs.databricks.com/aws/en/mlflow3/genai/prompt-version-mgmt/prompt-registry/
