# Usage Guide

How to clone this repo, set it up, and run the GadgetHub RAG pipeline yourself.

## 1. Prerequisites

- Python 3.13+
- An [OpenAI API key](https://platform.openai.com/api-keys) (embeddings + generation)
- A [Qdrant](https://qdrant.tech/) instance — either [Qdrant Cloud](https://cloud.qdrant.io/) (free tier works) or a local Docker instance:
  ```bash
  docker run -p 6333:6333 qdrant/qdrant
  ```

## 2. Clone and install

```bash
git clone https://github.com/<your-username>/data-ingestion.git
cd data-ingestion
```

Install dependencies with `uv` (recommended, matches `uv.lock`):

```bash
uv sync
```

Or with plain pip:

```bash
pip install -r requirements.txt
```

## 3. Configure environment variables

Copy the example file and fill in your own keys:

```bash
cp .env.example .env
```

Edit `.env`:

| Variable | Required | Notes |
|---|---|---|
| `OPENAI_API_KEY` | yes | embeddings (`text-embedding-3-small`) + generation (`gpt-4o-mini`) |
| `QDRANT_URL` | yes | e.g. `http://localhost:6333` or your Qdrant Cloud URL |
| `QDRANT_API_KEY` | only for Qdrant Cloud | leave empty for local Docker |
| `AGENT` | no | optional agent-selection flag, leave empty if unused |
| `LANGSMITH_*`, `LANGCHAIN_TRACING_V2` | no | optional LangSmith tracing |
| `CREWAI_TRACING_ENABLED` | no | optional CrewAI tracing |
| `APP_USER`, `APP_PASSWORD`, `CHAINLIT_AUTH_SECRET` | no | only needed if you build a chat frontend on top |

Never commit your real `.env` — it's already in `.gitignore`.

## 4. Run the pipeline

Open the notebook:

```bash
jupyter notebook vector_embedding/rag_pipeline.ipynb
```

Run the cells top to bottom:

1. **Setup** — loads `.env` and initializes RAGWire from `rag_pipeline.yaml`.
2. **Prepare the catalog** — stages each PDF in `data/*.pdf` into one markdown file per product (output goes to `data/_staged_products/`), so every chunk is exactly one product's full profile.
3. **Ingest** — embeds the staged products and indexes them into Qdrant (hybrid dense + sparse).
4. **Sanity-check retrieval** — run a plain semantic search and a category-filtered search to confirm indexing worked.
5. **Guardrailed recommendation agent** — the retrieval + generation logic that catches user demand/problem/hint and recommends products without hallucinating.
6. **Interactive Q&A** — chat loop to try it out directly in the notebook.

## 5. Adjust retrieval / generation behavior

Edit `vector_embedding/rag_pipeline.yaml`:

- `splitter.chunk_size` — keep this above your longest single staged product's character count, or a chunk can get cut mid-product.
- `embeddings.model` / `llm.model` — swap OpenAI models here.
- `vectorstore.collection_name` — change if you want a separate Qdrant collection per environment.
- `retriever.top_k` / `retriever.search_type` — tune how many chunks are retrieved and whether hybrid search is used.

Edit `vector_embedding/rag_metadata.yaml` to change what structured fields are extracted per product category (camera / laptop / smart_tv / smartwatch) and the no-hallucination extraction rules.

## 6. Add your own product catalogs

See `vector_embedding/ingest.md` for the full walkthrough — covers both:

- **A.** Adding more PDFs to an existing category (e.g. another smartwatch catalog).
- **B.** Adding a brand new category (e.g. headphones, tablets) — write a new splitter, extend the metadata schema, register it.

## 7. Troubleshooting

- **Retrieval returns nothing / wrong products** — re-check `data/_staged_products/` was generated; inspect one staged `.md` file to confirm the PDF parsed correctly.
- **Metadata fields are all null** — the PDF's table layout probably doesn't match what the splitter expects; see `ingest.md` step on inspecting raw extracted text before writing a splitter.
- **Qdrant connection errors** — confirm `QDRANT_URL` is reachable and, if using Qdrant Cloud, that `QDRANT_API_KEY` is set.
- **OpenAI errors** — check your API key has credit and access to the configured models.

## License

No license file is included yet — add one (e.g. MIT) if you want to make reuse terms explicit for others cloning this repo.
