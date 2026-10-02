# Sage — AI Knowledge Assistant

Sage is a local-first, full-stack retrieval-augmented generation (RAG) application. Create a private collection, upload documents, and ask questions about their contents. Sage retrieves relevant document passages, gives them to a language model as grounding context, streams the response, and returns citations to the source passages.

This README describes the application as implemented in this repository: its components, data flow, configuration, local setup, API, development commands, and operational notes.

## Contents

- [Features](#features)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [How Sage works](#how-sage-works)
- [Repository layout](#repository-layout)
- [Requirements](#requirements)
- [Run locally with Docker](#run-locally-with-docker)
- [Environment configuration](#environment-configuration)
- [Use the application](#use-the-application)
- [API reference](#api-reference)
- [Run services without Docker](#run-services-without-docker)
- [Tests and retrieval evaluation](#tests-and-retrieval-evaluation)
- [Troubleshooting](#troubleshooting)
- [Deployment notes](#deployment-notes)
- [Security and data notes](#security-and-data-notes)
- [License](#license)

## Features

- Email and password registration and login, with signed JWT access and refresh tokens.
- User-owned knowledge collections.
- Upload PDF, DOCX, Markdown (`.md` / `.markdown`), and plain text (`.txt`) documents.
- Background document processing with visible `pending`, `processing`, `ready`, and `failed` states.
- Text extraction, overlapping word-aware chunking, vector embeddings, and PostgreSQL storage.
- Semantic vector retrieval, or hybrid vector plus PostgreSQL full-text retrieval.
- Streaming assistant responses over Server-Sent Events (SSE).
- Source citations with document name, passage snippet, and a similarity score.
- Pluggable chat and embedding providers: local Ollama, OpenAI, and Anthropic chat.
- A retrieval endpoint for inspecting search results without generating an answer.

## Technology stack

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Web client | React 18, TypeScript, Vite | Single-page application, routes, forms, upload UI, chat UI |
| UI styling | Tailwind CSS, small local UI components, Lucide icons | Layout, styling, reusable controls, icons |
| Client data | TanStack Query | API request caching, mutation updates, document-status polling |
| Client validation | React Hook Form, Zod | Login, registration, and collection form validation |
| API | Python 3.11 container, FastAPI, Uvicorn | REST API, validation, authentication, SSE chat stream |
| Persistence and ORM | SQLAlchemy 2, psycopg2 | Relational data access and PostgreSQL connection |
| Database | PostgreSQL 16 with pgvector | Users, collections, documents, chunks, vectors, conversations, messages |
| Vector indexing | pgvector cosine distance and HNSW | Approximate nearest-neighbor vector retrieval |
| Lexical search | PostgreSQL English full-text search and GIN | Keyword-oriented retrieval for hybrid search |
| Local model server | Ollama | Local embedding and chat model APIs |
| Hosted model APIs | OpenAI, Anthropic | Optional remote chat and embedding providers |
| Container runtime | Docker Compose | Local database, API, and Ollama services |

The frontend is a separate development process in the standard local setup. The Compose file starts PostgreSQL, the API, and Ollama; it does not start the Vite development server.

## Architecture

```mermaid
flowchart LR
  Browser[Browser / React SPA<br/>Vite :5173] -->|HTTP JSON, multipart, SSE| API[FastAPI API<br/>:8000]
  API -->|SQL / pgvector| DB[(PostgreSQL 16 + pgvector<br/>:5434 on host)]
  API -->|embedding and chat requests| Ollama[Ollama<br/>:11434]
  API -. optional hosted providers .-> Hosted[OpenAI / Anthropic]
  DB --- DBVol[(sage_pgdata volume)]
  Ollama --- ModelVol[(sage_ollama volume)]
```

### Local service endpoints

| Service | Local address | Notes |
| --- | --- | --- |
| Frontend (Vite) | `http://localhost:5173` | Started with `npm run dev` from `frontend/` |
| API | `http://localhost:8000` | Interactive docs at `/docs`; health at `/api/health` |
| PostgreSQL | `localhost:5434` | Container port `5432` mapped to host port `5434` |
| Ollama | `http://localhost:11434` | Container model API; default models are pulled separately |

Within the Compose network the API connects to the database at `db:5432` and Ollama at `ollama:11434`. The browser uses the API's host address, normally `http://localhost:8000/api`.

## How Sage works

### 1. Account and collection

Registration creates a user row and returns an access token and refresh token. Passwords are stored as bcrypt hashes, not as clear text. Authenticated API requests send the access token in the `Authorization: Bearer ...` header.

Each user can create collections. Collection lookup, document operations, search, and chat check ownership before returning or modifying a collection. A collection is the boundary that groups documents and their searchable chunks.

### 2. Document upload and ingestion

The browser uploads a file as `multipart/form-data` to the selected collection. The API accepts `.pdf`, `.docx`, `.md`, `.markdown`, and `.txt` files up to 20 MiB. Empty files and unsupported extensions are rejected.

The API records the document and returns without waiting for indexing. A FastAPI background task performs these steps:

1. Set the document state to `processing`.
2. Extract text. PDF extraction retains page numbers; DOCX, Markdown, and plain text use page `0` (not paginated).
3. Split each page's text into word-aware chunks. Default size is 900 characters with 150 characters of overlap. Chunk indices increase through the document. A lightweight character-based estimate is stored as `token_count`; no tokenizer is downloaded.
4. Embed chunks in batches of 32 through the configured embedding provider.
5. Store chunk text, embedding vector, page, index, estimated token count, and collection/document IDs in PostgreSQL.
6. Set the document to `ready` and store its chunk count. If extraction or embedding fails, set it to `failed` and expose the error on the document record.

The web client polls document status every two seconds while any document is `pending` or `processing`. A ready document can be searched and used as chat context.

### 3. Retrieval

When a user searches or asks a question, Sage embeds the query and retrieves candidate chunks from the selected collection.

- **Vector mode:** order chunks by pgvector cosine distance, then return the best results.
- **Hybrid mode (default):** independently rank candidates by vector distance and PostgreSQL English full-text rank. Combine the ranked lists with Reciprocal Rank Fusion (RRF), then take the top results. RRF combines rank positions rather than directly mixing vector-distance and text-rank scales.

The candidate pool is up to three times the requested top-k (at least top-k). The default top-k is 5. Citations include the source document, snippet (up to 300 characters), rank/index, and score. The score shown is based on vector similarity; a text-only result uses a fallback display score.

### 4. Grounded, streaming answer

For chat, Sage stores the user's message, retrieves context chunks, and builds a prompt with numbered sources such as `[1] filename.pdf (p.3)`. The system instruction tells the model to answer only from those sources, cite them with bracketed numbers, and say when the provided documents do not contain enough information.

The backend sends the system instruction, prior conversation turns, and the newly assembled context/question to the configured chat model. It streams generated text back to the browser using SSE. The stream includes a conversation ID, token events, citations, completion, and (if a provider errors) an error event. Once generation ends, the assistant message and citation data are saved in PostgreSQL.

The current chat panel maintains the active conversation ID while that collection view is open. The API also exposes conversation and message history endpoints. The chat UI does not currently load previous conversations into the panel on page refresh.

If no chunks match, the model receives an explicit `No context sources were found` prompt and should report that there is not enough information from the documents.

## Repository layout

```text
.
├── docker-compose.yml            # PostgreSQL, API, Ollama, persistent volumes
├── .env.example                  # Optional Compose overrides
├── README.md
├── backend/
│   ├── Dockerfile                # API image (Python 3.11)
│   ├── requirements.txt          # Python runtime and test dependencies
│   ├── app/
│   │   ├── main.py               # FastAPI app, CORS, lifespan/schema init
│   │   ├── config.py             # Environment settings and defaults
│   │   ├── db.py                 # SQLAlchemy engine, pgvector extension/indexes
│   │   ├── models.py             # ORM entities and relationships
│   │   ├── schemas.py            # Pydantic request/response contracts
│   │   ├── security.py           # Password hashing and JWT creation/decoding
│   │   ├── deps.py               # Authenticated-user dependency
│   │   ├── routers/              # Auth, collection, document, search, chat endpoints
│   │   ├── rag/                  # Extraction, chunking, ingestion, retrieval, prompts
│   │   └── ai/                   # Provider interfaces, factory, Ollama/OpenAI/Anthropic
│   ├── alembic/                  # Alembic configuration and migration scaffolding
│   ├── seed.py                   # Optional demo account and sample document
│   ├── eval.py                   # Lightweight retrieval hit-rate evaluation
│   └── tests/                    # Chunk and retrieval utility tests
└── frontend/
    ├── Dockerfile                # Optional production static-site image
    ├── nginx.conf                # SPA route fallback for that image
    ├── package.json              # Frontend scripts and dependencies
    ├── .env.example              # Optional Vite API URL override
    └── src/
        ├── api/client.ts         # HTTP, token handling, SSE reader, API methods
        ├── hooks/useAuth.tsx     # Authentication state
        ├── pages/                # Login, registration, collections, collection detail
        ├── components/           # Layout, document upload, chat, UI components
        └── lib/                  # Form schemas and styling helpers
```

## Requirements

- Docker Desktop with Docker Compose (recommended for the API, PostgreSQL, and Ollama).
- Node.js 20 or later and npm (for the Vite frontend).
- Internet access for the initial Docker image, npm package, and model downloads.
- Disk space for Docker images, PostgreSQL data, npm packages, and local models. The default `llama3.1` model is approximately 4.9 GB; Ollama's container image and model storage also require several gigabytes.

The API Dockerfile uses Python 3.11. A local Python installation is needed only when running the backend outside Docker or running its Python tools locally.

## Run locally with Docker

Commands below use PowerShell and start from the repository root.

### 1. Make sure Docker is available

If `docker` is not found in the current PowerShell session, add Docker Desktop's CLI directory to that session's `PATH`:

```powershell
$dockerBin = "$env:LOCALAPPDATA\Programs\DockerDesktop\resources\bin"
$env:Path += ";$dockerBin"
docker --version
docker compose version
```

Start Docker Desktop first if its engine is stopped.

### 2. Start database, API, and Ollama

```powershell
docker compose up -d --build
docker compose ps
```

Compose starts:

- `db`: PostgreSQL 16 with pgvector. Its readiness check uses `pg_isready`.
- `api`: FastAPI, built from `backend/Dockerfile`. It waits for the database health check before starting.
- `ollama`: Ollama model server.

The database initializes on API startup. `app.db.init_db()` creates the pgvector extension, SQLAlchemy tables, the HNSW vector index, and the GIN full-text index when absent. The app currently calls this idempotent initializer at startup; Alembic is present as scaffolding but is not the Compose startup migration mechanism.

### 3. Download the default local models

The models are stored in the named `sage_ollama` volume and need to be downloaded the first time:

```powershell
docker compose exec ollama ollama pull nomic-embed-text
docker compose exec ollama ollama pull llama3.1
docker compose exec ollama ollama list
```

`nomic-embed-text` produces 768-dimensional embeddings. `llama3.1` is the default chat model. Model pulls may take several minutes or longer depending on the connection. The named volume preserves downloaded models across container restarts.

### 4. Install and run the frontend

In a second PowerShell window:

```powershell
Set-Location .\frontend
npm install
npm run dev
```

Open <http://localhost:5173>. The default frontend API URL is `http://localhost:8000/api`, so no frontend `.env` file is required for the default local ports. Leave the Vite terminal running while using the app.

### 5. Verify the services

From the repository root, with Docker on `PATH`:

```powershell
docker compose ps
Invoke-RestMethod http://localhost:8000/api/health
Invoke-RestMethod http://localhost:8000/api/ai/info
docker compose exec ollama ollama list
```

The health endpoint should return `status: ok`. `docker compose ps` should show all three containers up and the database healthy. API docs are at <http://localhost:8000/docs>.

### Stop and restart

Stop containers while keeping database and model data:

```powershell
docker compose stop
```

Start them again:

```powershell
docker compose up -d
```

`docker compose down` removes the containers and network but keeps named volumes by default. Do not add `-v` unless you intentionally want to delete the persisted database and model volumes. Start the frontend separately with `npm run dev` whenever you want to use the web client.

### Optional demo seed

To create the documented demo account, sample collection, and sample document, run:

```powershell
docker compose exec api python seed.py
```

The seed script reports `demo@sage.dev` / `password123`. It is not required: users can register through the application. The seed inserts data into the current database; do not run it if you do not want demo records. For a non-demo or production environment, use a strong unique password and secret values.

## Environment configuration

All settings have defaults for the local Compose setup. The root `.env` is optional and is read by Compose for overrides; `.env.example` documents the available Compose settings. Existing `.env` files should be edited deliberately, not overwritten by copying an example over them.

### Compose/API variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `JWT_SECRET` | `dev-secret-change-me` | Signs JWTs. Set a long random value outside local development. |
| `CLIENT_URL` | `http://localhost:5173` | Single allowed browser origin for API CORS. Must match the frontend origin. |
| `AI_PROVIDER` | `ollama` | Chat provider: `ollama`, `openai`, or `anthropic`. |
| `EMBEDDING_PROVIDER` | `ollama` | Embedding provider: `ollama` or `openai`. Anthropic has no embedding provider here. |
| `EMBEDDING_DIM` | `768` | Width of stored pgvector embeddings; keep compatible with the selected embedding model and existing vectors. |
| `OLLAMA_CHAT_MODEL` | `llama3.1` | Ollama chat model name. |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | Ollama embedding model name. |
| `OPENAI_API_KEY` | empty | Required when selecting OpenAI for chat or embeddings. |
| `OPENAI_CHAT_MODEL` | `gpt-4o-mini` | OpenAI chat model. |
| `OPENAI_EMBED_MODEL` | `text-embedding-3-small` | OpenAI embedding model; provider requests the configured `EMBEDDING_DIM`. |
| `ANTHROPIC_API_KEY` | empty | Required when selecting Anthropic chat. |
| `ANTHROPIC_CHAT_MODEL` | `claude-3-5-sonnet-latest` | Anthropic chat model. |

Compose supplies the database URL and Ollama service URL internally. The Compose `DATABASE_URL` is `postgresql+psycopg2://sage:sage@db:5432/sage`; the Ollama base URL is `http://ollama:11434`. These container hostnames are not suitable for a backend process running directly on Windows.

To select OpenAI for chat and embeddings, create or edit the root `.env` with values such as:

```dotenv
AI_PROVIDER=openai
EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=your-key
```

Restart/recreate the API container after changing Compose environment variables:

```powershell
docker compose up -d --force-recreate api
```

Anthropic is chat-only in this codebase. For example, set `AI_PROVIDER=anthropic` and leave `EMBEDDING_PROVIDER=ollama` (or select OpenAI embeddings and configure its API key).

### Backend settings for direct local runs

`backend/.env.example` documents backend settings, including `DATABASE_URL`, `OLLAMA_BASE_URL`, RAG tuning options, and hosted provider keys. When running from `backend/`, the defaults point at PostgreSQL on `localhost:5434` and Ollama on `localhost:11434`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql+psycopg2://sage:sage@localhost:5434/sage` | SQLAlchemy database connection for local backend execution. |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Access token lifetime. |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Refresh token lifetime. |
| `CHUNK_SIZE` | `900` | Chunk size in characters. |
| `CHUNK_OVERLAP` | `150` | Chunk overlap in characters. |
| `TOP_K` | `5` | Default number of retrieved passages. |
| `RETRIEVAL_MODE` | `hybrid` | `hybrid` or `vector`. |
| `UPLOAD_DIR` | `uploads` | Present as a setting; current ingestion passes upload bytes to the background task rather than persisting them in this directory. |

`OLLAMA_BASE_URL` is `http://localhost:11434` for a backend running directly on the host. For an API container, Compose overrides it with `http://ollama:11434`.

### Frontend variable

| Variable | Default | Purpose |
| --- | --- | --- |
| `VITE_API_URL` | `http://localhost:8000/api` | Base URL used by the browser API client. Set in `frontend/.env` only when the API is hosted elsewhere. |

Vite environment values are compiled into the frontend at build time. For the optional frontend Docker image, the Dockerfile provides a `VITE_API_URL` build argument.

## Use the application

1. Open <http://localhost:5173> and choose **Register**, or log in with an existing account.
2. Create a knowledge base/collection and optionally add a description.
3. Open the collection and upload documents by clicking the upload area or dragging files onto it.
4. Wait until each document shows **Ready**. A document marked **Failed** displays its processing error.
5. Ask a question in the chat panel. Answers stream in as they are generated; citations show numbered source excerpts and the originating file.
6. Create another collection to keep unrelated document sets separate.

Supported extensions are PDF, DOCX, Markdown, and TXT. The current upload endpoint enforces a 20 MiB maximum per file.

## API reference

All API routes are under `/api`. FastAPI's interactive OpenAPI UI is available at `/docs`, with the raw schema at `/openapi.json`.

Except for registration, login, token refresh, health, and AI info, routes require `Authorization: Bearer <access_token>`. Request bodies are JSON except document upload, which uses multipart form data.

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/api/auth/register` | Create an account; returns user and access/refresh tokens. |
| `POST` | `/api/auth/login` | Authenticate; returns user and tokens. |
| `POST` | `/api/auth/refresh` | Exchange a refresh token for a new token pair. |
| `GET` | `/api/auth/me` | Return the authenticated user. |
| `GET` | `/api/collections` | List the current user's collections. |
| `POST` | `/api/collections` | Create a collection. |
| `GET` | `/api/collections/{collection_id}` | Get a collection owned by the current user. |
| `DELETE` | `/api/collections/{collection_id}` | Delete a collection and its dependent data. |
| `GET` | `/api/collections/{collection_id}/documents` | List documents and processing statuses. |
| `POST` | `/api/collections/{collection_id}/documents` | Upload and asynchronously ingest a document. |
| `DELETE` | `/api/collections/{collection_id}/documents/{document_id}` | Delete a document and its chunks. |
| `POST` | `/api/collections/{collection_id}/search` | Embed a query and return ranked citations/passages. |
| `POST` | `/api/collections/{collection_id}/chat` | Create or continue a conversation; returns an SSE response. |
| `GET` | `/api/collections/{collection_id}/conversations` | List conversations in a collection. |
| `GET` | `/api/conversations/{conversation_id}/messages` | Get stored conversation messages and citations. |
| `GET` | `/api/ai/info` | Report configured providers, model names, and embedding dimension. |
| `GET` | `/api/health` | Simple API liveness response. |

### Example request bodies

Create a collection:

```json
{
  "name": "Research papers",
  "description": "Papers and notes about retrieval systems"
}
```

Search a collection:

```json
{
  "query": "How does the system rank relevant passages?",
  "top_k": 5
}
```

Start or continue a chat (omit `conversation_id` to create a conversation):

```json
{
  "message": "Summarize the retrieval approach.",
  "conversation_id": null
}
```

### Chat SSE events

Each SSE message is JSON in a `data:` line. The current event types are:

| Event `type` | Fields | Meaning |
| --- | --- | --- |
| `meta` | `conversation_id` | Conversation created or selected for the request. |
| `token` | `value` | Next generated text fragment. |
| `citations` | `citations` | Numbered source records for the answer. |
| `error` | `message` | Provider/generation error. |
| `done` | — | Stream processing finished. |

## Run services without Docker

The standard development workflow runs only the frontend on Windows; the Compose-managed backend, database, and Ollama remain in Docker. Running the API directly on Windows is also supported, but still requires a reachable PostgreSQL database with pgvector and a reachable Ollama server (or configured hosted providers).

### Run just the frontend on the host

Start Docker services from the repository root, then:

```powershell
Set-Location .\frontend
npm install
npm run dev
```

### Run the API directly on Windows

Start PostgreSQL with pgvector and Ollama first. With the supplied Compose stack running, their host addresses are `localhost:5434` and `localhost:11434`.

```powershell
Set-Location .\backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

The backend defaults already point to the host-mapped database and Ollama addresses. If you configure different endpoints, use a `backend/.env` based on `backend/.env.example`. Do not start this direct backend and the Compose API simultaneously on the same host port `8000`.

### Build the production frontend container

The included `frontend/Dockerfile` builds the SPA and serves it with Nginx on port 80. It is not included as a service in the current `docker-compose.yml`. For example, build and run it separately from the repository root:

```powershell
docker build -t sage-frontend ./frontend
docker run --rm -p 8080:80 sage-frontend
```

Open <http://localhost:8080>. If the API is not at `http://localhost:8000/api`, pass a different `VITE_API_URL` at image build time, and configure backend `CLIENT_URL` to the browser origin (for example `http://localhost:8080`).

## Tests and retrieval evaluation

Run the backend tests inside the API container:

```powershell
docker compose exec api python -m pytest -q
```

The current tests cover chunk splitting, overlap, page tracking, edge cases, context prompt assembly, and reciprocal-rank-fusion behavior. They do not exercise a live model provider or a browser end-to-end flow.

The lightweight retrieval evaluator expects a seeded collection and uses a few fixed question/keyword pairs:

```powershell
docker compose exec api python seed.py
docker compose exec api python eval.py
```

It reports hit-rate@5. The seed step creates demo data in the current database (see [Optional demo seed](#optional-demo-seed)).

For frontend static type checking and production bundling:

```powershell
Set-Location .\frontend
npm run build
```

## Troubleshooting

### `docker` is not recognized in PowerShell

Add Docker Desktop's CLI folder to the current session's path, as shown in [Run locally with Docker](#run-locally-with-docker), and make sure Docker Desktop's engine is running.

### API container is not starting

Inspect status and logs:

```powershell
docker compose ps -a
docker compose logs --tail 100 api
docker compose logs --tail 100 db
```

The API waits for PostgreSQL's health check. If database startup is still in progress, allow it to become healthy and inspect its logs before restarting services.

### Port is already in use

The published ports are `5173` (Vite), `8000` (API), `5434` (PostgreSQL host mapping), and `11434` (Ollama). Stop the process already using the conflicting port or deliberately change the relevant port mapping/configuration. The database inside Compose remains on port `5432`.

### Chat or ingestion says a model is missing

Check configured names with `GET /api/ai/info` and list Ollama models:

```powershell
docker compose exec ollama ollama list
```

Pull the defaults if necessary:

```powershell
docker compose exec ollama ollama pull nomic-embed-text
docker compose exec ollama ollama pull llama3.1
```

### CORS error in the browser

The API allows the one origin configured by `CLIENT_URL`, defaulting to `http://localhost:5173`. Set it to the exact frontend origin (scheme, hostname, and port), then recreate the API container. A different hostname such as `127.0.0.1` is a different origin from `localhost`.

### Document processing fails

Check the document's displayed error and API logs. Confirm the file type is supported, the file is non-empty and no larger than 20 MiB, PostgreSQL is healthy, and the configured embedding service/model is available. A scanned PDF containing only images may have no extractable text because this project extracts PDF text and does not include OCR.

### Changing embedding configuration after indexing

The database vector column is created with `EMBEDDING_DIM` (default 768), and already stored vectors belong to the model/provider that created them. Keep dimensions compatible and avoid mixing incompatible embedding models in one collection. Changing provider/model may require re-ingesting documents so query and stored vectors use the same embedding space; do not drop or recreate the database casually.

## Deployment notes

- The included frontend image is a static Nginx build; the default Compose file does not include it.
- The Compose API configuration is intended for the Compose network, where the database and Ollama are available as `db` and `ollama`.
- For a deployed frontend, build it with the deployed API base URL in `VITE_API_URL` and set API `CLIENT_URL` to the deployed frontend origin.
- Use a managed PostgreSQL service with pgvector or a secured PostgreSQL+pgvector deployment. Back up database contents and test restores.
- Use strong secrets, HTTPS, restricted network access, and hosted model providers or appropriately provisioned Ollama for production.
- The Compose file's database credentials and JWT secret defaults are for local development and must not be treated as production secrets.

## Security and data notes

- Passwords are bcrypt-hashed. Access/refresh tokens are signed JWTs; the default local JWT secret is intentionally a development default and must be overridden for a real deployment.
- The frontend stores tokens in browser `localStorage`. Protect the machine/browser and use HTTPS when deployed.
- The default PostgreSQL credentials (`sage` / `sage`) are only intended for a local setup.
- Uploaded bytes are passed to the ingestion background task and are not retained as an uploaded-file archive by the current implementation. Extracted chunk text and embeddings are retained in PostgreSQL.
- `sage_pgdata` stores database records and `sage_ollama` stores models. Removing volumes deletes their associated persistent data. `docker compose down` without `-v` keeps the volumes.
- Collection/document deletions cascade to dependent records; the UI asks for confirmation before deleting a collection.

## License

MIT
