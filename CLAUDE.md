# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

This is a monorepo with Docker Compose orchestration containing two independent sub-repos:

- `backend/` — Express + TypeScript API and BullMQ worker
- `frontend/` — React + TypeScript + Vite SPA, plus `frontend/extension/` (Chrome MV3 extension sharing the same `src/`)

Each sub-directory has its own `.git`, `package.json`, and `node_modules`.

## Development

### Start the full stack (development mode)

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

This mounts `./backend` and `./frontend` directly into their containers so code changes are picked up by hot-reload without rebuilding images.

After changing `package.json` in either service:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm backend npm install
docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm frontend npm install
```

### Run services individually (outside Docker)

```bash
# Backend API (port 4000)
cd backend && npm run dev

# Background worker
cd backend && npm run worker:dev

# Frontend (port 5173)
cd frontend && npm run dev
```

### Build

```bash
cd backend && npm run build              # tsc → dist/
cd frontend && npm run build             # tsc + vite → dist/
cd frontend && npm run build:extension   # tsc + vite → dist-extension/
```

### Browser extension

`npm run build:extension` emits an unpacked Chrome MV3 extension into `frontend/dist-extension/`, loadable at `chrome://extensions`. `npm run dev:extension` rebuilds on change (still requires clicking reload in Chrome).

The extension records the audio of a meeting tab plus your microphone, encodes 16 kHz mono WAV in an offscreen document, and POSTs it to the normal `/meetings/upload` endpoint — the backend pipeline is unchanged. WAV is used because Gemini does not accept the WebM/Opus that `MediaRecorder` produces.

Two things it depends on:

- **`SESSION_COOKIE_SAMESITE=none`** — requests from `chrome-extension://` are cross-site, so a `lax` cookie is never sent. Implies `Secure`, which Chrome permits on `localhost`.
- **A stable extension ID**, pinned by `"key"` in `frontend/extension/manifest.json` and listed in `CLIENT_ORIGIN`. The private half (`frontend/extension-key.pem`) is gitignored and only needed to package a `.crx`.

Design notes and the full verification checklist live in `docs/browser-extension.md`.

### Type checking (no emit)

```bash
cd backend && npm run typecheck
cd frontend && npm run typecheck
```

### Database migrations

Migrations run automatically on server and worker startup via `migrateDatabase()`. To run manually:

```bash
cd backend && npm run db:migrate
```

SQL files live in `backend/migrations/` and are applied in filename order. The `schema_migrations` table tracks what has been applied.

## Environment Variables

Copy `.env.example` to `.env` at the repo root. Key variables:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `SESSION_SECRET` | Secret for `express-session` (min 16 chars; also read from `JWT_ACCESS_SECRET`) |
| `SUPABASE_S3_*` | Supabase Storage S3 credentials for recording uploads/downloads |
| `SUPABASE_STORAGE_BUCKET` | S3 bucket name (default: `meeting-recordings`) |
| `GEMINI_API_KEY` | Google Gemini API key for transcription and AI features |
| `GEMINI_MODEL` | Gemini model for summary/action-item extraction (default: `gemini-1.5-flash`) |
| `GEMINI_TRANSCRIPTION_MODEL` | Gemini model for audio transcription (default: `gemini-3.5-flash`) |
| `SMTP_EMAIL_ADDRESS` / `SMTP_PASSWORD` | Email delivery for verification and password reset |
| `CLIENT_ORIGIN` | Comma-separated allowed CORS origins (include `chrome-extension://<id>` for the extension) |
| `SESSION_COOKIE_SAMESITE` | `lax`/`strict`/`none`. Must be `none` for the browser extension; implies `Secure` |
| `SESSION_COOKIE_SECURE` | Overrides the cookie `Secure` flag; normally left unset |

If `GEMINI_API_KEY` is absent, AI features degrade gracefully (fallback summaries, no action items).

## Architecture

### Backend (`backend/src/`)

**Two processes share the same codebase:**

- `server.ts` — HTTP API via `app.ts` (Express)
- `worker.ts` — BullMQ worker that processes the `meeting-processing` queue

**Request path:**

1. Authenticated request hits `authRoutes` or `meetingRoutes` (all meeting endpoints require `requireAuth` middleware).
2. Controllers in `controllers/` delegate to services in `services/`.
3. `meetingController.ts#upload` stores the file in Supabase S3 via `storageService`, records metadata in PostgreSQL, and enqueues a job via `queueService`.

**Worker pipeline** (triggered by a BullMQ job with `{ meetingId }`):

1. Download recording from S3 (`storageService.downloadRecording`)
2. Transcribe audio via Gemini Files API (`transcriptionService.transcribeRecordingWithGemini`) — uploads file to Gemini, generates content, deletes the uploaded file.
3. Persist transcript and segments to PostgreSQL.
4. Generate summary (`aiService.generateSummary`) → insert into `summaries`.
5. Extract action items (`aiService.extractActionItems`) → insert into `action_items`.
6. Set meeting `status` to `completed`.

Meeting statuses progress: `uploaded` → `transcribing` → `summarizing` → `extracting_action_items` → `completed` (or `failed`).

**Session auth:** Cookie-based via `express-session` backed by PostgreSQL (`connect-pg-simple`, table `user_sessions`). Auth tokens for email verification, password reset, and 2FA challenges are stored in `auth_tokens` / `two_factor_login_challenges`.

**Database:** PostgreSQL with full-text search. `transcript_segments`, `summaries`, and `action_items` all have `TSVECTOR` generated columns with GIN indexes powering the `/meetings/search` and `/meetings/ask` endpoints.

**Config validation:** All env vars are parsed and validated at startup with Zod in `config/env.ts`. The server will not start with invalid config.

### Frontend (`frontend/src/`)

- **Routing:** React Router v7. All authenticated routes wrap `<ProtectedRoute>` → `<AppLayout>`.
- **API layer:** `api.ts` — a thin typed wrapper around `axios` with `withCredentials: true`. All calls share a single axios instance with `VITE_API_BASE_URL` as base.
- **State:** Zustand (`store/`) for global client state; `@tanstack/react-query` (via hooks in `hooks/`) for server state. `useAuth` is the primary auth hook.
- **UI:** Chakra UI v3 + Tailwind CSS + `lucide-react` icons.
- **Forms:** `react-hook-form` + Zod resolvers.

### Infrastructure

- **Queue:** BullMQ over Redis. Jobs retry up to 3× with exponential backoff (5 s base). `meeting-processing` is the only queue.
- **File storage:** Supabase Storage, accessed via the AWS S3 SDK (`@aws-sdk/client-s3`) with `forcePathStyle: true`. Storage keys follow `{userId}/{meetingId}/{timestamp}-{filename}`.
- **Services in `docker-compose.yml`:** `postgres`, `redis`, `backend` (API), `worker` (BullMQ), `frontend`.
