# Embedding Pipelines

> **Purpose**: Embedding model selection, batch vs streaming execution, Matryoshka dimensions, multimodal embeddings, platform-managed embeddings, and versioning/drift
> **Confidence**: 0.85
> **MCP Validated**: 2026-10-17

## Overview

Embedding pipelines turn text, images or documents into vectors for retrieval and ML. Choose the model by modality,
language, context length and dimension budget, then pin it: every vector must record the model and dimensions that
produced it. Current generations (as of 2026-10-17) include Voyage 4, Cohere Embed v5, Gemini Embedding 2 (multimodal),
OpenAI `text-embedding-3-*`, and on Databricks, managed embeddings computed by AI Search from Foundation Model API
endpoints. Benchmarks move quickly — evaluate candidates on your own queries rather than relying on a leaderboard snapshot.

## The Concept

```python
# Batch embedding with retries, dimension control and version metadata (OpenAI SDK)
from datetime import datetime, timezone

from openai import OpenAI
from tenacity import retry, stop_after_attempt, wait_exponential

client = OpenAI()
MODEL, DIMENSIONS = "text-embedding-3-large", 1024      # `dimensions` shortens text-embedding-3 vectors

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
def embed_batch(texts: list[str]) -> list[list[float]]:
    # Limits: 8,192 tokens per input, 2,048 inputs and 300,000 tokens per request
    resp = client.embeddings.create(model=MODEL, input=texts, dimensions=DIMENSIONS)
    return [item.embedding for item in resp.data]

def embed_documents(docs: list[dict], batch_size: int = 100) -> list[dict]:
    out = []
    for i in range(0, len(docs), batch_size):
        batch = docs[i:i + batch_size]
        for doc, vec in zip(batch, embed_batch([d["content"] for d in batch])):
            out.append({**doc, "embedding": vec, "embedding_model": MODEL, "dimensions": DIMENSIONS,
                        "embedded_at": datetime.now(timezone.utc).isoformat()})
    return out
```

```python
# Gemini Embedding 2 with the current SDK (google-genai; google-generativeai is deprecated and archived)
from google import genai
from google.genai import types

gclient = genai.Client()
result = gclient.models.embed_content(
    model="gemini-embedding-2",
    contents="What is the meaning of life?",
    config=types.EmbedContentConfig(output_dimensionality=768),   # 768 / 1536 / 3072 recommended
)
[embedding] = result.embeddings
vector = embedding.values
```

Gemini Embedding 2 is multimodal (text, image, video, audio, PDF; 8,192-token text limit), has no `task_type` (put the task
instruction in the text) and returns **one aggregated embedding** for multiple inputs — loop or use the Batch API for
per-item vectors. `gemini-embedding-001` (text only) embeds lists item by item and needs manual normalization below 3072
dimensions; Gemini Embedding 2 renormalizes truncated dimensions automatically.

## Quick Reference (as of 2026-10-17)

| Model | Provider | Dimensions | Context | Notes |
|-------|----------|-----------|---------|-------|
| `voyage-4-large` / `voyage-4` / `voyage-4-lite` | Voyage AI | 1024 default; 256, 512, 2048 | 32K | Series embeddings are mutually compatible; `voyage-code-4` for code |
| `voyage-4-nano` | Voyage AI | 1024 default; 256, 512, 2048 | 32K | Open-weight (Hugging Face) |
| `embed-v5.0-pro` / `embed-v5.0-fast` | Cohere | 256–2048 (2048 default) | 128K | Text, images, mixed (PDFs) |
| `embed-v4.0` | Cohere | 256, 512, 1024, 1536 (default) | 128K | Previous generation, multimodal |
| `gemini-embedding-2` | Google | 3072 default (MRL) | 8,192 tokens | Multimodal, auto-normalized |
| `text-embedding-3-large` / `-small` | OpenAI | `dimensions` parameter | 8,192 tokens | `text-embedding-ada-002` is the older model |
| `databricks-qwen3-embedding-0-6b` | Databricks FMAPI (Public Preview) | up to 1024 | ~32K | 100+ languages |
| `databricks-gte-large-en` / `databricks-bge-large-en` | Databricks | — | — | Referenced as AI Search managed-embedding models |
| BGE-M3, Nomic embed | Open source | 1024 / 768 | 8K | Self-hosted (e.g. `sentence-transformers` 6.1) |

| Mode | Latency | Use case |
|------|---------|----------|
| Batch API / Spark job | Minutes–hours | Backfills, re-embedding after a model change |
| Platform-managed (AI Search Delta Sync) | Sync interval | Lakehouse tables; no embedding job to own |
| Streaming / per request | Real time | New documents, query embeddings |
| Cache | < 1 ms | Repeated queries |

Matryoshka truncation (`dimensions` / `output_dimensionality`) applies to OpenAI `text-embedding-3-*` and Gemini Embedding
001/2; Voyage 3.5/4 and Cohere Embed v4/v5 offer a fixed set of output dimensions. Store the chosen dimension with every
vector.

## Common Mistakes

### Wrong

```python
# No version tracking: impossible to detect drift or mixed models later
vectors = embed(texts)
db.upsert(ids=ids, embeddings=vectors)
```

### Correct

```python
import numpy as np

db.upsert(ids=ids, embeddings=vectors, metadatas=[{
    "embedding_model": MODEL, "dimensions": DIMENSIONS,
    "embedded_at": datetime.now(timezone.utc).isoformat()} for _ in ids])

# Drift check: cosine distance between centroids of stored vs new embeddings
c_old, c_new = np.mean(stored_vectors, axis=0), np.mean(new_vectors, axis=0)
drift = 1 - (c_old @ c_new) / (np.linalg.norm(c_old) * np.linalg.norm(c_new))
if drift > 0.05:
    alert("Embedding drift detected -- review content or re-embed")
```

Never mix models or dimensions in one index; re-embed the whole corpus (or build a new index) when you change either.

## Related

- [rag-pipelines](../concepts/rag-pipelines.md)
- [vector-databases](../concepts/vector-databases.md)
- [databricks-ai-search](../concepts/databricks-ai-search.md)

## Sources

- https://docs.voyageai.com/docs/embeddings
- https://docs.cohere.com/docs/cohere-embed
- https://ai.google.dev/gemini-api/docs/embeddings
- https://github.com/google-gemini/deprecated-generative-ai-python
- https://developers.openai.com/api/docs/api-reference/embeddings/create
- https://docs.databricks.com/aws/en/machine-learning/foundation-model-apis/supported-models
- https://docs.databricks.com/aws/en/generative-ai/agent-bricks/knowledge-assistant
