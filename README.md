# Smart Meeting Assistant

Smart Meeting Assistant is a full-stack app for authenticated meeting uploads, background transcription, AI summaries, action item extraction, and meeting search.

Calendar integration is intentionally deferred until the core demo flow is complete.

## Repositories

This workspace contains the orchestration files and three service repositories:

- `frontend`: React, TypeScript, TailwindCSS
- `backend`: Express, PostgreSQL, Redis, BullMQ
- `transcription-service`: FastAPI wrapper around Faster-Whisper

## Quick Start

1. Copy `.env.example` to `.env`.
2. Fill in Supabase Storage S3 credentials and, optionally, `GEMINI_API_KEY`.
3. Run the development stack:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

This uses the development Dockerfiles and host-mounted source/data directories.

If backend or frontend dependencies change while using the development stack, refresh the matching `node_modules` volume:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm backend npm install
docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm frontend npm install
```

For the non-development stack, run:

```bash
docker compose up --build
```

4. Open the frontend at `http://localhost:5173`.
5. Backend health check: `http://localhost:4000/health`.

## MVP Flow

1. Register or log in.
2. Upload a meeting recording.
3. The backend stores the file in Supabase Storage and returns `processing`.
4. BullMQ queues the processing job.
5. The worker sends the recording to the Faster-Whisper service.
6. The worker stores transcript segments, generates a summary, extracts action items, and marks the meeting complete.
7. Search or ask questions across your meetings.
