# Architecture Specification (SvelteKit + FastAPI)

## 1. High-Level Design

┌─────────────────────────────────────────────────────────┐
│ Browser (SvelteKit Frontend) │
│ • Reactive UI components │
│ • API client (fetch wrappers) │
│ • Web Audio API for playback │
│ • Local storage for preferences │
└─────────────┬───────────────────────────────────────────┘
│ HTTP/JSON (localhost:8000)
▼
┌─────────────────────────────────────────────────────────┐
│ FastAPI Backend (localhost:8000) │
│ • REST API endpoints (profiles, islands, sentences) │
│ • TTS Engine Manager (Edge-TTS, F5-TTS, etc.) │
│ • AI/LLM Prompting & JSON Parsing │
│ • Audio Processing (PyDub) & Caching │
│ • SQLite Database (async SQLAlchemy) │
└─────────────┬───────────────────────────────────────────┘
│ File I/O
▼
┌─────────────────────────────────────────────────────────┐
│ Local Filesystem │
│ • backend/data/pimslika.db (SQLite) │
│ • backend/data/audio/ (cached .wav files) │
│ • backend/models/ (downloaded TTS weights) │
└─────────────────────────────────────────────────────────┘

## 2. Data Flow Patterns

### 2.1 CRUD Operations (Profiles, Islands, Sentences)

Frontend (SvelteKit)
│
├─► GET /api/v1/profiles
│ └─► FastAPI: SELECT * FROM profiles
│ └─► Return JSON array
│
├─► POST /api/v1/islands
│ ├─► Validate with Pydantic schema
│ ├─► INSERT INTO islands
│ ├─► Trigger background AI generation (async task)
│ └─► Return {id: 123, status: "generating"}
│
└─► GET /api/v1/islands/123/sentences
└─► Return sentences + decomposition JSON

### 2.2 TTS Generation Flow

Frontend
│
├─► POST /api/v1/tts/generate
│ {
│ "text": "je mange du pain",
│ "language": "fr-FR",
│ "engine": "f5-tts",
│ "voice_id": "default",
│ "speed": 1.0
│ }
│
▼
FastAPI
├─► Check audio_cache table for existing file
├─► If miss: route to selected TTSEngine.generate()
├─► Save .wav to backend/data/audio/{hash}.wav
├─► Insert cache record in audio_cache table
└─► Return { "audio_url": "/static/audio/abc123.wav" }
│
▼
Frontend
└─► Play via Web Audio API: new Audio("/static/audio/abc123.wav")

### 2.3 Learning Session Assembly

Frontend
│
├─► POST /api/v1/audio/sessions
│ {
│ "island_ids": [1, 2, 3],
│ "session_type": "socratic_drill",
│ "delays": { "word": 1500, "sentence": 3000 }
│ }
│
▼
FastAPI (Background Task)
├─► Fetch sentences + decomposition from DB
├─► Build script: [prompt, delay, answer, delay, ...]
├─► Fetch/generate audio segments for each step
├─► Stitch with PyDub + insert delays
├─► Save session_{id}.wav to disk
└─► Return { "session_url": "/static/audio/session_456.wav" }
│
▼
Frontend
└─► Load in custom AudioPlayer component with progress bar

## 3. Communication Patterns

### 3.1 Frontend → Backend (HTTP)
```typescript
// frontend/src/lib/api/client.ts
import { browser } from '$app/environment';

const API_BASE = browser ? 'http://localhost:8000/api/v1' : '';

export async function api<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const response = await fetch(`${API_BASE}${endpoint}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  });
  
  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new Error(error.detail || 'API request failed');
  }
  
  return response.json();
}

// Usage example
export const profilesApi = {
  list: () => api<Profile[]>('/profiles'),
  create: (data: CreateProfileInput) => 
    api<Profile>('/profiles', { method: 'POST', body: JSON.stringify(data) }),
};
```

### 3.2 Backend → Frontend (Server-Sent Events / WebSocket)
For long-running tasks (AI generation, TTS stitching):

```python
# backend/app/api/v1/endpoints/tts.py
from fastapi import BackgroundTasks
from fastapi.responses import StreamingResponse
import asyncio

async def generate_with_progress(task_id: str, request: TTSRequest):
    """Yield progress updates via SSE"""
    async def event_stream():
        for progress in range(0, 101, 10):
            yield f"data: {json.dumps({'task_id': task_id, 'progress': progress})}\n\n"
            await asyncio.sleep(0.1)
        yield f"data: {json.dumps({'task_id': task_id, 'done': True, 'audio_url': '/static/audio/xyz.wav'})}\n\n"
    
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

```svelte
<!-- frontend/src/lib/components/ProgressToast.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let taskId;
  
  onMount(() => {
    const eventSource = new EventSource(`http://localhost:8000/api/v1/tts/progress/${taskId}`);
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.done) {
        // Show success, play audio
        eventSource.close();
      } else {
        // Update progress bar
        progress = data.progress;
      }
    };
  });
</script>
```

## 4. File Serving Strategy
### 4.1 Static Audio Files

```python
# backend/app/main.py
from fastapi.staticfiles import StaticFiles
from pathlib import Path

app.mount(
    "/static/audio",
    StaticFiles(directory=Path(__file__).parent.parent / "data" / "audio"),
    name="audio"
)
```

Frontend plays audio via direct URL:
```svelte
<audio controls src="http://localhost:8000/static/audio/abc123.wav" />
```

### 4.2 Streaming Large Sessions (Optional)
For very long learning sessions (>10 min), use chunked streaming:

```python
# backend/app/api/v1/endpoints/audio.py
from fastapi.responses import StreamingResponse
from pydub import AudioSegment

async def stream_session(session_id: int):
    """Stream audio in chunks to avoid loading entire file in memory"""
    file_path = get_session_path(session_id)
    
    def iterfile():
        with open(file_path, "rb") as f:
            while chunk := f.read(8192):
                yield chunk
    
    return StreamingResponse(
        iterfile(),
        media_type="audio/wav",
        headers={"Content-Disposition": f"inline; filename=session_{session_id}.wav"}
    )
```

## 5. Error Handling Strategy
### Backend (FastAPI)
```python
from fastapi import HTTPException, status
from pydantic import ValidationError

@app.post("/api/v1/tts/generate")
async def generate_tts(request: TTSRequest):
    try:
        # ... generation logic ...
    except TTSEngineError as e:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail={"code": "TTS_ENGINE_ERROR", "message": str(e)}
        )
    except ValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail={"code": "VALIDATION_ERROR", "errors": e.errors()}
        )
```

### Frontend (SvelteKit)
```ts
// frontend/src/lib/api/errors.ts
export class ApiError extends Error {
  constructor(
    public code: string,
    public message: string,
    public status: number,
    public details?: any
  ) {
    super(message);
  }
}

export async function handleApiError(error: any): Promise<never> {
  if (error.response?.data?.detail) {
    const { code, message, errors } = error.response.data.detail;
    throw new ApiError(code, message, error.response.status, errors);
  }
  throw error;
}

// Usage in component
try {
  await profilesApi.create(newProfile);
} catch (err) {
  if (err instanceof ApiError) {
    toast.error(err.message); // User-friendly message
    console.error('API Error details:', err.details); // Dev logging
  }
}
```

## 6. Development Workflow
### Hot Reload Setup
- Backend: uvicorn --reload watches Python files
- Frontend: npm run dev watches Svelte files
- Shared Types: Optional: generate TypeScript from Pydantic schemas using openapi-typescript
  
### Debugging
```bash
# Backend logs
uvicorn app.main:app --reload --log-level debug

# Frontend logs
npm run dev  # Shows in browser console

# API testing
http GET http://localhost:8000/api/v1/profiles  # Using HTTPie
# or use the built-in docs: http://localhost:8000/docs
```

### Testing Strategy
```python
# backend/tests/test_api.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_profile():
    response = client.post("/api/v1/profiles", json={
        "name": "Test",
        "target_language": "fr-FR",
        "native_language": "en-US"
    })
    assert response.status_code == 201
    assert response.json()["name"] == "Test"
```

```ts
// frontend/src/lib/api/client.test.ts
import { describe, it, expect, vi } from 'vitest';
import { api } from './client';

global.fetch = vi.fn();

describe('api client', () => {
  it('handles successful response', async () => {
    (fetch as any).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ id: 1, name: 'Test' })
    });
    
    const result = await api('/profiles/1');
    expect(result.name).toBe('Test');
  });
});
```
