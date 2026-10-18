# Guardrails

> **Purpose**: Defense-in-depth safety layers for LLM applications — prompt injection defense, content safety, and the OWASP Top 10 for LLM Applications (2025)
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Guardrails are programmable layers that inspect inputs, retrieved context, tool calls, and outputs of an LLM system and
enforce policy. No single layer is sufficient: combine input validation, retrieval filtering, tool-use restrictions,
output checks, and monitoring. The OWASP Top 10 for LLM Applications 2025 is the common threat checklist; for RAG systems,
LLM08 (Vector and Embedding Weaknesses) and LLM07 (System Prompt Leakage) matter as much as LLM01 (Prompt Injection).

## The Pattern

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class RailAction(Enum):
    ALLOW = "allow"
    BLOCK = "block"
    MODIFY = "modify"
    ESCALATE = "escalate"

@dataclass
class RailResult:
    action: RailAction
    message: Optional[str] = None
    modified_content: Optional[str] = None
    violated_policy: Optional[str] = None

class GuardRail(ABC):
    @abstractmethod
    def check(self, content: str, context: dict) -> RailResult: ...

class PromptInjectionRail(GuardRail):
    """OWASP LLM01: pattern screen first, then a safety classifier."""
    def __init__(self, classifier_model: str = "meta-llama/Llama-Guard-4-12B"):
        self.classifier = classifier_model

    def check(self, content: str, context: dict) -> RailResult:
        if self._pattern_check(content) or self._classifier_check(content):
            return RailResult(RailAction.BLOCK, violated_policy="prompt_injection")
        return RailResult(RailAction.ALLOW)

    def _pattern_check(self, content: str) -> bool:
        patterns = ["ignore previous", "you are now", "disregard", "forget your instructions"]
        return any(p in content.lower() for p in patterns)

class OutputGroundingRail(GuardRail):
    """OWASP LLM09 Misinformation: answer must be supported by retrieved context."""
    def check(self, content: str, context: dict) -> RailResult:
        chunks = context.get("retrieved_chunks")
        if chunks and not self._is_grounded(content, chunks):
            return RailResult(RailAction.MODIFY, violated_policy="grounding",
                              modified_content="I don't have enough information to answer that.")
        return RailResult(RailAction.ALLOW)
```

## Defense-in-Depth Pipeline

```text
User input
  └─ [Input rails]      injection screen + classifier, PII redaction, topic scope
     └─ [Retrieval rails]  ACL-filtered retrieval, relevance threshold, content safety on chunks
        └─ [Tool rails]       argument validation, allowlists, human approval for destructive actions
           └─ [LLM]              operator instructions only in system / system-role messages
              └─ [Output rails]     grounding check, toxicity, PII scan, schema validation
                 └─ Response + audit log
```

## OWASP Top 10 for LLM Applications (2025)

| Risk | Guardrail strategy |
|------|--------------------|
| LLM01 Prompt Injection | Separate operator instructions from user/retrieved text; classifier + pattern screens; treat tool output as data |
| LLM02 Sensitive Information Disclosure | PII redaction before storage/prompting (e.g. Databricks `ai_mask`), output scanning, least-privilege data access |
| LLM03 Supply Chain | Model and package provenance, pinned versions, signed artifacts |
| LLM04 Data and Model Poisoning | Validate training/RAG sources, track lineage, review ingestion |
| LLM05 Improper Output Handling | Never execute or render model output unvalidated; parameterized SQL, escaping |
| LLM06 Excessive Agency | Tool allowlists, confirmation gates, scoped credentials |
| LLM07 System Prompt Leakage | Keep secrets and authorization logic out of prompts; enforce them in code |
| LLM08 Vector and Embedding Weaknesses | Per-tenant filters or indexes, ACL-aware retrieval, provenance on chunks |
| LLM09 Misinformation | Grounding checks, citations, LLM-as-judge evaluation |
| LLM10 Unbounded Consumption | Rate limits, token budgets, `max_tokens`, cost alerts |

## Guardrail Tools

| Tool | Type | Notes |
|------|------|-------|
| Llama Guard 4 (`meta-llama/Llama-Guard-4-12B`) | Open-weight safety classifier | Multimodal; MLCommons hazards S1–S13 plus S14 Code Interpreter Abuse |
| NeMo Guardrails | NVIDIA open-source framework | Configurable rails |
| Guardrails AI | Open-source Python library | Validator-based output checks |
| Unity Gateway (Databricks) | Managed gateway | PII masking, rate limits, usage tracking for models, MCP servers and skills |
| Claude `stop_reason: "refusal"` | Provider safeguard | Handle before reading `content`; see [Claude API Surface](../concepts/anthropic-api-surface.md) |

## Quick Reference

| Rail | Stage | Technique |
|------|-------|-----------|
| Prompt injection | Input | Pattern screen + Llama Guard 4 |
| PII redaction | Input / storage | Regex + NER, or `ai_mask` in Databricks SQL |
| Topic control | Input | Classifier or NeMo Guardrails |
| Retrieval scoping | Retrieval | Metadata filters per tenant/user |
| Tool validation | Pre-execution | Pydantic + allowlist + approval |
| Grounding | Output | LLM-as-judge or `RetrievalGroundedness` scorer (MLflow) |
| Format validation | Output | Provider structured outputs + Pydantic |

## Common Mistakes

### Wrong

```python
# Guardrails only on output -- prompt injection already reached the model and its tools
response = llm.generate(user_input)
filtered = toxicity_filter(response)
```

### Correct

```python
pipeline = GuardrailPipeline(rails=[
    PromptInjectionRail(classifier_model="meta-llama/Llama-Guard-4-12B"),   # pre-LLM
    PIIRedactor(),
    TopicRail(["billing", "support"]),
    ToolValidationRail(allowed_tools=["search", "lookup"]),
    OutputGroundingRail(),                                                 # post-LLM
    FormatValidator(schema),
])
result = pipeline.run(user_input)
```

## Related

- [Tool Calling](../concepts/tool-calling.md)
- [Chatbot Architecture Pattern](../patterns/chatbot-architecture.md)
- [Evaluation Framework](../patterns/evaluation-framework.md)

## Sources

- https://genai.owasp.org/llm-top-10/
- https://huggingface.co/meta-llama/Llama-Guard-4-12B
- https://docs.databricks.com/aws/en/unity-gateway/
- https://docs.databricks.com/aws/en/large-language-models/ai-functions
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
