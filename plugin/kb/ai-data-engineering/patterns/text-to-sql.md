# Text-to-SQL

> **Purpose**: Natural-language querying over structured data — Genie Agents first when the data is on Databricks, and a guarded, single-dialect pipeline only when it is not
> **MCP Validated**: 2026-10-17

## When to Use

- Self-service analytics or a conversational BI layer over warehouse tables
- An agent or app that needs governed answers to data questions
- **Data in Unity Catalog** → curate a Genie Agent and call it (Conversation API or Genie One MCP server); do not rebuild it
- **Data elsewhere** (e.g. PostgreSQL) → the guarded pipeline below: schema context, structured SQL generation,
  read-only execution, row limits and evaluation

## Implementation A: Genie Agent as a tool (Databricks)

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

def ask_genie(space_id: str, question: str, conversation_id: str | None = None) -> dict:
    """Ask a curated Genie Agent; returns SQL, rows and any text reply. Retries are the caller's job."""
    waiter = (w.genie.create_message(space_id=space_id, conversation_id=conversation_id, content=question)
              if conversation_id else w.genie.start_conversation(space_id=space_id, content=question))
    out = {"conversation_id": conversation_id, "sql": None, "rows": None, "columns": None, "text": None}
    try:
        msg = waiter.result()                               # polls until a terminal status
    except Exception as exc:                                # failed/cancelled message or timeout
        out["text"] = f"Genie error: {exc}"
        return out
    out["conversation_id"] = msg.conversation_id
    if msg.error:                                           # FAILED messages carry an error field
        out["text"] = f"Genie error: {msg.error}"
        return out
    for att in msg.attachments or []:
        if att.query:
            res = w.genie.get_message_attachment_query_result(
                space_id, msg.conversation_id, msg.message_id, att.attachment_id)
            sr = res.statement_response
            out["sql"] = att.query.query
            out["columns"] = [c.name for c in sr.manifest.schema.columns]
            out["rows"] = sr.result.data_array
        if att.text:
            out["text"] = att.text.content                 # summary, or a clarifying question
    return out
```

```python
# Expose it to Claude as a tool (strict schema; tool_choice stays "auto")
GENIE_TOOL = {
    "name": "ask_sales_data",
    "description": ("Answer questions about orders, revenue and customers from the governed sales "
                    "lakehouse. Use it for any numeric or factual question about sales; do not use it for "
                    "general knowledge. Returns the SQL that ran and up to 1,000 rows."),
    "strict": True,
    "input_schema": {"type": "object", "additionalProperties": False, "required": ["question"],
                     "properties": {"question": {"type": "string",
                                                 "description": "Self-contained question in plain language"}}},
}
```

Accuracy work happens in the Genie Agent, not in the caller: column comments, at least five tested example SQL queries,
join specs, metric views, and benchmarks. See [Genie Agents](../concepts/genie-agents.md).

## Implementation B: Guarded pipeline outside Databricks (PostgreSQL)

```python
import anthropic
import sqlparse
from pydantic import BaseModel, Field
from sqlalchemy import create_engine, inspect, text

client = anthropic.Anthropic()
engine = create_engine(PG_URL)                 # a read-only database role is the first guardrail

class SQLAnswer(BaseModel):
    sql: str = Field(description="One PostgreSQL SELECT statement with a LIMIT")
    tables_used: list[str]

def schema_context(tables: list[str]) -> str:
    insp = inspect(engine)
    return "\n".join(f"{t}({', '.join(c['name'] + ' ' + str(c['type']) for c in insp.get_columns(t))})"
                     for t in tables)

def generate_sql(question: str, schema: str, feedback: str = "") -> SQLAnswer:
    resp = client.messages.parse(
        model="claude-sonnet-5-5", max_tokens=16000,
        system=("You write PostgreSQL for analysts. Use only the tables and columns in <schema>. "
                "Read-only SELECT with LIMIT 1000 at most, because results feed a UI table."),
        messages=[{"role": "user",
                   "content": f"<schema>\n{schema}\n</schema>\n\n{question}\n{feedback}"}],
        output_format=SQLAnswer,
    )
    return resp.parsed_output

FORBIDDEN = {"INSERT", "UPDATE", "DELETE", "DROP", "ALTER", "TRUNCATE", "GRANT", "COPY", "CALL"}

def validate(sql: str) -> str | None:
    stmts = [s for s in sqlparse.parse(sql) if s.token_first() is not None]
    if len(stmts) != 1 or stmts[0].get_type() != "SELECT":
        return "exactly one SELECT statement is allowed"
    if any(tok.normalized in FORBIDDEN for tok in stmts[0].flatten()):
        return "forbidden keyword"
    return None

def run(sql: str, timeout_s: int = 30, max_rows: int = 1000) -> dict:
    with engine.begin() as conn:                              # one transaction
        conn.execute(text("SET TRANSACTION READ ONLY"))
        conn.execute(text(f"SET LOCAL statement_timeout = '{int(timeout_s)}s'"))
        result = conn.execute(text(sql))
        rows = result.fetchmany(max_rows)
        return {"columns": list(result.keys()), "rows": rows, "truncated": len(rows) == max_rows}

def text_to_sql(question: str, tables: list[str], max_attempts: int = 3) -> dict:
    schema, feedback = schema_context(tables), ""
    for _ in range(max_attempts):
        answer = generate_sql(question, schema, feedback)
        if err := validate(answer.sql):
            feedback = f"<previous_error>{err}</previous_error>"
            continue
        try:
            return {"sql": answer.sql, **run(answer.sql)}
        except Exception as exc:                              # feed the DB error back once more
            feedback = f"<previous_sql>{answer.sql}</previous_sql><previous_error>{exc}</previous_error>"
    raise RuntimeError(f"no valid SQL after {max_attempts} attempts")
```

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| Genie polling | SDK waiter (`.result()`) | Docs: poll every 1–5 s with backoff, give up after ~10 min |
| `max_rows` | 1000 | Row cap per result |
| `timeout_s` | 30 | `SET LOCAL statement_timeout` (PostgreSQL) |
| `max_attempts` | 3 | Generate → validate → execute loop |
| Database role | read-only | Enforce read-only in the database, not only in code |
| Model | `claude-sonnet-5-5` | Structured output for SQL; no `temperature` on current Claude models |

## Example Usage

```python
r = ask_genie(SPACE_ID, "Revenue by region for the last full quarter")
print(r["sql"], r["columns"], r["rows"][:3] if r["rows"] else r["text"])

follow = ask_genie(SPACE_ID, "Only the top 3 regions", conversation_id=r["conversation_id"])

pg = text_to_sql("How many orders last month?", ["orders", "customers"])
```

Evaluate either path with fixed question sets: Genie benchmarks inside the agent, or `mlflow.genai.evaluate` with
`Correctness` and custom scorers that compare result sets.

## See Also

- [Genie Agents](../concepts/genie-agents.md)
- [LLMOps Patterns](../concepts/llmops-patterns.md)
- [RAG Pipeline Implementation](rag-pipeline-implementation.md)

## Sources

- https://docs.databricks.com/aws/en/genie-agents/conversation-api
- https://github.com/databricks/databricks-sdk-py/blob/main/docs/workspace/dashboards/genie.rst
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://www.postgresql.org/docs/current/runtime-config-client.html (statement_timeout)
- https://www.postgresql.org/docs/current/sql-set-transaction.html
