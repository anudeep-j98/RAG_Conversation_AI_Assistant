# Papeer — Research Paper Assistant

A conversational AI assistant for students and researchers to upload, explore, and verify academic papers through natural language chat.

---

## Project Description

Papeer is a Retrieval-Augmented Generation (RAG) application built with LangGraph, LangChain, and Streamlit. Users upload research papers (PDF, TXT, Markdown, web URL, or ArXiv ID) into isolated sessions, then ask questions about them. The system routes each query intelligently and answering directly from paper content, searching the web for current developments, or verifying whether a claim from a paper has been superseded by newer research.

---



## Target Users

- **Students** reading and trying to understand dense academic papers
- **Researchers** who want to quickly cross-reference claims across multiple papers
- **Literature reviewers** checking whether findings or methods from older papers still hold today
- **Anyone** who wants a conversational interface to a set of documents without manual reading

---



## Features


| Feature                    | Description                                                                                                                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Paper Q&A**              | Ask questions about uploaded papers; the system retrieves relevant chunks and generates grounded answers                                                                        |
| **Claim Verification**     | Ask the assistant to verify a claim — it searches the web and ArXiv to determine if the claim is current or superseded, and returns links to newer papers if applicable         |
| **Web Search**             | For questions about current developments or explicit search requests, live Tavily results are incorporated                                                                      |
| **Direct Answers**         | General knowledge questions are answered without retrieval or web calls                                                                                                         |
| `**/btw` Command**         | A side-channel for off-topic questions outside the session context. The LLM decides to answer directly or search the web. These exchanges are **not stored in session history** |
| **Multi-session UI**       | Open multiple independent sessions simultaneously, each with its own paper collection and conversation history                                                                  |
| **Auto Session Naming**    | Session titles are automatically generated (3–5 words) from the first message using the LLM                                                                                     |
| **Multiple Paper Sources** | Load papers via file upload (PDF, TXT, MD), web URL, or ArXiv ID/title search                                                                                                   |
| **Graph State Inspector**  | Each assistant turn exposes an expandable JSON view of the LangGraph state for debugging                                                                                        |
| **Streaming Responses**    | Assistant responses stream token-by-token with a cursor animation                                                                                                               |


---



## How to Use



### 1. Start a session

Launch the app and a default session is created automatically. Use **New Chat** in the sidebar to start additional sessions.

### 2. Upload papers

In the sidebar, choose one of three loading methods:

- **File Upload** — drag and drop a PDF, TXT, or MD file
- **Web URL** — paste one or more URLs (one per line)
- **ArXiv** — enter a paper title or ArXiv ID (e.g. `2303.08774`)

Loaded papers are listed under "Loaded Papers" in the sidebar.

### 3. Ask questions

Type in the chat input. Example queries:

- *"What methodology does the paper use for evaluation?"*
- *"Verify the claim that encoder-decoder models are the best approach for translation."*
- *"What are the latest developments in diffusion models?"*



### 4. Use `/btw` for off-topic questions

Prefix any message with `/btw` to ask a question outside the current paper context. These exchanges are not saved to the session:

```
/btw What is the difference between RLHF and DPO?
```

---



## Installation

Papeer uses [uv](https://github.com/astral-sh/uv) for dependency management.

```bash
# Clone the repository
git clone <repo-url>
cd rag-papeer-project

# Install all dependencies
uv sync

# Copy the example env file and fill in your keys
cp .env.example .env

# Run the Streamlit app
uv run streamlit run app.py
```

To add a new dependency:

```bash
uv add <package-name>
```

To run a backend module directly (useful during development):

```bash
uv run python -m backend.<module_name>
```

---



## Required API Keys

All keys are loaded from a `.env` file in the project root via `python-dotenv`.


| Variable         | Purpose                                                                | Where to Get It                                    |
| ---------------- | ---------------------------------------------------------------------- | -------------------------------------------------- |
| `OPENAI_API_KEY` | LLM inference (`gpt-5-mini`) and embeddings (`text-embedding-3-small`) | [platform.openai.com](https://platform.openai.com) |
| `TAVILY_API_KEY` | Web search for current developments and claim verification             | [tavily.com](https://tavily.com)                   |
| `QDRANT_URL`     | Qdrant Cloud endpoint for the vector store                             | [cloud.qdrant.io](https://cloud.qdrant.io)         |
| `QDRANT_API_KEY` | Authentication for Qdrant Cloud                                        | [cloud.qdrant.io](https://cloud.qdrant.io)         |


`.env` file format:

```env
OPENAI_API_KEY=sk-...
TAVILY_API_KEY=tvly-...
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key
```

---



## Architecture

```
app.py (Streamlit UI)
│
├── backend/rag_graph.py       — LangGraph RAG workflow (router → retrieve/verify/direct → answer)
├── backend/btw_handler.py     — Off-topic /btw handler (streaming, not stored in history)
├── backend/vector_store.py    — Qdrant Cloud vector store with cached embeddings
├── backend/paper_loader.py    — Multi-source paper loader (PDF, TXT, MD, URL, ArXiv)
└── backend/models.py          — Pydantic models for routing and structured LLM outputs
```



### RAG Graph Decision Flow

```
User Query
    │
    ▼
 Router (LLM)
    │
    ├── direct_answer ──────────────────────────► Generate Answer
    │
    ├── retrieve ──► Agent (retriever + web tools) ──► Relevancy Check
    │                        │                              │
    │                        │◄── Query Rewrite (max 3) ────┘
    │                        └──────────────────────────────► Generate Answer
    │
    └── verify_claim ──► Web Search + ArXiv Search ──► Verdict + Paper Links
```

---



## How the Project Is Production Optimized


| Optimization            | Details                                                                                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Embedding cache**     | `CacheBackedEmbeddings` writes to `./embedding_cache/` so identical text is never re-embedded across sessions — reduces OpenAI API calls and latency          |
| **Session isolation**   | Each session gets its own Qdrant collection (`papeer_{session_id}`) and a separate LangGraph SQLite checkpointer thread — prevents cross-session data leakage |
| **Graph caching**       | The LangGraph graph is built once with `@st.cache_resource` and reused across all Streamlit reruns                                                            |
| **Streaming responses** | `graph.stream()` is used with message mode so responses appear token-by-token rather than waiting for the full generation                                     |
| **Session persistence** | `sessions.json` persists session metadata; SQLite stores full conversation state — app restarts restore the previous session seamlessly                       |
| **Temp file cleanup**   | Uploaded files are written to a temp path, processed, then deleted regardless of success or failure                                                           |
| **Async evaluation**    | The evaluation pipeline uses throttled concurrency (3 workers, 5 s throttle) to stay within API rate limits                                                   |
| **ArXiv reliability**   | Claim verification uses two targeted Tavily searches (general web + `site:arxiv.org`) instead of the `arxiv` Python library, which had reliability issues     |


---



## Constraints and Why


| Constraint                               | Why                                                                                                                                                                                                             |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Max 3 query rewrites**                 | The RAG graph caps query rewrites at 3 retries before falling back to a plain LLM answer. Without this cap, ambiguous or unanswerable queries would loop indefinitely, burning API tokens and blocking the user |
| **Chunk size 1000 / overlap 200**        | Balances retrieval precision (smaller = more focused) against context preservation across chunk boundaries. The 200-char overlap ensures sentences split across chunks are still retrievable                    |
| **Tavily max 3 results for** `/btw`      | Keeps the context window manageable for side-channel queries that are intentionally lightweight and unsaved                                                                                                     |
| `**/btw` exchanges not stored**          | These are deliberately out-of-context questions. Storing them would pollute session history and confuse the LLM's understanding of the paper-focused conversation                                               |
| **Session-scoped Qdrant collections**    | Prevents papers from one session leaking into another. Each collection is namespaced by session UUID                                                                                                            |
| **Claim verification uses two searches** | A general web search catches blog posts and news; an `arxiv.org`-targeted search catches academic superseding work. One search alone misses one of these two important source types                             |
| `**k=4` default retrieval chunks**       | Balances context richness against prompt length. Too few chunks miss relevant content; too many dilute focus and increase cost                                                                                  |


---



## Evaluation

Papeer includes an automated RAG evaluation pipeline (`evaluate.py`) built on [DeepEval](https://github.com/confident-ai/deepeval).

### Metrics (threshold: 0.7)


| Metric                   | What It Measures                                                    |
| ------------------------ | ------------------------------------------------------------------- |
| **Contextual Precision** | Are the retrieved chunks relevant to the query?                     |
| **Contextual Recall**    | Does the retrieved context cover all expected information?          |
| **Contextual Relevancy** | Is the context relevant to both the input and the expected output?  |
| **Answer Relevancy**     | Does the generated answer actually address the question?            |
| **Faithfulness**         | Is the answer grounded in the retrieved context (no hallucination)? |

### Running Evaluation

```bash
uv run python evaluate.py
```

- On first run, synthetic golden test cases are generated from `documents/Openclaw_Research_Report.pdf` and cached to `goldens.json`
- Results are written to `eval_results.json` with per-test metric scores, pass/fail status, and failure reasons
- Subsequent runs reuse cached goldens unless `goldens.json` is deleted

#### Output of evaluate.py
```
| **Metric**               | **Average Score**  | **Pass Rate** | **Total Samples**|
| -----------------------  | ------------------ | ------------- | ---------------- |
| 🎯 Contextual Precision |          **0.94** |    **90.00%** |            **10** |
| 🔄 Contextual Recall    |          **0.96** |    **90.00%** |            **10** |
| 📚 Contextual Relevancy |          **0.56** |    **30.00%** |            **10** |
| ✅ Answer Relevancy     |          **0.99** |   **100.00%** |            **10** |
| 🛡️ Faithfulness         |          **0.99** |   **100.00%** |            **10** |    
```


---


## Project File Reference

This section describes every file and folder in the repository and what each one does.

### Root — Application Entry Points


| File              | Purpose                                                                                                                                                                                                                                                          |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `**app.py**`      | Main Streamlit application. Handles the UI (sidebar sessions, document upload, chat), wires user input into the LangGraph RAG pipeline, streams assistant responses, and manages session metadata in `sessions.json`. This is the file you run to start the app. |
| `**evaluate.py**` | Automated RAG quality evaluation using DeepEval. Generates or loads golden test cases, runs the RAG graph against them, scores answers on five metrics, and writes results to `eval_results.json`.                                                               |
| `**main.py**`     | Minimal placeholder script (`Hello from rag-papeer-project!`). Not used by the app; the real entry point is `app.py`.                                                                                                                                            |


### Root — Runtime & Generated Data


| File                      | Purpose                                                                                                                                                                                        |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `**sessions.json**`       | Persists session metadata (UUID, display name, creation timestamp, whether auto-named). Survives app restarts so chat sessions reappear in the sidebar.                                        |
| `**goldens.json**`        | Cached synthetic Q&A test pairs generated by DeepEval's `Synthesizer` from the sample PDF. Reused on subsequent evaluation runs unless deleted.                                                |
| `**eval_results.json**`   | Output of the last `evaluate.py` run — per-test metric scores, pass/fail status, and failure reasons. Created after evaluation completes.                                                      |
| `**graph.png**`           | Visual diagram of the LangGraph workflow (for documentation or debugging).                                                                                                                     |




### `backend/` — Core Logic


| File                          | Purpose                                                                                                                                                                                                                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `**backend/rag_graph.py**`    | The heart of the RAG system. Defines `RAGState`, builds the LangGraph workflow (router → agent/verify/direct → answer), implements retrieval tools (vector store + Tavily web search), relevancy checking, query rewriting, and claim verification. Exports `build_graph()`. |
| `**backend/vector_store.py**` | Qdrant Cloud integration. Creates session-scoped collections (`papeer_{session_id}`), embeds documents with cached `text-embedding-3-small`, and exposes `add_paper()`, `list_papers()`, and `search()`.                                                                     |
| `**backend/paper_loader.py**` | Document ingestion. Loads and chunks papers from PDF (PyMuPDF), TXT, Markdown, web URLs, and ArXiv (via Atom API + PDF download). Uses 1000-char chunks with 200-char overlap.                                                                                               |
| `**backend/btw_handler.py**`  | Handles `/btw` side-channel queries. Routes to direct LLM answer or Tavily web search, streams the response, and deliberately bypasses the vector store and session checkpointer.                                                                                            |
| `**backend/models.py**`       | Pydantic schemas for structured LLM outputs: `RouterDecision`, `RelevancyDecision`, `ClaimVerificationResult`, `BtwRouteDecision`, and `SupersedingPaper`.                                                                                                                   |




### `documents/` — Sample Data


| File                                         | Purpose                                                                                                                                                      |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `**documents/Openclaw_Research_Report.pdf**` | Sample research report used by the evaluation pipeline (`evaluate.py`) to generate golden test cases. You can upload your own papers through the UI instead. |

### Data Flow Summary

```
User uploads paper (app.py sidebar)
        │
        ▼
paper_loader.py  →  chunks documents
        │
        ▼
vector_store.py  →  embeds & stores in Qdrant (session-scoped)
        │
User asks question (app.py chat)
        │
        ▼
rag_graph.py     →  router → retrieve / verify / direct → generate answer
        │                    │
        │                    └── vector_store.search() + Tavily web search
        ▼
checkpoints.db   →  persists conversation state per session
```



#### AI Agent output for question -

```
{
  "messages": [
    {
      "type": "HumanMessage",
      "content": "where is FIFA 2026 held in?"
    },
    {
      "type": "AIMessage",
      "content": ""
    },
    {
      "type": "ToolMessage",
      "content": "Found 5 web result(s) for: FIFA World Cup 2026 host countries where is it held"
    },
    {
      "type": "AIMessage",
      "content": ""
    },
    {
      "type": "AIMessage",
      "content": "FIFA World Cup 2026 is being held across **North America**, co-hosted by **Canada, Mexico, and the United States**.\n\nIt will take place in **16 host cities/venues** across those three countries."
    }
  ],
  "session_id": "8a98bb99-e595-495e-acdf-acc7b8f4a39f",
  "query": "where is FIFA 2026 held in?",
  "route": "retrieve",
  "retrieved_docs": [
    {
      "content": "# Fifa World Cup 2026: your guide to all 16 host cities. The 2026 Fifa World Cup will be a unique event in the history of this iconic football tournament, with an expanded 48-team format taking place over 39 days and an entire continent – with Canada, Mexico and the USA co-hosting. Check the full ma",
      "metadata": {
        "url": "https://www.cathaypacific.com/cx/en_US/inspiration/travel/world-cup-2026-host-cities.html",
        "title": "Fifa World Cup 2026: your guide to all 16 host cities - Cathay Pacific"
      }
    },
    {
      "content": "# FIFA World Cup 2026™ Venues. The FIFA World Cup 2026™ will take place across North America, with matches hosted at 16 venues in Canada, Mexico, and the United States. Luxury suites will include amenities such as high-end catering and dedicated entrances, making for a truly unforgettable experience",
      "metadata": {
        "url": "https://fifaworldcup26.suites.fifa.com/venues",
        "title": "FIFA World Cup 2026™ Venues"
      }
    },
    {
      "content": "2026 World Cup opens with Mexico hosting South Africa\nScripps News\n669000 subscribers\n64 likes\n12782 views\n11 Jun 2026\nMexico makes history Thursday as the first country to host the men's FIFA World Cup three times, opening the 2026 tournament against South Africa at Estadio Azteca in Mexico City at",
      "metadata": {
        "url": "https://www.youtube.com/watch?v=-0OEpj813qk",
        "title": "2026 World Cup opens with Mexico hosting South Africa"
      }
    },
    {
      "content": "In 2017 a joint bid was officially unveiled by the US, Canada and Mexico to host the 2026 World Cup. Texas didn't join the bid because of their",
      "metadata": {
        "url": "https://www.reddit.com/r/imaginarymaps/comments/1u3xn9v/2026_world_cup_host_countries_cities_infographic",
        "title": "2026 World Cup host Countries & Cities - Infographic | (OC) - Reddit"
      }
    },
    {
      "content": "[Skip to main content](https://www.fifa.com/en/tournaments/mens/worldcup/canadamexicousa2026/host-cities#main-content). [](https://www.fifa.com/en/tournaments/mens/worldcup/canadamexicousa2026/host-cities?searchOverlay=1). [![Image 2: FIFA](https://digitalhub.fifa.com/transform/157d23bf-7e13-4d7b-94",
      "metadata": {
        "url": "https://www.fifa.com/en/tournaments/mens/worldcup/canadamexicousa2026/host-cities",
        "title": "Host Countries and Cities - FIFA"
      }
    }
  ],
  "retrieval_attempts": 1,
  "claim_verdict": null,
  "claim_source": null,
  "superseding_papers": [],
  "answer": "FIFA World Cup 2026 is being held across **North America**, co-hosted by **Canada, Mexico, and the United States**.\n\nIt will take place in **16 host cities/venues** across those three countries.",
  "is_relevant": true,
  "rewrite_count": 0
}
```



#### Below Attached the screenshot of streamlit application
#   R A G _ C o n v e r s a t i o n _ A I _ A s s i s t a n t  
 