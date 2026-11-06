# 🕸️ Napoleon GraphRAG — Multi-Hop Question Answering with LightRAG, Neo4j & FAISS

A Graph-based Retrieval-Augmented Generation (GraphRAG) pipeline that turns a long-form biography of **Napoleon Bonaparte** into a **knowledge graph + vector index**, then answers complex, multi-hop historical questions over it.

Built with [LightRAG](https://github.com/HKUDS/LightRAG), **Neo4j** (graph storage), **FAISS** (vector storage) and **OpenAI** (`gpt-4o-mini` + embeddings), running in Google Colab.

---

## ✨ Highlights

- **Hybrid retrieval**: combines graph traversal (entities and relations) with vector similarity search over text chunks (`mix` mode).
- **Automatic knowledge-graph construction**: the LLM extracts entities and relationships from raw text, no manual schema.
- **Multi-hop reasoning**: successfully answered questions requiring up to **7 chained reasoning hops** (see [Evaluation](#-evaluation)).
- **Persistent storage**: Neo4j holds the graph, FAISS indexes persist to `./rag_storage`, so re-runs skip re-indexing.
- **Fully async pipeline**: ingestion and querying built on `asyncio`.

---

## 🧠 How It Works

```
 data.txt (Napoleon biography)
        │
        ▼
  Chunking (500 tokens, 100 overlap)
        │
        ▼
  LLM entity & relation extraction (gpt-4o-mini)
        │
        ├──────────────► Neo4j  (knowledge graph)
        │
        └──────────────► FAISS  (entity / relationship / chunk vectors)

 Question ─► keyword extraction (high-level + low-level)
        │
        ├─ local search  : matching entities + their relations
        ├─ global search : matching relationships
        └─ naive search  : top vector chunks
                │
                ▼
        Merged context ─► gpt-4o-mini ─► Answer + references
```

---

## 📊 The Resulting Knowledge Graph

Indexing the text produced **47 chunks**, **487 nodes** and **388 relationships** in Neo4j.

![Knowledge graph](docs/graph.png)
<!-- Save your Neo4j graph screenshot as docs/graph.png -->

| Entity type    | Count |
|----------------|------:|
| person         | 126 |
| location       | 98 |
| event          | 85 |
| organization   | 59 |
| concept        | 53 |
| data           | 17 |
| artifact / content | 8 / 8 |
| date           | 6 |
| group / naturalobject / legal_code / family / other / document | 3 / 3 / 2 / 2 / 2 / 2 |
| agreement / creature | 1 / 1 |
| UNKNOWN        | 11 |

The graph forms a few dense hubs (around Napoleon himself, his campaigns, and his family and political circle), with many isolated peripheral entities.

---

## 🧰 Tech Stack

| Component        | Choice |
|------------------|--------|
| RAG framework    | `lightrag-hku` |
| Graph database   | Neo4j (AuraDB) via `Neo4JStorage` |
| Vector store     | FAISS (`FaissVectorDBStorage`) |
| LLM              | OpenAI `gpt-4o-mini` |
| Embeddings       | OpenAI `openai_embed` |
| Runtime          | Google Colab, Python 3, `nest_asyncio` |

**Chunking config:** `chunk_token_size=500`, `chunk_overlap_token_size=100`
**Retrieval config:** `mode="mix"`, `top_k=5`

---

## 🚀 Getting Started

### 1. Install dependencies

```bash
pip install lightrag-hku neo4j openai faiss-cpu nest_asyncio python-dotenv
```

### 2. Configure credentials

Set these as environment variables (in Colab, use **Secrets**, never hard-code them):

```python
import os
os.environ["OPENAI_API_KEY"]   = "<your-openai-key>"
os.environ["NEO4J_URI"]        = "neo4j+s://<your-instance>.databases.neo4j.io"
os.environ["NEO4J_USERNAME"]   = "<username>"
os.environ["NEO4J_PASSWORD"]   = "<password>"
```

You can create a free Neo4j AuraDB instance at <https://neo4j.com/cloud/platform/aura-graph-database/>.

### 3. Add your data

Place your source text at `rag_storage/data.txt` (or pass any path to `main`).

### 4. Run

```python
import asyncio

asyncio.run(main(
    question="Who killed Napoleon?",
    mode="mix",
    data_path="/content/rag_storage/data.txt",
))
```

On the first run, the file is chunked, entities and relations are extracted, and everything is written to Neo4j and FAISS. Subsequent runs detect the duplicate document and go straight to querying.

---

## 🔌 Core Functions

| Function | Purpose |
|----------|---------|
| `initialize_rag()` | Builds the `LightRAG` instance with Neo4j and FAISS backends, initializes storages and shared pipeline status. |
| `index_data(rag, file_path)` | Reads a text file and calls `rag.ainsert()` to chunk, embed, extract and store. |
| `run_async_query(rag, question, mode, top_k)` | Runs `rag.aquery()` with the chosen retrieval mode. |
| `main(question, mode, data_path)` | End-to-end: initialize, index, query, print. |

**Supported query modes** in LightRAG include `naive`, `local`, `global`, `hybrid` and `mix`. This project uses `mix`, which blends graph and vector retrieval.

---

## 🧪 Evaluation

The system was tested with questions of increasing reasoning depth.

| # | Question (abridged) | Hops | Result |
|---|---------------------|:----:|:------:|
| 1 | Who killed Napoleon? | 1 | ✅ Correct: stomach cancer, no proof of foul play |
| 2 | Napoleon's family | 1 | ✅ |
| 3 | Disease of autopsy and the family member who shared it | 2 | ✅ Stomach cancer, his father |
| 4 | Years between death and entombment at Les Invalides | 2 | ✅ 1821 to 1861 = 40 years |
| 5 | Successor named at 1815 abdication and his fate | 2 | ✅ Napoleon II, died of tuberculosis, no heirs |
| 6 | Challenges to the arsenic-poisoning theory | 2 | ✅ Arsenic in other hair samples, 19th-century medicinal use |
| 7 | Remains returned in 1840 and the rulers involved | 3 | ✅ Louis-Philippe and Napoleon III |
| 8 | Invasion of Spain to power vacuum to independence leaders | 3 | ✅ Bolívar and San Martín |
| 9 | Abdication, successor, legend, tomb designer, reign | 5 | ✅ Napoleon III, Louis Visconti, 1861 |
| 10 | Full chain from abdication to current location of remains | 6 | ✅ |
| 11 | Spain, vacuum, leaders, ideology, country that uniquely honors Napoleon | 7 | ✅ Traced the full chain |
| 12 | German unification chain with year calculation and second nation | 8 | ❌ Partially incorrect at 8 hops |
| 13 | Out-of-scope question about Nepal's borders | n/a | ✅ Correctly refused: *"I do not have enough information"* |

**Takeaways**
- Reliable up to ~7 hops of chained reasoning.
- Gracefully declines when the answer is not in the knowledge base rather than hallucinating.
- The 8-hop question exposed the limit: errors compound as context and retrieval span more distant parts of the graph.

---

## ⚠️ Limitations & Ideas for Improvement

- **Single-document corpus**: one biography; answers are only as good as the source.
- **No reranker**: LightRAG warns that rerank is enabled but no rerank model is configured. Adding one (e.g. Cohere or a BGE reranker) should improve context quality for deep questions.
- **Entity extraction noise**: 11 `UNKNOWN`-typed nodes and many disconnected entities. Better prompts, a stronger extraction model, or entity-merging passes would tighten the graph.
- **Small `top_k` (5)**: raising it, or using `chunk_top_k`, may help very long reasoning chains.
- **Stronger LLM**: swapping `gpt-4o-mini` for a larger model may push the reliable hop count higher.
- **Evaluation**: add a labeled QA set and automatic metrics (e.g. RAGAS) instead of manual inspection.

---

## 🔐 Security Note

Never commit API keys or database passwords. Load them from environment variables or Colab Secrets. If credentials were ever committed or shared, **rotate them immediately**.

---

## 📁 Project Structure

```
.
├── LightRAG_Napoleon.ipynb   # Main notebook
├── rag_storage/
│   ├── data.txt              # Source text
│   ├── faiss_index_*.index   # Persisted FAISS indexes
│   └── ...                   # LightRAG caches and KV stores
├── docs/
│   └── graph.png             # Neo4j graph screenshot
└── README.md
```

---

## 🙏 Acknowledgements

- [LightRAG](https://github.com/HKUDS/LightRAG) by HKUDS
- [Neo4j](https://neo4j.com/) and [FAISS](https://github.com/facebookresearch/faiss)
- OpenAI for the LLM and embedding models
