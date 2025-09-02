# PanamaPapers-LLM-KnowledgeGraph — Extraction, Visualization & Querying

> End‑to‑end notebooks that build a Property Graph from unstructured text (Panama Papers) using **LlamaIndex** and query it with **embedding + LLM retrievers**, with interactive graph visualizations via **PyVis**.


## ✨ What’s inside
- **4 extraction strategies** with LlamaIndex property‑graph extractors: _Schema‑based_, _Free‑form_, _Dynamic LLM_, and _Implicit relations_.
- **Stored indexes** for each strategy so you can reload without recomputation.
- **Interactive HTML graph views** (PyVis / NetworkX export).
- **Two concise notebooks**: one to **build** graphs, one to **query** them.


## 📦 Repository structure
```
PanamaPapers-LLM-KnowledgeGraph/
  README.md
  config.py
  requirements.txt
  data/
    panama_papers/
      panama_papers.pdf
  notebooks/
    01_kg_extraction.ipynb
    02_kg_querying.ipynb
  outputs/
    kg_dynamic_llm.html
    kg_free_form.html
    kg_implicit.html
    kg_schema_llm.html
```

- **data/**: raw input documents (demo uses the Wikipedia article on the Panama Papers).  
- **notebooks/**: Jupyter workflows for extraction and querying.  
- **outputs/**: exported, interactive graph visualizations per strategy.  
- **storage/** (created on first run): persisted LlamaIndex storage per extractor.


## 🚀 Quickstart
### 1) Environment
- **Python**: 3.9–3.11 recommended
- Install deps:
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```
> `requirements.txt` pins: `pyvis==0.3.2`, `llama_index==0.10.65`.

### 2) Configure OpenAI
This repo reads your key from `config.py` and pushes it to the environment in the notebooks.

- Option A — Edit `config.py`:
```python
OPENAI_API_KEY = "sk-..."
```
- Option B — Use environment variable (preferred):
```bash
export OPENAI_API_KEY=sk-...
# (Optionally leave config.py empty)
```

> Models used in notebooks (editable):
> - `LLM_MODEL = "gpt-4o-mini"`
> - `EMBEDDING_MODEL = "text-embedding-3-small"`


## 🏗️ 01_kg_extraction.ipynb — Build the graph
Open the notebook and run cells top‑to‑bottom. Key parameters:
```python
path_input_text = "data/panama_papers"   # directory with PDFs / text
path_output_storage = "storage"          # per‑extractor storage
path_output = "outputs"                  # exported HTML graphs
```

### Extraction strategies
Each section creates/persists an index and exports an interactive graph (HTML) named by the extractor id.

1. **Schema‑based extraction** — `SchemaLLMPathExtractor`
   - Define a **validation schema** (node/edge types + properties) to constrain triples.
   - Good when you know your domain entities/relations.
   - Exports: `outputs/kg_schema_llm.html` and storage under `storage/schema_llm/` (or similar).

2. **Free‑form extraction** — `SimpleLLMPathExtractor`
   - Let the LLM propose entities/relations directly from text.
   - Fast to prototype; may be noisy.
   - Exports: `outputs/kg_free_form.html` + storage.

3. **Dynamic LLM extraction** — `DynamicLLMPathExtractor`
   - LLM adapts paths it extracts based on context; balances structure and recall.
   - Used as the default **querying** graph in the second notebook.
   - Exports: `outputs/kg_dynamic_llm.html` + storage `storage/dynamic_llm/`.

4. **Implicit relation extraction** — `ImplicitPathExtractor`
   - Surfaces relations that are implied rather than explicit in text.
   - Exports: `outputs/kg_implicit.html` + storage.

#### What gets saved
Every strategy:
```python
# persist index storage for reuse
index.storage_context.persist(persist_dir=f"{path_output_storage}/{extractor_name}/")

# export a PyVis / NetworkX HTML for quick inspection
index.property_graph_store.save_networkx_graph(name=f"{path_output}/kg_{extractor_name}.html")
```

> Tip: open the HTML exports directly in your browser to explore nodes/edges interactively.


## 🔎 02_kg_querying.ipynb — Ask the graph
Point to the storage you want to query (defaults to dynamic LLM):
```python
path_storage = "storage/dynamic_llm"
LLM_MODEL = "gpt-4o-mini"
EMBEDDING_MODEL = "text-embedding-3-small"
TEMPERATURE = 0.1
```

### Load index
```python
from llama_index.core import StorageContext, load_index_from_storage
index = load_index_from_storage(StorageContext.from_defaults(persist_dir=path_storage))
```

### Available retrievers (used in this repo)
1. **VectorContextRetriever** — embedding search over graph‑aware chunks/paths.
2. **LLMSynonymRetriever** — augments retrieval with synonym expansion via LLM.

Minimal pattern (excerpt):
```python
# build a sub‑retriever (e.g., VectorContextRetriever) with your models
# ... configure include_text=True, max_keywords, path_depth, etc.

retriever = index.as_retriever(sub_retrievers=[sub_retriever])
query_engine = index.as_query_engine(sub_retrievers=[retriever])

print(query_engine.query(
    "Who were the main people involved in the Panama Papers scandal?"
).response)
```

> The notebook also demonstrates a question on ICIJ involvement.


## 🧪 Changing the dataset
Drop your own PDFs / `.txt` files under `data/your_corpus/` and point `path_input_text` to that folder. The `SimpleDirectoryReader` in LlamaIndex will ingest PDFs and text out of the box.


## ⚙️ Tuning & customization
- **Schema design**: in the schema‑based section, enumerate node types (e.g., `Person`, `Organization`, `Location`) and allowed relations with validation rules.
- **Models**: swap `LLM_MODEL` / `EMBEDDING_MODEL` to your preferred OpenAI (or other provider if you adapt the code).
- **Depth & keywords**: in retrievers, adjust `path_depth`, `max_keywords`, and whether to `include_text`.
- **Storage layout**: change `persist_dir` names to keep multiple runs side‑by‑side.


## 📈 Outputs you should see
- `outputs/kg_schema_llm.html`
- `outputs/kg_free_form.html`
- `outputs/kg_dynamic_llm.html`
- `outputs/kg_implicit.html`

Open any of these in a browser to inspect nodes/edges, hover for details, and zoom/pan.


## 🧰 Troubleshooting
- **`openai.AuthenticationError` or empty results**: ensure `OPENAI_API_KEY` is set and has access to the specified models.
- **`ModuleNotFoundError: llama_index`**: `pip install -r requirements.txt` inside an active virtualenv.
- **Graphs don’t render**: some browsers block local JS; use a lightweight server:
  ```bash
  python -m http.server 8000
  # then open http://localhost:8000/outputs/kg_dynamic_llm.html
  ```
- **Slow/expensive runs**: use smaller models, limit pages, or sample files in `data/`.


## 🗺️ Roadmap ideas (optional)
- Add a **Neo4j / Memgraph** sink and Cypher querying.
- Try **KnowledgeGraphRAGRetriever** / **PGQueryEngine** in LlamaIndex for hybrid graph + text RAG.
- Add **evaluation** (precision/recall of extracted triples) against a small hand‑labeled set.
- Package the notebook logic into a reusable Python module + CLI.


## 🙌 Acknowledgements
Built with [LlamaIndex](https://github.com/run-llama/llama_index) property‑graph tooling and OpenAI models; visualized with PyVis/NetworkX.


## 📄 License
MIT License

Copyright (c) 2025 Siddharth Pal
---

### Badges (copy/paste if you want)
```
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)]()
[![LlamaIndex](https://img.shields.io/badge/LlamaIndex-0.10.65-9cf)]()
[![PyVis](https://img.shields.io/badge/PyVis-0.3.2-lightgrey)]()
```

