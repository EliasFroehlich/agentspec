# Agentic Workflow Pattern

> **Purpose**: Multi-step agent workflow with plan-and-execute orchestration, structured plans, and per-role model/effort selection
> **MCP Validated**: 2026-10-17

## When to Use

- Tasks that need several reasoning steps and tool calls and are hard to specify fully in advance
- Work that decomposes into independent subtasks (fan-out) with a review step
- Pipelines where cheaper models or lower effort can execute steps planned by a stronger model
- Not for single extractions or classifications — use one structured-output call instead

## Architecture

```text
User request
   │
Planner (claude-opus-5-5, effort high)  ── structured Plan (2–5 tasks) via messages.parse
   │
   ├── Worker 1 (claude-sonnet-5-5, effort medium) + tools/MCP
   ├── Worker 2 (claude-sonnet-5-5, effort medium) + tools/MCP
   └── Worker 3 (claude-haiku-5-5, effort low)     + tools/MCP
   │
Reviewer (claude-opus-5-5) ── PASS → final output | NEEDS_REVISION → re-execute (bounded)
```

## Implementation (LangGraph orchestration + Anthropic SDK calls)

```python
import operator
from typing import Annotated, Literal, TypedDict

import anthropic
from pydantic import BaseModel
from langgraph.graph import END, START, StateGraph

client = anthropic.Anthropic()

class Task(BaseModel):
    id: str
    description: str
    worker: Literal["sonnet", "haiku"]

class Plan(BaseModel):
    tasks: list[Task]

class Review(BaseModel):
    verdict: Literal["PASS", "NEEDS_REVISION"]
    feedback: str

WORKERS = {"sonnet": ("claude-sonnet-5-5", "medium"), "haiku": ("claude-haiku-5-5", "low")}

class WorkflowState(TypedDict):
    request: str
    plan: list[Task]
    results: dict[str, str]
    review: Review | None
    attempts: Annotated[int, operator.add]

def text_of(resp) -> str:
    return "".join(b.text for b in resp.content if b.type == "text")   # blocks by type

def plan_step(state: WorkflowState):
    resp = client.messages.parse(
        model="claude-opus-5-5", max_tokens=16000,
        system="Decompose the request into 2-5 independent tasks a worker can finish alone.",
        messages=[{"role": "user", "content": state["request"]}],
        output_format=Plan,
    )
    return {"plan": resp.parsed_output.tasks}

def execute_step(state: WorkflowState):
    results = {}
    feedback = state["review"].feedback if state["review"] else ""
    for task in state["plan"]:
        model, effort = WORKERS[task.worker]
        resp = client.messages.create(
            model=model, max_tokens=16000, output_config={"effort": effort},
            messages=[{"role": "user", "content": f"{task.description}\n\nReviewer notes: {feedback}"}],
        )
        results[task.id] = text_of(resp)
    return {"results": results, "attempts": 1}

def review_step(state: WorkflowState):
    resp = client.messages.parse(
        model="claude-opus-5-5", max_tokens=16000,
        system="Check each result against its task. Say what must change if anything fails.",
        messages=[{"role": "user", "content": f"Plan: {state['plan']}\nResults: {state['results']}"}],
        output_format=Review,
    )
    return {"review": resp.parsed_output}

def route_review(state: WorkflowState) -> str:
    if state["review"].verdict == "PASS" or state["attempts"] >= 2:
        return END
    return "execute"

graph = StateGraph(WorkflowState)
graph.add_node("plan", plan_step)
graph.add_node("execute", execute_step)
graph.add_node("review", review_step)
graph.add_edge(START, "plan")
graph.add_edge("plan", "execute")
graph.add_edge("execute", "review")
graph.add_conditional_edges("review", route_review, [END, "execute"])
workflow = graph.compile()
```

## Cost Comparison (illustrative)

```text
Assumption per step: 2,000 input + 500 output tokens (thinking counts as output), no caching.
Prices per MTok as of 2026-10-17: Opus 5.5 $4/$20, Sonnet 5.5 $2/$10, Haiku 5.5 $0.10/$0.50.

Opus 5.5 step  = 2000×4/1M + 500×20/1M   = $0.018
Sonnet 5.5 step = 2000×2/1M + 500×10/1M  = $0.009
Haiku 5.5 step  = 2000×0.10/1M + 500×0.50/1M = $0.00045

10 steps, Opus everywhere          = $0.180
2 Opus (plan, review) + 8 Sonnet   = $0.036 + $0.072  = $0.108  (~40% less)
2 Opus (plan, review) + 8 Haiku    = $0.036 + $0.0036 = $0.040  (~78% less)
```

Measure before committing to a cascade: one capable model with `effort` tuned per route often matches it, keeps a single
prompt-cache namespace (caches are per model), and is simpler. Judge cost per completed task, including retries.
OpenAI equivalents for a mixed stack: `gpt-6-astra` ($10/$50), `gpt-6.1-sol` ($2/$10), `gpt-6-luna` ($0.10/$0.50).

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `planner_model` | `claude-opus-5-5` | Planning and review; thinking always on, effort default `medium` |
| `worker_model` | `claude-sonnet-5-5` | Task execution; `claude-haiku-5-5` for simple, high-volume steps |
| `effort` | per role | `high` for planning/review, `medium`/`low` for workers |
| `max_attempts` | `2` | Bound on review → re-execute loops |
| `max_tokens` | `16000` | Leave room for thinking; stream for larger values |
| `parallel_execution` | `False` | Run independent tasks concurrently (`asyncio` / LangGraph `Send`) |

## Example Usage

```python
result = workflow.invoke({
    "request": "Compare the three most-used open-source LLM observability tools and write a short report.",
    "plan": [], "results": {}, "review": None, "attempts": 0,
})
print(result["review"].verdict, list(result["results"]))
```

## See Also

- [Multi-Agent Systems](../concepts/multi-agent-systems.md)
- [Claude API Surface](../concepts/anthropic-api-surface.md)
- [Tool Calling](../concepts/tool-calling.md)
- [Evaluation Framework](../patterns/evaluation-framework.md)

## Sources

- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/effort
- https://platform.openai.com/docs/models
- https://docs.langchain.com/oss/python/langgraph/overview
