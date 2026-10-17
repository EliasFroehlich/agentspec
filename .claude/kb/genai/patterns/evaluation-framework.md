# Evaluation Framework Pattern

> **Purpose**: Evaluate LLM, RAG and agent quality with MLflow 3 GenAI evaluation (scorers + tracing), RAGAS, DeepEval, and LLM-as-judge gates in CI
> **MCP Validated**: 2026-10-17

## When to Use

- Measuring RAG quality (grounding, relevance, correctness) before and after a change
- Comparing prompt versions, models, or effort levels on the same dataset
- Continuous quality monitoring on sampled production traces
- Blocking a deploy when quality drops (CI gate)

## Tool Choice

| Tool | Version (PyPI, 2026-10-17) | Best for |
|------|----------------------------|----------|
| MLflow 3 GenAI (`mlflow.genai.evaluate`) | `mlflow` 3.17.1 | Default on Databricks: traces, built-in LLM judges, custom scorers, eval datasets, production monitoring, Prompt Registry |
| RAGAS | `ragas` 0.4.3 | Component-level RAG metrics |
| DeepEval | `deepeval` 4.3.1 | pytest-style tests in CI |
| Langfuse | `langfuse` 5.0.2 | Self-hosted tracing + scores outside Databricks |
| Arize Phoenix, LangSmith, Braintrust | — | Hosted or OSS tracing/eval UIs; pick by existing stack |

## Implementation: MLflow 3 (recommended on Databricks)

```python
import mlflow
from mlflow.genai.scorers import (
    Correctness, Guidelines, RelevanceToQuery, RetrievalGroundedness, Safety, scorer,
)

mlflow.anthropic.autolog()            # or mlflow.openai / mlflow.langchain; traces every call

eval_data = [
    {"inputs": {"question": "What is the refund window?"},
     "expectations": {"expected_facts": ["30 days", "original payment method"]}},
    {"inputs": {"question": "Do you ship to Canada?"},
     "expectations": {"expected_response": "Yes, standard shipping only."}},
]

def predict_fn(question: str) -> str:          # receives the unpacked "inputs" keys
    return rag_app.answer(question)

@scorer
def cites_a_source(outputs) -> bool:           # code-based check; name becomes the metric
    return "[Source" in outputs

results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=predict_fn,
    scorers=[
        Correctness(),                          # needs expected_facts or expected_response
        RelevanceToQuery(),
        RetrievalGroundedness(),                # needs a RETRIEVER span in the trace
        Safety(),
        Guidelines(name="tone", guidelines="The response must be polite and under 120 words."),
        cites_a_source,
    ],
)
print(results.metrics)                          # aggregates; per-row feedback is on each trace
```

Use `mlflow.genai.evaluate`, not the classic `mlflow.evaluate`. The same scorers can run on production traces
(Databricks: Evaluate & monitor), and judges can be pointed at a specific endpoint with `model="databricks:/<endpoint>"`.

## RAGAS (current class-based API)

```python
from langchain_anthropic import ChatAnthropic
from ragas import EvaluationDataset, evaluate
from ragas.llms import LangchainLLMWrapper
from ragas.metrics import FactualCorrectness, Faithfulness, LLMContextRecall

dataset = EvaluationDataset.from_list([
    {"user_input": q, "retrieved_contexts": ctxs, "response": answer, "reference": ref}
    for q, ctxs, answer, ref in rows
])
evaluator_llm = LangchainLLMWrapper(ChatAnthropic(model="claude-sonnet-5-5"))
result = evaluate(dataset=dataset,
                  metrics=[LLMContextRecall(), Faithfulness(), FactualCorrectness()],
                  llm=evaluator_llm)
```

## DeepEval (Anthropic judge)

```python
from deepeval import evaluate
from deepeval.metrics import AnswerRelevancyMetric, FaithfulnessMetric
from deepeval.models import AnthropicModel
from deepeval.test_case import LLMTestCase

judge = AnthropicModel(model="claude-opus-5-5")   # a bare model string is treated as OpenAI
case = LLMTestCase(
    input="What is RAG?",
    actual_output="RAG retrieves documents before generation.",
    retrieval_context=["RAG augments LLM responses with retrieved knowledge."],
)
evaluate([case], [FaithfulnessMetric(threshold=0.85, model=judge),
                  AnswerRelevancyMetric(threshold=0.80, model=judge)])
```

## LLM-as-Judge Best Practices

| Practice | Description |
|----------|-------------|
| Structured verdicts | Judge returns a schema (score, rationale) via structured outputs, not free text |
| Calibrate with humans | Compare judge labels with expert labels; on Databricks, align judges from SME feedback |
| Capable judge, cheaper app | Judge with `claude-opus-5-5` or `claude-sonnet-5-5`; never with a retired model |
| Explicit rubrics | Guidelines per criterion; one criterion per scorer |
| Pairwise comparisons | Compare two outputs side by side for A/B decisions |
| Hold the dataset fixed | Same inputs across versions; record model, prompt version, effort |

## CI/CD Quality Gate

```python
# Replace the keys with the exact names printed by results.metrics for your scorers
THRESHOLDS = {"correctness": 0.85, "relevance_to_query": 0.80, "safety": 0.99}

def ci_gate(results) -> None:
    """Fail the build when an aggregate metric drops below its threshold."""
    failures = [f"{k}: {results.metrics.get(k, 0):.2f} < {v}"
                for k, v in THRESHOLDS.items() if results.metrics.get(k, 0) < v]
    if failures:
        raise SystemExit("Quality gate failed: " + "; ".join(failures))
```


## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `judge_model` | `claude-opus-5-5` | Judge for DeepEval/RAGAS; MLflow built-in judges use the platform default unless `model=` is set |
| `dataset_size` | `50–200` rows | Enough to see a regression; grow from production traces |
| `min_correctness` | `0.85` | Gate threshold |
| `min_relevance` | `0.80` | Gate threshold |

## See Also

- [RAG Pipeline](../patterns/rag-pipeline.md)
- [Guardrails](../concepts/guardrails.md)
- [LLMOps Patterns](../../ai-data-engineering/concepts/llmops-patterns.md)

## Sources

- https://mlflow.org/docs/latest/genai/eval-monitor/
- https://mlflow.org/docs/latest/genai/tracing/
- https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/
- https://docs.ragas.io/en/stable/getstarted/rag_eval/
- https://deepeval.com/integrations/models/anthropic
- https://pypi.org/project/mlflow/ ; https://pypi.org/project/ragas/ ; https://pypi.org/project/deepeval/
