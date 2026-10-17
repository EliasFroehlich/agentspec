# Tool Calling

> **Purpose**: LLM tool/function calling across providers, the Model Context Protocol (MCP 2026-07-28), and safe argument handling
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Tool calling lets a model request a function call with structured JSON arguments; your application executes it and returns
the result. The model never executes anything itself. Each provider uses a slightly different tool schema, so keep one
provider-neutral registry and export per provider. MCP standardizes tool discovery and invocation across clients; its
current revision (2026-07-28) is stateless and supports two standard transports, stdio and Streamable HTTP.

## The Pattern

```python
from dataclasses import dataclass
from typing import Any, Callable

@dataclass
class ToolDefinition:
    name: str
    description: str          # 3-4+ sentences: what, when (and when not), each parameter
    parameters: dict          # JSON Schema with "required" and "additionalProperties": False
    function: Callable
    requires_confirmation: bool = False

class ToolRegistry:
    """Provider-neutral registry; export the schema each API expects."""
    def __init__(self):
        self._tools: dict[str, ToolDefinition] = {}

    def register(self, tool: ToolDefinition) -> None:
        self._tools[tool.name] = tool

    def anthropic_tools(self) -> list[dict]:   # Claude Messages API
        return [{"name": t.name, "description": t.description,
                 "input_schema": t.parameters, "strict": True} for t in self._tools.values()]

    def openai_tools(self) -> list[dict]:      # OpenAI Responses API function tools
        return [{"type": "function", "name": t.name, "description": t.description,
                 "parameters": t.parameters, "strict": True} for t in self._tools.values()]

    def execute(self, name: str, arguments: dict) -> Any:
        tool = self._tools.get(name)
        if tool is None:
            raise ValueError(f"Unknown tool: {name}")
        if tool.requires_confirmation:
            raise PermissionError(f"{name} needs human approval")
        return tool.function(**arguments)
```

## MCP: server and client (Python SDK `mcp` 2.x)

```python
# mcp 2.3.0 (PyPI, 2026-10-02). In 1.x the server class was mcp.server.fastmcp.FastMCP.
from mcp.server import MCPServer

mcp = MCPServer("warehouse-tools")

@mcp.tool()
def row_count(table: str) -> int:
    """Return the row count of a Unity Catalog table given as catalog.schema.table."""
    return run_readonly(f"SELECT count(*) FROM {quote_ident(table)}")
# Resources: @mcp.resource("docs://{page}") exposes read-only context the same way
```

```python
import asyncio
from mcp import Client

async def main() -> None:
    async with Client("http://localhost:8000/mcp") as client:   # a URL means Streamable HTTP
        result = await client.call_tool("row_count", {"table": "main.sales.orders"})
        print(result.structured_content)

asyncio.run(main())
```

| MCP 2026-07-28 change | Practical effect |
|-----------------------|------------------|
| Stateless: no `initialize` handshake, no `Mcp-Session-Id` | Every request carries protocol version and client capabilities in `_meta`; servers implement `server/discover` |
| Multi Round-Trip Requests (`resultType: "input_required"`) | Replace server-initiated sampling/elicitation/roots requests |
| `subscriptions/listen` | Replaces the HTTP GET stream and `resources/subscribe` |
| SSE resumability removed | A broken stream loses the request; re-issue it with a new ID |
| **Deprecated**: Roots, Sampling, Logging; HTTP+SSE transport; OAuth Dynamic Client Registration (use Client ID Metadata Documents) | Do not adopt them in new servers |

## Provider Comparison

| Feature | Anthropic (Messages) | OpenAI (Responses) | MCP |
|---------|----------------------|--------------------|-----|
| Tool schema key | `input_schema` | `parameters` | `inputSchema` (JSON Schema 2020-12) |
| Schema guarantee | `strict: true` on the tool | `strict: true` on the function | Server validates |
| Forcing a call | `tool_choice` `any`/`tool` returns 400 on Fable 5.1, Opus 5.5, Sonnet 5.5 — use `auto` + prompt + `strict` | `tool_choice: {"type": "function", "name": ...}` | N/A |
| Large tool catalogs | Tool search (`tool_search_tool_regex_20251119` / `_bm25_20251119`) + `defer_loading` | `defer_loading` on function tools + tool search | Deterministic `tools/list` order helps prompt caching |
| Remote MCP from the API | MCP connector, beta `mcp-client-2025-11-20` (HTTP servers only) | `{"type": "mcp", "server_label": ...}` tool | — |
| Computer use | `computer_toolset_20260801` on Opus 5.5 / Sonnet 5.5 (Claude API, Google Cloud) | Provider tool | Via tools |
| Agent SDK | Claude Agent SDK | OpenAI Agents SDK | Supported by most frameworks |

## Common Mistakes

### Wrong

```python
# Trusting model-generated arguments and string-building SQL
args = json.loads(call.arguments)
db.execute(f"SELECT * FROM {args['table']} WHERE id = {args['id']}")
```

### Correct

```python
from pydantic import BaseModel, Field

class LookupArgs(BaseModel):
    table: str = Field(pattern=r"^[a-z_]+\.[a-z_]+\.[a-z_]+$")   # allowlisted shape
    id: int = Field(ge=1)

args = LookupArgs.model_validate(tool_use.input)    # Claude: dict already parsed
if args.table not in ALLOWED_TABLES:
    raise PermissionError(args.table)
db.execute("SELECT * FROM IDENTIFIER(:t) WHERE id = :id", {"t": args.table, "id": args.id})
```

## Best Practices

| Practice | Rationale |
|----------|-----------|
| Validate every argument (Pydantic) and allowlist identifiers | Prevents injection and path traversal |
| Require confirmation for destructive tools | OWASP LLM06 Excessive Agency |
| Write detailed tool descriptions | Under-description is the most common tool-use failure |
| Return all parallel `tool_result` blocks in one user message; `is_error: true` on failure | Keeps parallel calling working |
| Use tool search / `defer_loading` instead of a fixed tool-count rule | Large catalogs stay usable |

## Related

- [Claude API Surface](../concepts/anthropic-api-surface.md)
- [Multi-Agent Systems](../concepts/multi-agent-systems.md)
- [Guardrails](../concepts/guardrails.md)

## Sources

- https://modelcontextprotocol.io/specification/2026-07-28/changelog
- https://modelcontextprotocol.io/specification/2026-07-28/basic/transports
- https://github.com/modelcontextprotocol/python-sdk ; https://pypi.org/project/mcp/
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool
- https://platform.claude.com/docs/en/agents-and-tools/mcp-connector
- https://platform.openai.com/docs/guides/structured-outputs
- https://developers.openai.com/api/reference/resources/responses/methods/create
