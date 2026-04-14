# Citadel

AI-powered traffic safety analytics platform for accident detection from uploaded media and live DOT camera networks.

## What It Does

- Detects accidents from videos/images using a YOLO26 ONNX model.
- Monitors live Caltrans + Iowa DOT cameras (snapshot and HLS stream modes).
- Creates incident events and ticket workflow records (`issued -> pending -> resolved`).
- Stores evidence with a hybrid strategy: local-first + asynchronous Supabase Storage backfill.
- Dispatches alerts through Twilio, Email, Webhook, and Telegram with cooldown controls.
- Provides a TypeScript dashboard with overview stats, incidents, monitoring, cameras, and settings.

## Tech Stack

- Frontend: TypeScript, Vite, Vanilla CSS, Leaflet
- Backend: FastAPI, SQLAlchemy, Uvicorn
- Model runtime: ONNX Runtime + OpenCV + Pillow
- Database: Supabase Postgres
- Auth: Supabase Auth (email/password)
- Storage: Supabase Storage (`ticket-evidence`) with local evidence fallback

## Repository Layout

- `backend/` FastAPI app, services, model pipeline, and API routes
- `frontend/` Vite app and dashboard UI
- `weights/` optional local model artifacts

## Quick Start

Prerequisites:

- Python 3.10+
- Node.js 18+

Clone and install:

```bash
git clone https://github.com/gopesh353/Citadel.git
cd Citadel

python -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt

cd frontend
npm install
cd ..
```

Create env files:

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Run locally:

```bash
# Terminal 1
cd backend
uvicorn app.main:app --reload --port 8000

# Terminal 2
cd frontend
npm run dev
```

- Frontend: `http://localhost:3000`
- API: `http://localhost:8000`
- Swagger: `http://localhost:8000/docs`

## Environment Variables

Use `backend/.env.example` and `frontend/.env.example` as source of truth.

Backend required:

- `DATABASE_URL` - Supabase Postgres connection string
- `SUPABASE_URL`
- `SUPABASE_PUBLISHABLE_KEY`
- `SUPABASE_SECRET_KEY`

Backend common optional:

- `MODEL_PATH` (default `./models/model.onnx`)
- `MODEL_URL` (auto-download source)
- `EVIDENCE_DIR` (default `./evidence`)
- `UPLOADS_DIR` (default `./uploads`)
- `CITADEL_BASE_URL` (public URL used in notifications)
- `CAMERA_LIST_CACHE_TTL_HOURS` (default `24`)
- Twilio/Telegram notification credentials

Frontend:

- `VITE_SUPABASE_URL`
- `VITE_SUPABASE_PUBLISHABLE_KEY`
- `VITE_API_URL` (set for production, optional in local dev)

## Auth and Access Model

- Login endpoint: `POST /api/auth/login`
- Frontend stores Supabase access + refresh tokens and sends bearer token on API calls.
- Most API endpoints require auth via `get_current_admin` (Supabase token validation).
- Public health endpoint: `GET /api/health`

## Evidence Storage Behavior

Current behavior is intentionally resilient:

1. Detection writes evidence locally (`EVIDENCE_DIR`) first.
2. Backend enqueues async upload to Supabase Storage bucket `ticket-evidence`.
3. Event/ticket `evidence_path` is updated to the remote URL after successful upload.
4. Local evidence remains as fallback/back-compat and is served at `/evidence/...`.

This allows safe operation even if transient storage upload issues occur.

## API Summary

Core:

- `GET /api/health`
- `POST /api/auth/login`
- `GET /api/stats`
- `GET /api/stats/services`
- `GET /api/settings`
- `PUT /api/settings`
- `GET /api/settings/notifications`
- `PUT /api/settings/notifications`
- `DELETE /api/settings/data`

Detection:

- `POST /api/detect/video`
- `POST /api/detect/image`
- `POST /api/detect/images`
- `GET /api/detect/progress/{job_id}`

Incidents / Events / Tickets:

- `GET /api/incidents`
- `GET /api/incidents/{incident_id}`
- `PATCH /api/incidents/{incident_id}/status`
- `GET /api/incidents/stats/overview`
- `GET /api/events`
- `GET /api/events/{event_id}`
- `GET /api/tickets`
- `GET /api/tickets/{ticket_id}`
- `PATCH /api/tickets/{ticket_id}`

Alerts:

- `GET /api/alerts`
- `GET /api/alerts/stats`

Cameras and monitoring:

- `GET /api/cameras/districts`
- `GET /api/cameras` (supports `source=caltrans|iowa`, search, limit)
- `GET /api/cameras/{camera_id}/info`
- `GET /api/cameras/{camera_id}/snapshot`
- `GET /api/cameras/{camera_id}/snapshot-url`
- `GET /api/cameras/{camera_id}/snapshot-changed`
- `GET /api/cameras/{camera_id}/stream-info`
- `GET /api/cameras/hls-proxy/{path}`
- `GET /api/cameras/iowa-hls-proxy/{path}`
- `POST /api/cameras/monitor/start`
- `POST /api/cameras/monitor/{camera_id}/pause`
- `POST /api/cameras/monitor/{camera_id}/resume`
- `POST /api/cameras/monitor/{camera_id}/stop`
- `POST /api/cameras/monitor/stop`
- `GET /api/cameras/monitor/status`
- `GET /api/cameras/monitor/{camera_id}/status`

Iowa-specific compatibility routes:

- `GET /api/iowa/cameras`
- `GET /api/iowa/cameras/regions`
- `GET /api/iowa/cameras/{camera_id}/snapshot`
- `GET /api/iowa/cameras/{camera_id}/info`

## Interview Prep Guide (Backend + YOLO)

If you need to prepare deeply for technical interviews around this project, focus on the following in this order.

### 1) Backend system architecture (must know end-to-end)

- FastAPI app boot flow in `backend/app/main.py`:
  - verify DB connection (`verify_connection`)
  - ensure DB indexes (`ensure_performance_indexes`)
  - create runtime dirs (`evidence`, `uploads`, `data`)
  - load model once (`AccidentDetector.load()`)
  - wire detector into processor/routes/monitor service
  - preload Caltrans + Iowa camera metadata
  - restore active monitor sessions from DB
- Route boundaries in `backend/app/routes/*`:
  - auth, detection, incidents/events/tickets, stats, settings, alerts, cameras
- Service boundaries in `backend/app/services/*`:
  - camera ingest/caching/proxy
  - monitor loop orchestration
  - event/ticket creation
  - notification dispatch

### 2) Core request/data flow you should be able to explain

#### Manual upload flow (`/api/detect/video`, `/api/detect/image`, `/api/detect/images`)

1. Validate extension/type
2. Save upload (video path) or decode image
3. Read runtime settings (threshold/toggles)
4. Run inference via `VideoProcessor`
5. Deduplicate near-duplicate accidents
6. Persist Event rows, create Ticket rows for accidents
7. Store evidence locally first
8. Async backfill evidence to Supabase Storage
9. Dispatch notifications (Twilio/Email/Webhook/Telegram pathways)

#### Live monitoring flow (camera monitor routes)

1. Start monitor thread per camera (snapshot or HLS stream mode)
2. Poll based on camera update frequency (or stream interval)
3. Skip unchanged snapshots via HEAD/content change checks
4. Run detection on new frame
5. Create events/tickets + update monitor status metrics
6. Persist active monitor state for restart recovery

### 3) Data/model layer (know entity relationships)

From `backend/app/models.py` and related services:

- `Event` = detection occurrence (type/confidence/severity/source/evidence/bbox metadata)
- `Ticket` = workflow object created from accident events
- `ActiveMonitor` = persisted monitor state (camera_id, stream mode, pause state)
- Status lifecycle and operational behavior:
  - incidents/tickets progress through operational states (issued/pending/resolved style workflow)
  - monitor state survives restarts through `active_monitors` table restoration

### 4) Config and environment controls (high interview value)

From `backend/app/config.py`:

- inference controls:
  - `MODEL_PATH`, `MODEL_URL`
  - `CONFIDENCE_THRESHOLD_MANUAL`, `CONFIDENCE_THRESHOLD_CCTV`
  - `DETECT_ACCIDENTS`
- storage/runtime:
  - `EVIDENCE_DIR`, `UPLOADS_DIR`, `DATA_DIR`
- platform dependencies:
  - `DATABASE_URL`, Supabase keys
- notification controls:
  - Twilio + Telegram credentials and cooldown
- camera caching:
  - `CAMERA_LIST_CACHE_TTL_HOURS`

### 5) YOLO/ONNX pipeline details (you must be precise)

From `backend/app/detection/detector.py` and `processor.py`:

- runtime:
  - ONNX Runtime on CPU (`CPUExecutionProvider`)
  - model auto-downloads if missing
- preprocessing:
  - force RGB
  - resize to `640x640`
  - normalize to `[0,1]`
  - `HWC -> CHW`, add batch dim (`NCHW`)
- output parsing:
  - supports both common output shapes:
    - `[x, y, w, h, conf, class_id]`
    - `[x, y, w, h, class_scores...]`
  - handles transposed outputs (`(6, N)` vs `(N, 6+)`)
  - keeps only class `0` (accident) above threshold
  - rescales boxes back to original image dimensions
  - clamps boxes to image bounds
- post-inference semantics:
  - severity mapping by confidence:
    - high >= 0.85
    - medium >= 0.70
    - low < 0.70
  - per-image/per-frame dedup keeps highest-confidence accident
  - video dedup suppresses near-frame duplicates
- evidence generation:
  - annotate bbox + confidence onto frames
  - save JPEG evidence locally first for low-latency responses

### 6) Concurrency, reliability, and performance topics to prepare

- Why CPU inference + single model load in lifespan
- `asyncio.to_thread` use to avoid blocking event loop for heavy processing
- thread-based monitor design (one thread per active camera)
- lock usage for shared state (`_progress`, monitor maps)
- cache strategy:
  - in-memory camera cache + on-disk CSV cache
- resilience strategy:
  - local evidence write before remote storage backfill
  - graceful handling of notification/storage failures
  - monitor restore on restart
- stream capture reliability:
  - persistent ffmpeg process + restart/backoff behavior

### 7) Security and production-readiness questions to expect

- auth model:
  - bearer token checks through `get_current_admin`
  - protected API routes vs public health check
- input validation:
  - strict extension allowlists for uploads
- operational safety:
  - idempotent monitor persistence methods
  - DB transaction handling (`commit`/`rollback`)
- likely improvements you can discuss:
  - background job queue for long-running processing
  - stronger rate limits and request size limits
  - model versioning/rollbacks
  - observability (structured logs, tracing, metrics dashboards)

### 8) Backend files to master before interview

- `backend/app/main.py`
- `backend/app/config.py`
- `backend/app/detection/detector.py`
- `backend/app/detection/processor.py`
- `backend/app/routes/detection.py`
- `backend/app/services/monitor_service.py`
- `backend/app/services/camera_service.py`
- `backend/app/services/event_service.py`
- `backend/app/services/ticket_service.py`
- `backend/app/models.py`, `backend/app/schemas.py`, `backend/app/database.py`

### 9) Interview drill checklist (practice verbally)

- Explain one complete request path from upload -> detection -> event -> ticket -> notification.
- Explain difference between manual upload and live monitoring pipelines.
- Explain why dedup exists at multiple layers and how windows are chosen.
- Explain how evidence storage is made robust against transient cloud failures.
- Explain YOLO preprocessing and output decoding math step-by-step.
- Explain failure handling (bad file, model missing, DB failure, notification failure, stream failure).
- Explain scaling options (horizontal API scale, async workers, batching, GPU/accelerator inference).

## Testing

Backend tests:

```bash
cd backend
pytest
```

Frontend production build check:

```bash
cd frontend
npm run build
```

## Deployment Notes

- Frontend can be deployed on Vercel.
- Backend can be deployed on Render/Heroku-like services.
- Ensure backend dependency source includes `backend/requirements.txt` (contains Supabase Python client).

## License

Educational and research use.
