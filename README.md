# Multi-Modal RAG Document Chatbot

## Project overview

A web-based document assistant that indexes files and wiki pages, retrieves relevant passages, and answers questions with source citations. The application combines a React chat interface with a FastAPI backend and a per-conversation retrieval-augmented generation (RAG) pipeline.

## Features

- **Multi-modal document ingestion:** Upload PDF, PNG, JPG/JPEG, BMP, TIFF, CSV, and Excel (`.xls`/`.xlsx`) files. The pipeline extracts PDF text and embedded images, and uses a vision model to describe image content.
- **Wiki ingestion:** Index an Azure DevOps Wiki and its child pages; an optional personal access token (PAT) can be supplied for private wiki access.
- **Grounded chat:** Ask questions about indexed content and receive streamed answers with source details such as file/page, wiki title, URL, and excerpt.
- **Hybrid retrieval:** Combine semantic search (FAISS) with BM25 keyword search to find relevant content.
- **Conversation isolation:** Each conversation has its own document index, upload directory, and chat history.
- **Document management:** List, preview, remove, or clear indexed content for a conversation.
- **Caching and persistence:** Cache embeddings and LLM responses, persist FAISS indexes, and reuse image extraction results.
- **Responsive chat UI:** Maintain conversations in browser storage, render Markdown/code, and switch between light and dark themes.
- **Optional tracing:** Integrate LangSmith tracing by configuring an API key.

## Technologies

| Area | Technologies |
|---|---|
| Frontend | React 18, Vite, JavaScript, CSS, React Markdown |
| API | Python 3.11+, FastAPI, Uvicorn, Pydantic |
| RAG orchestration | LangChain, LangChain OpenAI integration |
| Language and vision models | Groq OpenAI-compatible API |
| Embeddings | Sentence Transformers (`sentence-transformers/all-MiniLM-L6-v2`) |
| Retrieval | FAISS vector store, BM25 keyword retrieval |
| Document processing | PyMuPDF, Pillow, pandas, openpyxl |
| Optional observability | LangSmith |

## Installation and local development

### Prerequisites

- Python 3.11 or newer
- Node.js and npm
- A Groq API key

### 1. Configure the backend

From the project root, create and activate a virtual environment, then install the backend dependencies:

```powershell
cd backend
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e .
```

The embedding model runs locally. Download it once before starting the API (the backend configures Hugging Face libraries for offline use at runtime):

```powershell
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')"
```

Create a `.env` file in the project root:

```dotenv
GROQ_API_KEY=your_groq_api_key

# Optional: these are the defaults used by the backend.
# GROQ_API_BASE=https://api.groq.com/openai/v1
# GROQ_LLM_MODEL=openai/gpt-oss-120b
# GROQ_VISION_MODEL=qwen/qwen3.8-27b

# Optional: comma-separated browser origins allowed by the API.
# ALLOWED_ORIGINS=http://localhost:5173,http://127.0.0.1:5173

# Optional LangSmith tracing.
# LANGSMITH_API_KEY=your_langsmith_api_key
# LANGSMITH_PROJECT_NAME=doc_chatbot
```

Start the API from the `backend` directory so its relative upload and cache paths resolve correctly:

```powershell
uvicorn app.api:app --reload --port 8000
```

The API is available at `http://localhost:8000`; interactive OpenAPI documentation is at `http://localhost:8000/docs`.

### 2. Configure the frontend

Open another terminal at the project root:

```powershell
cd frontend
npm ci
npm run dev
```

Open `http://localhost:5173`. Vite forwards requests from `/api` to the backend at `http://127.0.0.1:8000`.

To create a production frontend build:

```powershell
npm run build
```

The generated static assets are written to `frontend/dist`. In production, configure the web server or reverse proxy to forward API requests to the deployed FastAPI service.

## Project structure

```text
Multi_Model_RAG/
├── README.md
├── .env                         # Local secrets and optional configuration
├── backend/
│   ├── app/
│   │   ├── api.py               # FastAPI routes and application lifecycle
│   │   ├── config.py            # Model, chunking, and cache settings
│   │   ├── pipeline.py          # Document loading and text chunking
│   │   ├── embeddings/          # Local cached embeddings
│   │   ├── extractors/          # Vision-model image descriptions
│   │   ├── ingestion/           # Wiki and web-page fetching
│   │   ├── loaders/             # PDF, image, CSV, and Excel loaders
│   │   ├── rag/                 # Conversational RAG chain and chat history
│   │   ├── retriever/           # FAISS and hybrid retrieval
│   │   ├── services/            # Chat streaming, indexing, resources, schemas
│   │   └── splitter/            # Recursive text chunking
│   ├── pyproject.toml
│   └── requirements.txt
└── frontend/
    ├── index.html
    ├── package.json
    ├── vite.config.js           # Development server and API proxy
    └── src/
        ├── api.js               # API client and SSE reader
        ├── App.jsx              # Conversation and application state
        ├── components/          # Chat, sidebar, input, and UI components
        └── styles/              # Application styles
```

At runtime, the backend creates session-specific uploads and cache data beneath `backend/data/` and `backend/cache/`. These are generated application data, not source files.

## API documentation

The backend API uses the paths below. The frontend development proxy exposes the same routes under `/api` (for example, `/api/chat`).

| Method | Path | Parameters / body | Response |
|---|---|---|---|
| `GET` | `/health` | — | `{"ok": true}` |
| `GET` | `/files` | Query: `session_id` | `{"files": ["..."]}` |
| `POST` | `/upload` | Multipart form: `session_id`, one or more `files` | `{"indexed_files": [...], "new_chunks": 12}` |
| `POST` | `/wiki` | JSON: `session_id`, `wiki_url`, optional `pat` | `{"indexed_files": [...], "new_chunks": 12}` |
| `DELETE` | `/file` | Query: `session_id`, `filename` | `{"ok": true, "indexed_files": [...]}` |
| `DELETE` | `/files` | Query: `session_id` | `{"ok": true}` |
| `POST` | `/reset` | JSON: `{"session_id": "..."}` | `{"ok": true}` |
| `POST` | `/chat` | JSON: `{"session_id": "...", "message": "..."}` | Server-Sent Events (SSE) |
| `GET` | `/thumbnail` | Query: `session_id`, `filename` | JPEG image bytes for supported PDFs/images |

### Chat stream

`POST /chat` returns `text/event-stream`. Each event is a JSON object in an SSE `data:` frame:

```text
data: {"type":"session","session_id":"conversation-id"}
data: {"type":"token","text":"The answer"}
data: {"type":"sources","sources":[{"source":"report.pdf","source_type":"text","title":"","url":"","page":0,"snippet":"...","truncated":false}]}
data: {"type":"done"}
```

Token frames may arrive incrementally. A `sources` frame contains retrieved source metadata and may be followed by the terminal `done` event. On generation failure, the stream emits an `error` frame with a user-facing `message`, followed by `done`. Chat requests return HTTP 400 if the session has no indexed documents.

Example request bodies:

```json
{
  "session_id": "conversation-id",
  "message": "Summarize the key points in the uploaded report."
}
```

```json
{
  "session_id": "conversation-id",
  "wiki_url": "https://dev.azure.com/organization/project/_wiki/wikis/wiki-name",
  "pat": "optional-personal-access-token"
}
```

The API also publishes interactive request and response schemas at `/docs`.

## Future enhancements

- Support additional document formats and richer table/chart extraction.
- Add authentication, authorization, upload limits, and per-user quotas.
- Persist chat history in a database and provide administrative retention controls.
- Add ingestion progress reporting and background jobs for large corpora.
- Expand automated API, retrieval-quality, and end-to-end UI test coverage.
- Add deployment examples and production observability/health metrics.