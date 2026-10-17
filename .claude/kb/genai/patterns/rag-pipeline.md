# RAG Pipeline Pattern

> **Purpose**: End-to-end RAG implementation with hybrid search, reranking, context engineering, and evaluation
> **MCP Validated**: 2026-10-17

## When to Use

- Question answering over private/enterprise documents
- Grounding LLM responses in specific knowledge bases
- Customer support with accurate, cited answers

## Architecture

```text
INGESTION                              QUERY
=========                              =====
Documents -> [Chunker] -> [Embedder]   User Query -> [Query Transform]
                             |                            |
                             v                            v
                       [Vector Store] <---------- [Hybrid Search]
                                                  (vector + BM25)
                                                       |
                                                  [Reranker] -> [LLM Generation] -> Response
```

## Implementation

```python
from dataclasses import dataclass, field
from typing import Optional
import hashlib

@dataclass
class Document:
    content: str
    metadata: dict
    doc_id: str = ""
    def __post_init__(self):
        if not self.doc_id:
            self.doc_id = hashlib.md5(self.content[:500].encode()).hexdigest()

@dataclass
class RetrievedChunk:
    text: str
    metadata: dict
    score: float
    source: str
    chunk_id: str

@dataclass
class RAGResponse:
    answer: str
    sources: list[dict]
    confidence: float
    chunks_used: int

class ProductionRAGPipeline:
    def __init__(self, vector_store, embedding_model, reranker, llm):
        self.vector_store = vector_store
        self.embedding_model = embedding_model
        self.reranker = reranker
        self.llm = llm

    def ingest(self, documents: list[Document], chunk_size: int = 512,
               chunk_overlap: int = 50):
        """Ingest documents: chunk, embed, store."""
        for doc in documents:
            chunks = self._chunk(doc, chunk_size, chunk_overlap)
            embeddings = self.embedding_model.embed_batch([c.text for c in chunks])
            for chunk, emb in zip(chunks, embeddings):
                self.vector_store.upsert(
                    id=chunk.chunk_id, vector=emb,
                    text=chunk.text, metadata=chunk.metadata,
                )

    def retrieve(self, query: str, top_k: int = 20,
                 rerank_top_n: int = 5) -> list[RetrievedChunk]:
        """Hybrid search with reranking."""
        expanded = self._expand_query(query)
        vector_results = self.vector_store.search(
            vector=self.embedding_model.embed(expanded), top_k=top_k)
        keyword_results = self.vector_store.keyword_search(query=expanded, top_k=top_k)
        fused = self._reciprocal_rank_fusion(vector_results, keyword_results)
        return self.reranker.rerank(query, fused[:top_k], top_n=rerank_top_n)

    def generate(self, query: str, chunks: list[RetrievedChunk]) -> RAGResponse:
        """Generate grounded, cited answer."""
        context_parts = [f"[Source {i+1}] {c.text}" for i, c in enumerate(chunks)]
        context = "\n---\n".join(context_parts)
        prompt = f"""Answer based ONLY on the provided sources.
Cite sources using [Source N]. If sources lack the answer, say so.

Sources:
{context}

Question: {query}
Answer:"""
        answer = self.llm.generate(prompt)
        sources = [{"source": c.source, "score": c.score} for c in chunks]
        return RAGResponse(
            answer=answer, sources=sources,
            confidence=sum(c.score for c in chunks) / len(chunks),
            chunks_used=len(chunks),
        )

    def _reciprocal_rank_fusion(self, *result_lists, k: int = 60) -> list:
        """Combine ranked lists using RRF."""
        scores: dict[str, float] = {}
        chunk_map: dict[str, RetrievedChunk] = {}
        for results in result_lists:
            for rank, chunk in enumerate(results):
                scores[chunk.chunk_id] = scores.get(chunk.chunk_id, 0) + 1/(k+rank+1)
                chunk_map[chunk.chunk_id] = chunk
        sorted_ids = sorted(scores, key=scores.get, reverse=True)
        return [chunk_map[cid] for cid in sorted_ids]
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `chunk_size` | `512` | Tokens per chunk |
| `chunk_overlap` | `50` | Overlap between chunks |
| `top_k` | `20` | Candidates from initial retrieval |
| `rerank_top_n` | `5` | Final chunks after reranking |
| `embedding_model` | `text-embedding-3-large` | Any current embedding model; see [embedding pipelines](../../ai-data-engineering/concepts/embedding-pipelines.md) |

## Example Usage

```python
pipeline = ProductionRAGPipeline(
    vector_store=MyVectorStore(index="knowledge-base"),       # adapter over your vector DB
    embedding_model=MyEmbedder(model="text-embedding-3-large"),  # adapter with embed / embed_batch
    reranker=MyReranker(model="rerank-v4.0-pro"),   # Cohere; rerank-v4.0-fast for lower latency
    llm=MyLLM(model="claude-sonnet-5-5"),           # adapter exposing generate(prompt)
)
docs = [Document(content=text, metadata={"source": "manual.pdf"}) for text in texts]
pipeline.ingest(docs)
question = "How do I reset my password?"
response = pipeline.generate(question, pipeline.retrieve(question))
```

## Variant: Databricks AI Search (managed hybrid retrieval)

When documents live in a lakehouse, let AI Search (formerly Vector Search) own embedding, sync, and hybrid scoring:

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
hits = w.vector_search_indexes.query_index(
    index_name="main.docs.chunks_index",            # Delta Sync index over a chunk table
    columns=["chunk_id", "chunk_to_retrieve", "source_path"],
    query_text="How do I reset my password?",
    query_type="HYBRID",                              # ANN + BM25 keyword scoring
    num_results=5,
)
chunks = [row[1] for row in hits.result.data_array]  # last column is the score
```

SQL equivalent: `SELECT * FROM vector_search(index => 'main.docs.chunks_index', query_text => '...',
query_type => 'HYBRID', num_results => 5)`. Build the chunk table with `ai_parse_document` + `ai_prep_search`; see
[Databricks-native RAG](../../ai-data-engineering/patterns/databricks-native-rag.md).

## See Also

- [RAG Architecture](../concepts/rag-architecture.md)
- [Evaluation Framework](../patterns/evaluation-framework.md)
- [Guardrails](../concepts/guardrails.md)

## Sources

- https://docs.cohere.com/docs/rerank
- https://platform.claude.com/docs/en/about-claude/pricing
- https://docs.databricks.com/aws/en/ai-search/ai-search/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/vector_search
