# GenAI Architecture Quick Reference

> Fast lookup tables. For code examples, see linked files.
> **MCP Validated:** 2026-10-17 — versions and prices are "as of 2026-10-17"; re-check before quoting.

## Agent Frameworks (PyPI versions)

| Framework | Maintainer | Version | Strength |
|-----------|-----------|---------|----------|
| LangGraph | LangChain | 1.2.16 | Graph state machines, durable execution, interrupts |
| LangChain `create_agent` | LangChain | 1.4.6 | Agent loop + middleware; legacy chains in `langchain-classic` |
| Claude Agent SDK | Anthropic | 0.2.168 | Claude Code harness as a library: built-in tools, subagents, hooks, MCP |
| OpenAI Agents SDK | OpenAI | 0.24.0 (+ JS `@openai/agents`) | Handoffs, guardrails, tracing, sessions |
| Google ADK | Google | 2.12.0 | Python, TypeScript, Go, Java |
| Pydantic AI | Pydantic | 2.57.0 | Typed agents, durable execution |
| Microsoft Agent Framework | Microsoft | 1.22.0 (`agent-framework`) | Successor to AutoGen (archived) |
| CrewAI | CrewAI Inc | 1.15.28 | Role-based crews, `mcps` field |
| Databricks Agent Bricks | Databricks | platform | Hosts any framework: Agent Runtime, `DurableAgentServer`, Sandbox, memory |

## Building a Claude Agent

| Approach | Use when |
|----------|----------|
| Manual loop (`anthropic`) | You want full control of the loop |
| Tool Runner (`client.beta.messages.tool_runner`) | Custom tools, SDK runs the loop |
| Claude Agent SDK (`query()`, `ClaudeSDKClient`) | Coding/filesystem agent with built-in tools on your infra |
| Managed Agents (beta) | Anthropic hosts the loop and a per-session sandbox |

## Model Landscape (as of 2026-10-17)

| Model | ID | Context | $/MTok in/out |
|-------|----|---------|---------------|
| Claude Fable 5.1 | `claude-fable-5-1` | 1M | 10 / 50 |
| Claude Opus 5.5 | `claude-opus-5-5` | 1M | 4 / 20 |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | 1M | 2 / 10 |
| Claude Haiku 5.5 | `claude-haiku-5-5` | 1M | 0.10 / 0.50 (prompt ≤ 100K) |
| GPT-6 Astra / 6.1 Sol / 6 Luna | `gpt-6-astra` / `gpt-6.1-sol` / `gpt-6-luna` | 1.05M | 10/50 · 2/10 · 0.1/0.5 |
| Gemini 4 Argon / 3.8 Flash | `gemini-4-argon` / `gemini-3.8-flash` | — | see Google pricing |
| Databricks FMAPI | `system.ai.<model>` (e.g. `system.ai.claude-opus-5-5`) | provider | Unity Gateway billing |

Retired or deprecated: Claude Haiku 3.5 / Sonnet 3.7 (retired), Sonnet 4 / Opus 4 / Opus 4.1 (retired), Sonnet 4.5
(retires 2026-11-30); GPT-4o / 4o-mini (removal 2027-01-14); `gpt-4.5-preview` (removed); Gemini 2.0 Flash (shut down).

## Tool Calling and MCP

| Item | Current state |
|------|---------------|
| MCP spec | Revision 2026-07-28: stateless, `server/discover`, transports stdio + Streamable HTTP; HTTP+SSE, Roots, Sampling, Logging deprecated |
| MCP Python SDK | `mcp` 2.3.0: `from mcp.server import MCPServer`, `from mcp import Client` |
| Claude MCP connector | Beta `mcp-client-2025-11-20`: `mcp_servers` + `mcp_toolset` tool, remote HTTP only |
| Forced tool use (Claude) | `tool_choice` `any`/`tool` → 400 on Fable 5.1 / Opus 5.5 / Sonnet 5.5 |
| Large tool catalogs | Tool search + `defer_loading` (Claude and OpenAI) |
| Computer use (Claude 5.5) | `computer_toolset_20260801` |

## Databricks Layer

| Need | Use |
|------|-----|
| NL → SQL over governed data | Genie Agents (formerly Genie Spaces) + Conversation API |
| Business context for external agents | Genie One MCP server |
| Retrieval | AI Search (formerly Vector Search) |
| Eval + tracing | MLflow 3 (`mlflow.genai.evaluate`, scorers, Tracing) |
| Legacy | Knowledge Assistant, Supervisor Agent → Genie Agents / Agent Bricks CLI |

## Evaluation and Observability

| Tool | Type | Note |
|------|------|------|
| MLflow 3 GenAI | OSS + Databricks managed | Default on Databricks; OTel-compatible tracing |
| RAGAS 0.4 | OSS | `EvaluationDataset` + class metrics |
| DeepEval 4.3 | OSS | pytest-style; `AnthropicModel` for Claude judges |
| Langfuse 5.x | OSS / cloud | OTel-based SDK, `get_client()`, `@observe` |
| LangSmith, Arize Phoenix, Braintrust | Commercial / OSS | Pick by existing stack |

## Common Pitfalls

| Don't | Do |
|-------|-----|
| Force a tool call to get JSON | Structured outputs (`messages.parse`, `output_config.format`) |
| Send `temperature`/prefill/`budget_tokens` to current Claude models | Prompt + `effort` + adaptive thinking |
| Read `response.content[0].text` | Select blocks by `type` |
| Re-implement text-to-SQL on Databricks | Curate a Genie Agent and call it |
| Cascade models before measuring | Try one model with per-route effort first |
| Skip evaluation | MLflow/RAGAS + LLM-as-judge from day one |

## Related Documentation

| Topic | Path |
|-------|------|
| Claude API surface | `concepts/anthropic-api-surface.md` |
| Databricks agent platform | `concepts/databricks-agent-platform.md` |
| Full Index | `index.md` |

## Sources

- https://platform.claude.com/docs/en/about-claude/pricing · https://platform.claude.com/docs/en/about-claude/model-deprecations · https://modelcontextprotocol.io/specification/2026-07-28/changelog
- https://platform.openai.com/docs/models · https://platform.openai.com/docs/deprecations · https://ai.google.dev/gemini-api/docs/models · https://ai.google.dev/gemini-api/docs/deprecations
- https://docs.databricks.com/aws/en/agents/concepts/ · https://docs.databricks.com/aws/en/genie/ · https://pypi.org/ (package versions)
