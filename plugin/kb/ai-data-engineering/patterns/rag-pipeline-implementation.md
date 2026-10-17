# RAG Pipeline Implementation (LangChain 1.x)

> **Purpose**: End-to-end RAG with LangChain 1.x packages — loading, splitting, embedding, Chroma storage, two-step and agentic retrieval, and RAGAS evaluation
> **MCP Validated**: 2026-10-17

## When to Use

- Question answering over private documents with source attribution, outside a managed platform
- Prototyping retrieval quickly before choosing a production vector store
- On Databricks, prefer [Databricks-native RAG](databricks-native-rag.md) (managed parsing, chunking and AI Search)

Package versions on PyPI as of 2026-10-17: `langchain` 1.4.6, `langchain-core` 1.6.12, `langchain-text-splitters` 1.1.3,
`langchain-chroma` 1.1.0, `langchain-community` 0.4.2 (loaders), `langchain-anthropic` 1.7.6, `langchain-openai` 1.7.2.
Legacy chains such as `RetrievalQA` moved to `langchain-classic` (`langchain_classic.chains`).

## Implementation

```python
"""RAG with LangChain 1.x: loaders -> splitter -> embeddings -> Chroma -> retrieval -> answer."""

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.tools import tool
from langchain_chroma import Chroma
from langchain_community.document_loaders import DirectoryLoader, PyPDFLoader, TextLoader
from langchain_openai import OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

# --- 1. Load ---
documents = (
    DirectoryLoader("data/documents/", glob="**/*.pdf", loader_cls=PyPDFLoader).load()
    + DirectoryLoader("data/documents/", glob="**/*.txt", loader_cls=TextLoader).load()
)

# --- 2. Split ---
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200,
                                          separators=["\n\n", "\n", ". ", " ", ""])
chunks = splitter.split_documents(documents)

# --- 3. Embed + store ---
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
    persist_directory="data/chroma_db",
    collection_name="rag_documents",
)
retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 4, "fetch_k": 20})

# --- 4a. Two-step RAG: always retrieve, then answer (predictable cost and latency) ---
model = init_chat_model("claude-sonnet-5-5")          # no temperature: current Claude models reject it

def answer(question: str) -> dict:
    docs = retriever.invoke(question)
    sources = "\n".join(
        f"<source id='{i}' file='{d.metadata.get('source')}' page='{d.metadata.get('page')}'>\n"
        f"{d.page_content}\n</source>" for i, d in enumerate(docs))
    reply = model.invoke([
        {"role": "system", "content": "Answer only from the sources and cite source ids. "
                                      "If the sources lack the answer, say you don't know."},
        {"role": "user", "content": f"<sources>\n{sources}\n</sources>\n\n{question}"},
    ])
    return {"answer": reply.text, "contexts": [d.page_content for d in docs]}

# --- 4b. Agentic RAG: the model decides when and what to retrieve ---
@tool
def search_docs(query: str) -> str:
    """Search the internal policy documents and return the most relevant passages with sources."""
    docs = retriever.invoke(query)
    return "\n\n".join(f"[{d.metadata.get('source')} p.{d.metadata.get('page')}] {d.page_content}"
                       for d in docs)

agent = create_agent(
    model="claude-sonnet-5-5",
    tools=[search_docs],
    system_prompt="Use search_docs for any question about internal policies and cite the sources.",
)
result = agent.invoke({"messages": [{"role": "user", "content": "What are the data retention policies?"}]})
print(result["messages"][-1].text)
```

## Evaluation with RAGAS (current API)

```python
from langchain_anthropic import ChatAnthropic
from ragas import EvaluationDataset, evaluate
from ragas.llms import LangchainLLMWrapper
from ragas.metrics import FactualCorrectness, Faithfulness, LLMContextRecall

questions = ["What are the data retention policies?"]
references = ["Data is retained for 7 years per compliance requirements."]

rows = []
for q, ref in zip(questions, references):
    out = answer(q)
    rows.append({"user_input": q, "retrieved_contexts": out["contexts"],
                 "response": out["answer"], "reference": ref})

result = evaluate(
    dataset=EvaluationDataset.from_list(rows),
    metrics=[LLMContextRecall(), Faithfulness(), FactualCorrectness()],
    llm=LangchainLLMWrapper(ChatAnthropic(model="claude-sonnet-5-5")),
)
print(result)
```

For tracing and evaluation dashboards, `mlflow.langchain.autolog()` plus `mlflow.genai.evaluate` works with the same
`answer` function; see [Evaluation Framework](../../genai/patterns/evaluation-framework.md).

## Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `chunk_size` | 1000 | Characters per chunk (`RecursiveCharacterTextSplitter` counts characters by default) |
| `chunk_overlap` | 200 | Overlap between adjacent chunks |
| `search_type` | `mmr` | `similarity`, `mmr`, or `similarity_score_threshold` |
| `k` / `fetch_k` | 4 / 20 | Returned documents / MMR candidate pool |
| `persist_directory` | `data/chroma_db` | Chroma on-disk storage |
| Model | `claude-sonnet-5-5` | Any chat model via `init_chat_model`; omit `temperature` for current Claude models |

## Example Usage

```python
# Add one more file to an existing store
new_chunks = splitter.split_documents(PyPDFLoader("report.pdf").load())
vectorstore.add_documents(new_chunks)

print(answer("Who approves retention exceptions?")["answer"])
```

## See Also

- [RAG Pipelines Concept](../concepts/rag-pipelines.md)
- [Embedding Pipelines](../concepts/embedding-pipelines.md)
- [Vector DB Operations](vector-db-operations.md)
- [Databricks-native RAG](databricks-native-rag.md)

## Sources

- https://docs.langchain.com/oss/python/releases/langchain-v1
- https://docs.langchain.com/oss/python/migrate/langchain-v1
- https://docs.langchain.com/oss/python/langchain/retrieval
- https://docs.ragas.io/en/stable/getstarted/rag_eval/
- https://pypi.org/project/langchain/ ; https://pypi.org/project/langchain-chroma/ ; https://pypi.org/project/langchain-text-splitters/
