# GenAI Architecture Knowledge Base

> **Purpose**: Architecture patterns for GenAI systems -- multi-agent orchestration, agentic workflows, RAG, chatbots, MCP, agent SDKs, the current Claude API surface, and the Databricks agent platform
> **MCP Validated**: 2026-10-17

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/multi-agent-systems.md](concepts/multi-agent-systems.md) | Orchestration topologies; LangGraph, Claude Agent SDK, OpenAI Agents SDK, ADK, Pydantic AI, Microsoft Agent Framework |
| [concepts/rag-architecture.md](concepts/rag-architecture.md) | RAG variants, chunking, context engineering, managed building blocks |
| [concepts/state-machines.md](concepts/state-machines.md) | LangGraph StateGraph and Functional API for deterministic flows |
| [concepts/tool-calling.md](concepts/tool-calling.md) | Provider tool schemas, MCP 2026-07-28, MCP Python SDK 2.x, safe arguments |
| [concepts/guardrails.md](concepts/guardrails.md) | Defense in depth, Llama Guard 4, OWASP Top 10 for LLM Applications 2025 |
| [concepts/anthropic-api-surface.md](concepts/anthropic-api-surface.md) | Current Claude models, structured outputs, adaptive thinking/effort, rejected parameters, caching, agent surfaces |
| [concepts/databricks-agent-platform.md](concepts/databricks-agent-platform.md) | Genie One/Agents/Code, Agent Bricks, Unity Gateway, FMAPI, legacy offerings |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/agentic-workflow.md](patterns/agentic-workflow.md) | Plan-and-execute with structured plans and per-role model/effort |
| [patterns/chatbot-architecture.md](patterns/chatbot-architecture.md) | Production chatbot: routing, guardrails, RAG, MCP connector, context management |
| [patterns/rag-pipeline.md](patterns/rag-pipeline.md) | End-to-end RAG with hybrid search and reranking; Databricks AI Search variant |
| [patterns/evaluation-framework.md](patterns/evaluation-framework.md) | MLflow 3 GenAI evaluation, RAGAS, DeepEval, CI quality gate |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/genai-patterns.yaml](specs/genai-patterns.yaml) | Patterns, frameworks, models (as of 2026-10-17), Databricks layer, decision matrix |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Multi-Agent Systems** | Supervisor, handoffs, plan-and-execute; start with the simplest tier that works |
| **RAG Architecture** | Hybrid search, reranking, context-enriched chunks, citations |
| **State Machines** | Graph-based orchestration with LangGraph |
| **Tool Calling** | Strict schemas, MCP, tool search for large catalogs |
| **Guardrails** | Layered rails mapped to OWASP LLM 2025 |
| **Claude API Surface** | Structured outputs instead of tool-forcing; effort instead of `budget_tokens`; no `temperature`/prefill on current models |
| **Databricks Agent Platform** | Genie Agents for NL-to-SQL, Agent Bricks for code agents, Unity Gateway for models/MCP |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/tool-calling.md, concepts/anthropic-api-surface.md, concepts/rag-architecture.md |
| **Intermediate** | concepts/multi-agent-systems.md, concepts/databricks-agent-platform.md, patterns/rag-pipeline.md |
| **Advanced** | patterns/agentic-workflow.md, patterns/evaluation-framework.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| genai-architect | patterns/agentic-workflow.md, patterns/chatbot-architecture.md, concepts/databricks-agent-platform.md | Design multi-agent and chatbot systems |
| genai-architect | patterns/rag-pipeline.md, patterns/evaluation-framework.md | Build RAG pipelines with evaluation |
| llm-specialist, ai-prompt-specialist | concepts/anthropic-api-surface.md | Current request shapes for Claude |
