# Chatbot Architecture Pattern

> **Purpose**: Production chatbot with intent routing, guardrails, RAG, remote MCP tools, and provider-side context management
> **MCP Validated**: 2026-10-17

## When to Use

- Customer-facing conversational AI requiring reliability and safety
- Support bots with intent routing and escalation flows
- Domain assistants that need grounded answers (RAG) and a few transactional tools
- Any chatbot requiring auditable conversation flows

## Architecture

```text
User message
  └─ Input rails (injection, PII, topic)
     └─ Intent classifier (small model, structured output)
        ├─ FAQ / RAG        → retrieval + grounded answer with citations
        ├─ Transactional    → tool calls (local tools or remote MCP via the MCP connector)
        └─ Escalation       → human handoff
     └─ Output rails (grounding, toxicity, format)
        └─ Response + citations + trace
```

## Implementation

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Literal

import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()

class Intent(str, Enum):
    FAQ = "faq"
    TRANSACTIONAL = "transactional"
    ESCALATION = "escalation"
    CHITCHAT = "chitchat"

class IntentResult(BaseModel):
    intent: Literal["faq", "transactional", "escalation", "chitchat"]

@dataclass
class ChatbotConfig:
    model: str = "claude-sonnet-5-5"
    router_model: str = "claude-haiku-5-5"
    system_prompt: str = "You are the support assistant for ACME. Answer only from provided sources."
    escalation_threshold: int = 3          # failed turns before human handoff

@dataclass
class ProductionChatbot:
    config: ChatbotConfig
    rag: object | None = None
    guardrails: object | None = None
    messages: list[dict] = field(default_factory=list)   # append-only history
    failed_turns: int = 0

    def classify(self, text: str) -> Intent:
        resp = client.messages.parse(
            model=self.config.router_model, max_tokens=2048,   # room for thinking
            messages=[{"role": "user", "content": f"Classify the support intent:\n{text}"}],
            output_format=IntentResult,
        )
        return Intent(resp.parsed_output.intent)

    def respond(self, user_message: str) -> str:
        if self.guardrails and self.guardrails.check_input(user_message).action == "block":
            return "I can't help with that request."
        intent = self.classify(user_message)
        if intent is Intent.ESCALATION or self.failed_turns >= self.config.escalation_threshold:
            self.failed_turns = 0
            return "I'm connecting you with a human agent."
        content = user_message
        if intent is Intent.FAQ and self.rag:
            chunks = self.rag.retrieve(user_message)
            if not chunks:
                self.failed_turns += 1
                return "I don't have information on that topic yet."
            sources = "\n".join(f"<source id='{i}'>{c.text}</source>" for i, c in enumerate(chunks))
            content = f"<sources>\n{sources}\n</sources>\n\n{user_message}"
        self.messages.append({"role": "user", "content": content})
        resp = client.messages.create(
            model=self.config.model, max_tokens=16000,
            system=[{"type": "text", "text": self.config.system_prompt,
                     "cache_control": {"type": "ephemeral"}}],          # frozen, cached prefix
            messages=self.messages,
        )
        self.messages.append({"role": "assistant", "content": resp.content})  # keep all blocks
        answer = "".join(b.text for b in resp.content if b.type == "text")
        if self.guardrails:
            answer = self.guardrails.check_output(answer).modified_content or answer
        self.failed_turns = 0
        return answer
```

## Context Management (instead of trimming history by hand)

| Need | Claude API option | Notes |
|------|-------------------|-------|
| Long conversations near the context limit | Compaction, beta `compact-2026-01-12`, `context_management={"edits": [{"type": "compact_20260112"}]}` on `client.beta.messages.create` | Append `response.content` (compaction blocks) every turn |
| Old tool results bloating context | Context editing, beta `context-management-2025-06-27`, `{"type": "clear_tool_uses_20250919"}` | Clears, does not summarize |
| Mode switch or injected state mid-session | Append `{"role": "system", "content": "..."}` to `messages` | Keeps the cached prefix; must follow a user turn |
| Stable instructions | Frozen top-level `system` with `cache_control` | Never interpolate timestamps or user IDs into it |

Keep history append-only: on current models (Fable 5.1, Opus 5.5, Sonnet 5.5, Haiku 5.5) editing or deleting earlier turns
invalidates preserved thinking blocks and the prompt cache.

## Remote MCP Tools (MCP connector)

```python
resp = client.beta.messages.create(
    model="claude-sonnet-5-5", max_tokens=16000,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://mcp.example.com/mcp",
                  "name": "orders", "authorization_token": ORDERS_TOKEN}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "orders"}],   # both halves are required
    messages=[{"role": "user", "content": "Where is order 1234?"}],
)
```

The connector reaches only publicly exposed HTTP servers (Streamable HTTP or SSE); local stdio servers need a client-side
MCP client instead. On Databricks, Genie One and Databricks SQL are available as Databricks-provided MCP servers.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `model` | `claude-sonnet-5-5` | Answer model |
| `router_model` | `claude-haiku-5-5` | Intent classification with structured output |
| `escalation_threshold` | `3` | Failed turns before human handoff |
| `rag_similarity_threshold` | `0.7` | Minimum retrieval relevance (tune per embedding model) |
| `session_ttl_minutes` | `30` | Session expiration in your session store |

## See Also

- [State Machines](../concepts/state-machines.md)
- [Guardrails](../concepts/guardrails.md)
- [RAG Pipeline](../patterns/rag-pipeline.md)
- [Claude API Surface](../concepts/anthropic-api-surface.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.claude.com/docs/en/build-with-claude/compaction
- https://platform.claude.com/docs/en/build-with-claude/context-editing
- https://platform.claude.com/docs/en/agents-and-tools/mcp-connector
- https://docs.databricks.com/aws/en/agents/mcp-tools/
