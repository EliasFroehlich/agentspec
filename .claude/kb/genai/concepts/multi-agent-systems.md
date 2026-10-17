# Multi-Agent Systems

> **Purpose**: Orchestration patterns and current frameworks for coordinating specialized LLM agents
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Multi-agent systems split a task across specialized agents, each with its own instructions and tools, coordinated by a
graph, a supervisor, or handoffs. Start with the simplest tier that works: a single call or a code-controlled workflow
handles most tasks; reach for agents only when the work is open-ended and errors are recoverable. On Databricks, Agent
Bricks hosts agents built with any framework, and Genie Agents can be called as tools.

## Core Topologies

```text
SEQUENTIAL        CONCURRENT          SUPERVISOR (graph)        HANDOFFS (OpenAI Agents SDK)
A -> B -> C       A --|                  Supervisor             A --handoff--> B
                  B --|-> merge         /    |    \             B --handoff--> C
                  C --|                A     B     C            (agents delegate dynamically)
```

## Framework Comparison (PyPI versions as of 2026-10-17)

| Framework | Version | Model | Notes |
|-----------|---------|-------|-------|
| LangGraph | 1.2.16 | StateGraph + Functional API | Durable execution, interrupts; deploy with LangSmith Deployment |
| LangChain `create_agent` | `langchain` 1.4.6 | Agent loop on LangGraph | Replaces `langgraph.prebuilt.create_react_agent`; middleware |
| Claude Agent SDK | `claude-agent-sdk` 0.2.168 | Claude Code harness as a library | Built-in Read/Edit/Bash/Grep tools, subagents, hooks, MCP |
| OpenAI Agents SDK | `openai-agents` 0.24.0 (JS: `@openai/agents`) | Handoffs | Guardrails, tracing, sessions, MCP |
| Google ADK | `google-adk` 2.12.0 | ADK 2.0 | Python, TypeScript (GA), Go, Java |
| Pydantic AI | `pydantic-ai` 2.57.0 | Typed agent loop | Durable execution (Temporal, DBOS, Prefect) |
| Microsoft Agent Framework | `agent-framework` 1.22.0 | Multi-agent workflows | Successor to AutoGen (`microsoft/autogen` is archived); Python, .NET |
| CrewAI | 1.15.28 | Role-based crews | MCP via the `mcps` field on agents |
| Databricks Agent Bricks | platform | Agent Runtime + `DurableAgentServer` | Runs any of the above; memory on Lakebase, MLflow tracing |

## The Pattern: LangGraph Supervisor

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langchain.chat_models import init_chat_model

model = init_chat_model("claude-sonnet-5-5")   # no temperature: current Claude models reject it

def researcher(state: MessagesState):
    return {"messages": [model.invoke(state["messages"])]}

def writer(state: MessagesState):
    return {"messages": [model.invoke(state["messages"])]}

def route(state: MessagesState) -> str:
    return "writer" if "RESEARCH_COMPLETE" in state["messages"][-1].text else "researcher"

graph = StateGraph(MessagesState)
graph.add_node("researcher", researcher)
graph.add_node("writer", writer)
graph.add_edge(START, "researcher")
graph.add_conditional_edges("researcher", route, ["writer", "researcher"])
graph.add_edge("writer", END)
app = graph.compile()
```

## The Pattern: Claude Agent SDK with a Subagent

```python
import asyncio
from claude_agent_sdk import AgentDefinition, ClaudeAgentOptions, ResultMessage, query

async def main() -> None:
    options = ClaudeAgentOptions(
        system_prompt="You review pull requests for correctness and report issues with file:line.",
        allowed_tools=["Read", "Grep", "Glob"],        # auto-approved; others follow permission_mode
        permission_mode="default",
        max_turns=10,
        agents={"test-runner": AgentDefinition(
            description="Runs the test suite and summarizes failures",
            prompt="Run the tests and report each failing case.",
            tools=["Bash", "Read"], model="haiku")},
    )
    async for message in query(prompt="Review the current diff", options=options):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(main())
```

For multi-turn sessions use `ClaudeSDKClient`; for custom in-process tools use `@tool` + `create_sdk_mcp_server()` and pass
them through `ClaudeAgentOptions(mcp_servers=...)` with `allowed_tools=["mcp__<server>__<tool>"]`.

## Four Ways to Build a Claude Agent

| Approach | You write | Harness | Deployment |
|----------|-----------|---------|------------|
| Manual loop (`anthropic`) | The `while stop_reason == "tool_use"` loop | You | You |
| Tool Runner (`client.beta.messages.tool_runner`) | Tool functions | SDK | You |
| Claude Agent SDK | Prompt + options | Claude Code harness | You (or Agent Bricks Agent Runtime) |
| Managed Agents (beta) | Agent config + tool results | Anthropic | Anthropic (per-session sandbox) |

## Cost: Tiering vs One Model at Lower Effort

| Strategy | When it wins |
|----------|--------------|
| One capable model, `effort` tuned per route | Default first try: lower effort on current models often matches older models at high effort, and one model keeps one prompt-cache namespace |
| Planner (Opus 5.5) + workers (Sonnet 5.5 / Haiku 5.5) | Fan-out work dominated by reading or simple steps; measure cost per completed task, not per request |

## Common Mistakes

### Wrong

```python
# One agent with every tool and an "expert at everything" prompt
agent = create_agent(model="claude-opus-5-5", tools=ALL_50_TOOLS,
                     system_prompt="You are an expert at research, coding, review, deploy...")
```

### Correct

```python
from langchain.agents import create_agent

researcher = create_agent(model="claude-sonnet-5-5", tools=[web_search, doc_search],
                          system_prompt="Find and cite sources for the question.")
reviewer = create_agent(model="claude-opus-5-5", tools=[],
                        system_prompt="Check the draft against the cited sources.")
```

## Related

- [State Machines](../concepts/state-machines.md)
- [Tool Calling](../concepts/tool-calling.md)
- [Claude API Surface](../concepts/anthropic-api-surface.md)
- [Databricks Agent Platform](../concepts/databricks-agent-platform.md)
- [Agentic Workflow Pattern](../patterns/agentic-workflow.md)

## Sources

- https://code.claude.com/docs/en/agent-sdk/python
- https://docs.langchain.com/oss/python/releases/langchain-v1 ; https://docs.langchain.com/oss/python/migrate/langchain-v1 (`.text` is a property)
- https://docs.langchain.com/oss/python/langgraph/overview
- https://openai.github.io/openai-agents-python/ ; https://github.com/openai/openai-agents-js
- https://google.github.io/adk-docs/
- https://ai.pydantic.dev/
- https://github.com/microsoft/agent-framework ; https://github.com/microsoft/autogen
- https://docs.crewai.com/en/mcp/overview
- https://docs.databricks.com/aws/en/agents/concepts/
- https://pypi.org/ (versions for each package listed above)
