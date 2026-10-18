# Genie Agents (Databricks text-to-SQL)

> **Purpose**: Databricks Genie Agents (formerly Genie Spaces) — curated natural-language agents over Unity Catalog tables and metric views, their APIs, and how to make them accurate
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

The Genie family has three products: **Genie One** (business-user UI, mobile, Slack), **Genie Agents** (domain-specific
agents that data teams curate with tables, metric views, SQL examples and instructions; formerly Genie Spaces) and **Genie
Code** (coding assistant). A Genie Agent turns a question into SQL, runs it on a SQL warehouse and answers conversationally,
with the caller's access to the data and warehouse (users, or service principals with the needed permissions). Accuracy comes from curation — table/column comments, example SQL,
join specs, metric views and benchmarks — not from prompt tricks in the caller.

## The Concept

```python
# Chat mode API via databricks-sdk (POST /api/2.0/genie/spaces/{space_id}/... endpoints)
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()                                   # OAuth U2M for users, M2M for service principals

msg = w.genie.start_conversation(space_id=SPACE_ID,
                                 content="What were total sales last month by region?").result()
for att in msg.attachments or []:
    if att.query:
        print(att.query.query)                          # generated SQL
        result = w.genie.get_message_attachment_query_result(
            SPACE_ID, msg.conversation_id, msg.message_id, att.attachment_id)
        rows = result.statement_response.result.data_array
    if att.text:
        print(att.text.content)                         # answer summary or a clarifying question

follow_up = w.genie.create_message(space_id=SPACE_ID, conversation_id=msg.conversation_id,
                                   content="Only the top 3 regions").result()
```

```bash
# Same flow from the CLI (async: poll get-message until COMPLETED / FAILED / CANCELLED)
databricks genie start-conversation --no-wait SPACE_ID "What were total sales last month?"
databricks genie get-message SPACE_ID CONV_ID MSG_ID
```

## Quick Reference

| API family | Use | Notes |
|------------|-----|-------|
| Chat mode (Conversation API) | Ask questions, follow-ups, fetch SQL and results | Poll every 1–5 s with backoff (max ~1 min), stop after ~10 min |
| Agent mode API | Stream reasoning, SQL executions and a final report | Beta / preview enrollment; Responses-style streaming |
| Management API | `POST /api/2.0/genie/spaces`, `GET /api/2.0/genie/spaces` with `serialized_space` | CI/CD: export, version, recreate across workspaces |
| Genie One MCP server | External agents get business context over MCP | Recommended by Databricks for NL business questions |

| Curation element (`serialized_space`) | Purpose |
|---------------------------------------|---------|
| `data_sources.tables` / `metric_views` | Three-level names; descriptions and column configs |
| `instructions.text_instructions` | High-level guidance (at most one per agent) |
| `instructions.example_question_sqls` | Tested question → SQL pairs (aim for at least five) |
| `instructions.join_specs` | Joins with an explicit `--rt=FROM_RELATIONSHIP_TYPE_<CARDINALITY>--` annotation |
| `instructions.sql_snippets` | Reusable filters, expressions, measures |
| `benchmarks` | Ground-truth SQL per question to measure accuracy (aim for at least five) |

Limits and practices: 200,000 conversations per agent (delete old ones via the conversations endpoint); start a new
conversation per user session; implement your own retries with exponential backoff; requires the Databricks SQL
entitlement and `CAN USE` on a pro or serverless SQL warehouse. Billing: Genie One and Genie Agents usage by users is free
through 2027-01-31 (service principals are billed).

## Common Mistakes

### Wrong

```python
# Building a parallel text-to-SQL stack over the same governed tables
sql = llm(f"Schema:\n{ddl}\nQuestion: {q}\nReturn SQL only")
rows = spark.sql(sql).collect()          # bypasses curated joins, metric definitions and certified answers
```

### Correct

```python
# Improve the Genie Agent (comments, example SQL, join specs, metric views, benchmarks)
# and call it from apps or agents through the Conversation API or the Genie One MCP server.
msg = w.genie.start_conversation(space_id=SPACE_ID, content=q).result()
```

When the data source is a metric view, example SQL must follow `MEASURE()` rules: measures only through `MEASURE()` in
`SELECT`, never in `WHERE`/`GROUP BY`; filter aggregated results with `HAVING` or an outer query.

## Related

- [Text-to-SQL](../patterns/text-to-sql.md)
- [Databricks Agent Platform](../../genai/concepts/databricks-agent-platform.md)
- [LLMOps Patterns](../concepts/llmops-patterns.md)

## Sources

- https://docs.databricks.com/aws/en/genie/
- https://docs.databricks.com/aws/en/genie-agents/conversation-api
- https://docs.databricks.com/aws/en/agents/mcp-tools/
- https://github.com/databricks/databricks-sdk-py/blob/main/docs/workspace/dashboards/genie.rst
