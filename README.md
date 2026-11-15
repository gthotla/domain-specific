# Enterprise Assistant — Domain-Specific RAG Chatbot

A team-built web app that lets an organization upload its own documents and chat
against them through a retrieval-augmented generation (RAG) pipeline, with an
optional LoRA fine-tuning workflow and an admin dashboard.

## What it does

- **Document ingestion** — upload PDFs/CSVs/TXT files; text is extracted,
  chunked (`RecursiveCharacterTextSplitter`, 1000 chars / 100 overlap) and
  embedded with `BAAI/bge-large-en` sentence-transformers.
- **Grounded chat** — `POST /rag` embeds the query, retrieves top-k chunks from
  Pinecone or Qdrant (per-user metadata filter), and generates an answer
  grounded in retrieved context via Google Gemini + LangChain.
- **Fine-tuning workflow** — `services/Finetune.py` runs LoRA (Unsloth +
  `trl.SFTTrainer`, rank 16) on curated data (default base
  `unsloth/Meta-Llama-3.1-8B`) to adapt tone/terminology per domain.
- **Auth** — email/password (bcrypt) + Google and GitHub OAuth; JWT in
  `httpOnly` cookies; separate user and admin middleware.
- **Admin dashboard** — manage documents, users, and model behavior (React UI).

## Architecture

```
React (Vite)  ──►  Flask API  ──┬──►  MongoDB + GridFS (users, files, chunks)
                                ├──►  Pinecone / Qdrant (vector embeddings)
                                └──►  LLM endpoint (Gemini / configured URL)
```

Backend routes (`backend/app/routes/`): `user.py` (signup/login/OAuth/OTP),
`admin.py`, `file.py` (upload), `vector.py`, `rag.py` (query).

## Tech stack

| Layer    | Tech |
|----------|------|
| Frontend | React 18, Vite, Tailwind CSS |
| Backend  | Flask 3, LangChain, Flask-PyMongo, Flask-Dance (OAuth) |
| Vectors  | Pinecone + `langchain-pinecone`, Qdrant + `langchain-qdrant`, `sentence-transformers`, `torch`, `transformers` |
| Data     | MongoDB (GridFS for file blobs) |
| Auth     | PyJWT, bcrypt, Google/GitHub OAuth |

## Setup

### 1. Backend

```bash
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp example.env .env   # then fill in real values (never commit .env)
python app/app.py
```

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env  # set VITE_GOOGLE_CLIENT_ID
npm run dev
```

### Environment variables (`backend/example.env`)

| Key | Purpose |
|-----|---------|
| `MONGO_URI` | MongoDB connection string |
| `SECRET_KEY` / `JWT_SECRET` | Flask + JWT signing (generate fresh per deploy) |
| `FLASK_ENV` | `development` / `production` |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | GitHub OAuth |
| `QDRANT_URL` (+ `QDRANT_API_KEY`) | Qdrant endpoint |
| `COLLECTION_NAME` | Vector collection name |
| `CACHE_DIR` | Local embedding-model cache |
| `PINECONE_KEY` | Pinecone API key |
| `NROK_URL` | Base URL of the LLM inference endpoint |

## Configuration

- Copy `backend/example.env` to `backend/.env` and fill in real values.
- Set the frontend's `VITE_GOOGLE_CLIENT_ID` in `frontend/.env` (copy from
  `frontend/.env.example`).

## Known limitations

- No retrieval-eval suite yet; inference goes through a single endpoint.
- Large-file uploads untested at scale; no automated tests or CI yet.
