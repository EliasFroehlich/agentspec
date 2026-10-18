# Databricks Agent Platform (Genie, Agent Bricks, Unity Gateway)

> **Purpose**: Map the Databricks-native GenAI layer — Genie One/Agents/Code, Agent Bricks, Unity Gateway, Foundation Model APIs, AI Search, MLflow — and when to use each instead of building it yourself
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

On Databricks, most GenAI building blocks are managed and governed by Unity Catalog. **Genie Agents** (formerly Genie Spaces)
answer natural-language questions over curated tables and metric views without code. **Agent Bricks** is the developer
platform for agents written in code with any framework: Agent Runtime, AgentKit (durable execution), `DurableAgentServer`,
Databricks Sandbox, and memory/sessions on Lakebase. **Unity Gateway** is the single entry point for models, MCP servers and
skills. Knowledge Assistant and Supervisor Agent still exist but are marked **Legacy** in the docs (2026-10-14).

## The Concept

```text
            Business users                     Developers (code)
                 │                                     │
            Genie One  ──calls──►  Genie Agents   Agent Bricks CLI (scaffold → run → deploy)
           (UI, mobile, Slack)    (tables, metric      │
                                   views, documents)   ▼
                 ▲                        ▲        Agent Server (DurableAgentServer)
                 │                        │        on Agent Runtime (Databricks Apps)
            Genie One MCP  ◄──── tool ────┴──────  your loop: LangGraph / OpenAI Agents SDK /
                                                    Claude Agent SDK / plain SDK
                                                         │
     Unity Gateway: models (system.ai.*), MCP servers, skills, rate limits, budgets, PII masking
                                                         │
     Context: AI Search indexes · Genie Agents · UC functions · Sandbox · memory/sessions (Lakebase)
     Quality: MLflow Tracing (OTel) · mlflow.genai.evaluate + scorers · production monitoring
```

```python
# Query a Genie Agent from your own agent or app (databricks-sdk; Chat mode API)
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
msg = w.genie.start_conversation(space_id=SPACE_ID, content="Revenue by region last quarter").result()
for att in msg.attachments or []:
    if att.query:   # generated SQL + description
        rows = w.genie.get_message_attachment_query_result(
            SPACE_ID, msg.conversation_id, msg.message_id, att.attachment_id)
    elif att.text:  # clarifying question or summary
        print(att.text.content)
```

## Quick Reference

| Need | Databricks-native option | Notes |
|------|--------------------------|-------|
| NL questions over governed tables / metric views | Genie Agents + Conversation (Chat mode) API | Curate tables, SQL examples, instructions; see [Genie Agents](../../ai-data-engineering/concepts/genie-agents.md) |
| Agent that needs business context from any agent framework | Genie One MCP server (Databricks-provided MCP) | Recommended for NL business questions; Databricks SQL MCP for explicit SQL |
| Custom agent in code | Agent Bricks CLI → Agent Server → Agent Runtime | Any framework or harness; `agentbricks deploy` provisions resources |
| Model access without per-provider keys | Unity Gateway / Foundation Model APIs (`system.ai.<model>`, e.g. `system.ai.claude-opus-5-5`, endpoint `databricks-claude-opus-5-5`) | Also BYOK external providers as UC securables |
| Retrieval over documents | AI Search (formerly Vector Search) | See [Databricks AI Search](../../ai-data-engineering/concepts/databricks-ai-search.md) |
| Batch LLM inference in SQL/Spark | AI Functions (`ai_query`, `ai_extract`, `ai_classify`, `ai_parse_document`, …) | DBR 15.4 LTS+ baseline; some functions need newer runtimes |
| Tracing and evaluation | MLflow 3 (`mlflow.<lib>.autolog()`, `mlflow.genai.evaluate`, scorers) | See [Evaluation Framework](../patterns/evaluation-framework.md) |
| Prompt versioning | MLflow Prompt Registry (`mlflow.genai.register_prompt`, aliases) | See [prompt-template](../../prompt-engineering/patterns/prompt-template.md) |

| Offering | Status (2026-10-17) | Use instead for new work |
|----------|---------------------|--------------------------|
| Knowledge Assistant | Legacy | Genie Agents (low code) or Agent Bricks CLI (code) |
| Supervisor Agent | Legacy | Genie Agents (low code) or Agent Bricks CLI (code) |
| Genie Spaces (name) | Renamed | Genie Agents |
| Vector Search (name) | Renamed | AI Search |

Billing note: Genie One and Genie Agents usage by users is free through 2027-01-31 (service principals excluded); Genie Code is
pay-as-you-go from 2026-07-08 ([Genie](https://docs.databricks.com/aws/en/genie/)).

## Common Mistakes

### Wrong

```python
# Re-implementing text-to-SQL with a raw LLM call over DDL when the data already lives in Unity Catalog
sql = llm(f"Schema: {ddl}\nWrite SQL for: {question}")
spark.sql(sql)   # no curation, no governance, no certified answers
```

### Correct

```python
# Curate a Genie Agent once (tables, joins, SQL examples, instructions) and call it as a tool
msg = w.genie.start_conversation(space_id=SPACE_ID, content=question).result()
```

## Related

- [Multi-Agent Systems](../concepts/multi-agent-systems.md)
- [Tool Calling](../concepts/tool-calling.md)
- [Text-to-SQL](../../ai-data-engineering/patterns/text-to-sql.md)
- [Databricks-native RAG](../../ai-data-engineering/patterns/databricks-native-rag.md)

## Sources

- https://docs.databricks.com/aws/en/genie/
- https://docs.databricks.com/aws/en/genie-agents/conversation-api
- https://docs.databricks.com/aws/en/agents/concepts/
- https://docs.databricks.com/aws/en/agents/mcp-tools/
- https://docs.databricks.com/aws/en/unity-gateway/
- https://docs.databricks.com/aws/en/machine-learning/foundation-model-apis/supported-models
- https://docs.databricks.com/aws/en/generative-ai/agent-bricks/knowledge-assistant
- https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor
- https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://docs.databricks.com/aws/en/large-language-models/ai-functions
- https://github.com/databricks/databricks-sdk-py/blob/main/docs/workspace/dashboards/genie.rst
