# GadgetHub RAG Pipeline

A retrieval-augmented generation pipeline for a gadget-store product assistant — cameras, laptops, smart TVs, and smartwatches. Built around a hard anti-hallucination contract: every answer must trace back to a retrieved product chunk, nothing is inferred or guessed.

## What this does

- Ingests catalog PDFs (`data/*.pdf`) and splits them so **each chunk is exactly one product's full profile** — chunk size is tuned above the longest staged product (~850 chars) so a chunk never gets cut mid-product, which is what causes hallucinated specs.
- Extracts structured metadata per product (brand, name, price, and category-specific fields for each of the four categories) using a strict extraction prompt that forbids inference: missing fields become `null`, values are copied character-for-character, nothing is "corrected" or paraphrased.
- Embeds and indexes the staged products in Qdrant using hybrid (dense + sparse) retrieval.
- Answers user queries by retrieving the relevant product chunks and generating a short, non-verbose, no-hallucination response — catching user intent (a demand like "smart watch", a problem to solve, or a hint about a product) and suggesting matching products from the vector store.

## Catalog coverage

| Category | Source PDF | Staged products |
|---|---|---|
| Camera | `rag_camera_catalog_35_models.pdf` | 35 |
| Laptop | `rag_electronic_shop_laptop_catalog_global_usd.pdf` | 68 |
| Smart TV | `rag_electronic_shop_smart_tv_catalog_35_pages.pdf` | 23 |
| Smartwatch | `rag_smartwatch_catalog_30_models.pdf` | 30 |

## Stack

- **RAGWire** — RAG orchestration (loading, splitting, metadata extraction, hybrid retrieval)
- **LangChain / LangGraph** — agent and pipeline composition
- **OpenAI** — `text-embedding-3-small` for embeddings, `gpt-4o-mini` for generation
- **Qdrant** — vector store, hybrid dense/sparse search
- **Jupyter** — `vector_embedding/rag_pipeline.ipynb` is the main ingestion/build notebook

## Project layout

```
data/                          Source catalog PDFs + staged per-product markdown files
vector_embedding/
  rag_pipeline.ipynb           Main notebook: load → split → extract metadata → embed → index
  rag_pipeline.yaml            Pipeline config (splitter, embeddings, llm, vectorstore, retriever)
  rag_metadata.yaml            Metadata extraction schema + prompt (no-hallucination rules)
  ingest.md                    How to add new catalog PDFs or a whole new product category
```

## Setup

1. Copy `.env.example` to `.env` and fill in your own keys:
   - `OPENAI_API_KEY` — embeddings + generation
   - `QDRANT_URL`, `QDRANT_API_KEY` — vector store
2. Install dependencies (see `requirements.txt` / `pyproject.toml`).
3. Open `vector_embedding/rag_pipeline.ipynb` and run it top to bottom to ingest the catalogs and build the index.
4. To add more products or a new category, see `vector_embedding/ingest.md`.

## Design principles

- **No hallucination**: extraction and retrieval are both constrained to copy exact values from source text; nulls are preferred over guesses.
- **Guardrails over verbosity**: responses stay short and on-topic rather than chatty.
- **One product per chunk**: the retriever should never need to stitch partial context back together.
