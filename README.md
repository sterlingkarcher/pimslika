# Pimslika - Personal Language Learning App (v2)

## 🎯 Project Overview
Pimslika is a **local-first web application** for immersive language learning. It combines Pimsleur-style audio drills with Glossika-style sentence mining, powered by local AI.

**Architecture**: SvelteKit frontend (browser) + FastAPI backend (local server) + SQLite (local file)

## 🚀 Key Features
1. **Profile Management**: Multi-user support with native/target language pairs
2. **Language Islands**: Thematic collections generated via local LLM prompts
3. **Contextual Decomposition**: AI-driven word-level mapping (e.g., "du pain" vs "pain")
4. **Multi-Engine TTS**: Edge-TTS (cloud fallback), F5-TTS, ChatterBox, Qwen3-TTS (voice design)
5. **Learning Sessions**: Automated audio stitching for Socratic drills (Question → Delay → Answer)
6. **Offline-First**: All data stored locally in SQLite; audio cached on disk
7. **Browser UI**: Modern SvelteKit interface with reactive updates

## 🛠 Tech Stack

### Frontend (SvelteKit)
- Svelte 5 + TypeScript
- TailwindCSS + shadcn-svelte for UI components
- TanStack Query for server state management
- Web Audio API for playback control

### Backend (FastAPI)
- FastAPI + Uvicorn (async server)
- SQLite + SQLAlchemy (async)
- Pydantic v2 for data validation
- PyTorch + Transformers for local AI/TTS

### Shared
- Python 3.11+ (backend)
- Node 20+ (frontend)
- FFmpeg (audio processing)

## 📂 Project Structure
pimslika/
├── frontend/ # SvelteKit app
│ ├── src/
│ │ ├── lib/
│ │ │ ├── api/ # API client (fetch wrappers)
│ │ │ ├── components/ # Reusable Svelte components
│ │ │ ├── stores/ # Svelte stores (auth, settings)
│ │ │ └── utils/ # Helpers (audio player, formatters)
│ │ ├── routes/
│ │ │ ├── +layout.svelte
│ │ │ ├── dashboard/
│ │ │ ├── islands/
│ │ │ ├── settings/
│ │ │ └── player/
│ │ ├── app.html
│ │ └── hooks.server.ts # Optional: SSR auth
│ ├── static/ # Static assets
│ ├── svelte.config.js
│ ├── tailwind.config.ts
│ └── package.json
│
├── backend/ # FastAPI server
│ ├── app/
│ │ ├── api/
│ │ │ ├── v1/
│ │ │ │ ├── endpoints/
│ │ │ │ │ ├── profiles.py
│ │ │ │ │ ├── islands.py
│ │ │ │ │ ├── sentences.py
│ │ │ │ │ ├── tts.py
│ │ │ │ │ └── audio.py
│ │ │ │ └── deps.py # Dependencies (DB session, auth)
│ │ ├── core/
│ │ │ ├── config.py # Settings (pydantic-settings)
│ │ │ ├── db.py # SQLite engine + session
│ │ │ ├── tts/
│ │ │ │ ├── base.py # Abstract TTSEngine
│ │ │ │ ├── edge.py
│ │ │ │ ├── f5.py
│ │ │ │ ├── chatterbox.py
│ │ │ │ └── qwen3.py
│ │ │ ├── ai/
│ │ │ │ ├── llm.py # Local LLM prompting
│ │ │ │ └── decomposition.py
│ │ │ └── audio/
│ │ │ ├── processor.py # PyDub stitching
│ │ │ └── cache.py # Disk cache manager
│ │ ├── models/
│ │ │ ├── profile.py
│ │ │ ├── island.py
│ │ │ ├── sentence.py
│ │ │ └── voice_config.py
│ │ ├── schemas/ # Pydantic schemas (request/response)
│ │ ├── main.py # FastAPI app factory
│ │ └── worker.py # Background task runner (optional)
│ ├── data/
│ │ ├── pimslika.db # SQLite database
│ │ └── audio/ # Cached audio files
│ ├── alembic/ # Database migrations
│ ├── pyproject.toml
│ └── requirements.txt
│
├── scripts/
│ ├── start-dev.sh # Run both frontend + backend
│ ├── migrate.py # DB migration helper
│ └── backup.py # Backup DB + audio folder
│
├── docs/ # This documentation
├── docker-compose.yml # Optional: containerized dev
└── README.md

## 🏗 Implementation Guidelines
1. **API-First Design**: Backend exposes REST endpoints; frontend consumes via typed client
2. **Async Everywhere**: FastAPI endpoints are `async def`; SvelteKit uses `await fetch`
3. **Local-Only Security**: No auth needed for personal use; optional simple token for multi-user
4. **File Handling**: Audio files stored on disk; backend serves via `/static/audio/{path}`
5. **CORS**: Configure FastAPI to allow requests from `http://localhost:5173` (SvelteKit dev)
6. **Error Handling**: Backend returns structured errors; frontend displays user-friendly toasts

## 🚦 Getting Started (Dev)

### Prerequisites
```bash
# Backend
python3.11 -m venv backend/.venv
source backend/.venv/bin/activate
cd backend && pip install -r requirements.txt

# Frontend
cd frontend && npm install

# System dependencies
brew install ffmpeg  # macOS
# or: sudo apt install ffmpeg  # Linux
# or: choco install ffmpeg  # Windows
# or: choco install ffmpeg  # Windows
```

### Run Locally

```bash
# Terminal 1: Start FastAPI backend
cd backend
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

# Terminal 2: Start SvelteKit frontend
cd frontend
npm run dev

# Open browser: http://localhost:5173
# API docs: http://localhost:8000/docs
```

### One-Command Dev (Optional)
```bash
# Using concurrently
npm install -g concurrently
# Then in root:
concurrently "cd backend && uvicorn app.main:app --reload" "cd frontend && npm run dev"
```

## 📚 Documentation
- docs/01_architecture.md — System architecture & data flow
- docs/02_database_schema.md — SQLite schema
- docs/03_tts_engine.md — TTS engine implementations
- docs/04_frontend_components.md — SvelteKit component specs
- docs/05_backend_api.md — FastAPI endpoint specs
- docs/06_ai_workflow.md — LLM prompting
- docs/07_audio_processing.md — Audio stitching
- docs/08_implementation_guide.md — Step-by-step build path

## 🤝 Contributing (to yourself)
- Keep backend and frontend decoupled — communicate only via HTTP API
- Use Pydantic schemas on backend; generate TypeScript types for frontend (optional: openapi-typescript)
- Test TTS engines on your hardware before relying on them
- Backup backend/data/ regularly — it contains your learning progress!
