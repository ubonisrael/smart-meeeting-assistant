# Chrome Extension for Smart Meeting Assistant

## Context

Today the only way to get a meeting into the app is to already have a recording file and upload it by hand on `/upload`. That means recording a call by some other means first, then remembering to come back and upload it.

This adds a Chrome extension that records the meeting directly from the browser tab (Google Meet, Zoom web, Teams), mixes in your microphone, and uploads to the existing `POST /api/meetings/upload` endpoint — the same pipeline that already handles transcription, summary, and action items. The extension popup also carries a compact view of your meetings so you don't have to switch to the web app.

Decisions already made: **Chrome only** (MV3), **popup** as the primary surface, **reuse the existing session cookie** for auth, **WAV encoded in the extension**, **tab audio + microphone mixed**.

### Why WAV, specifically

Chrome's `MediaRecorder` produces WebM/Opus. Per the Gemini docs, supported audio types are WAV, MP3, AIFF, AAC, OGG Vorbis, and FLAC — `audio/webm` is **not** among them. Rather than add ffmpeg to the worker image or gamble on routing an audio-only WebM through Gemini's video path, the extension encodes 16 kHz mono 16-bit PCM WAV itself. `audio/wav` is already in the backend's `allowedMimeTypes` (`backend/src/controllers/meetingController.ts:17`), so **the entire backend pipeline is unchanged** apart from the cookie fix below.

The cost is size — ~1.9 MB/min, ~115 MB/hr — which drives the durability design below.

---

## Architecture

Everything lives inside the existing `frontend/` sub-repo so the popup can reuse the React app, `api.ts`, hooks, and Chakra theme directly. No new sub-repo, no duplicated API layer.

```
frontend/
├── extension/
│   ├── manifest.json          MV3 manifest (with pinned "key")
│   ├── background.ts          service worker — lifecycle, upload queue drain
│   ├── offscreen.html/.ts     capture, mix, encode  (the real work)
│   ├── recordingStore.ts      IndexedDB: chunks + recording metadata
│   ├── uploader.ts            upload with retry + transient/permanent classification
│   ├── pcm-worklet.js         AudioWorklet — copied verbatim, not bundled
│   ├── popup.html/.tsx        compact React popup
│   ├── options.html/.tsx      mic permission + max-duration setting
│   └── app.html               full app in a tab (same bundle as the web SPA)
├── vite.config.extension.ts   second build → dist-extension/
└── src/                       unchanged app, three small edits (below)
```

### Recording flow

1. Popup "Record this tab" → message to `background.ts`. The popup click is the required user gesture.
2. Background calls `chrome.tabCapture.getMediaStreamId({ targetTabId })`, ensures an offscreen document exists via `chrome.offscreen.createDocument({ reasons: ["USER_MEDIA"] })`, and forwards the stream id + tab title.
3. Offscreen builds **two** audio graphs from the tab stream:
   - **playback**: `new AudioContext()` at native rate → tab source → `destination`. Without this the tab goes silent for the user for the whole call — `tabCapture` steals the audio.
   - **capture**: `new AudioContext({ sampleRate: 16000 })` → tab source + mic source → merger → `pcm-worklet` → Int16 frames. Setting the context rate to 16000 makes the browser resample for us; no hand-rolled resampler.
4. Stop (from popup, or the stream's `ended` event when the tab closes) → assemble WAV, hand to the upload queue.
5. On `202`, clear state, badge off, notify. The existing worker pipeline takes it from there.

### Durability: never hold a meeting in memory

PCM is **never accumulated on the JS heap**. The worklet's frames are batched into ~5-second `Blob`s and written straight to IndexedDB (`recordingStore.ts`), with a `recordings` record tracking `{ id, tabTitle, startedAt, bytes, status }`. At stop, chunks are read back and assembled as `new Blob([wavHeader, ...chunks])` — browser-managed and disk-backed, so a two-hour call costs kilobytes of heap rather than 230 MB.

This also makes recordings survive a crash: if the offscreen document or the browser dies mid-call, the chunks are already on disk. On `chrome.runtime.onStartup` the background worker scans for records left in `recording` or `pending-upload` and finishes them — a killed browser costs you the tail of the audio, not the meeting.

A **max duration guard** (default 2 hours, configurable on the options page) auto-stops and uploads rather than growing without bound. The popup shows elapsed time and running size so a long call is visible before it becomes a problem.

### Upload reliability

The browser→backend leg is one large multipart POST, and this machine has shown ~1-in-8 TLS record corruption on multi-MB transfers. Two mitigations, no new backend endpoints:

- **Retry with backoff in `uploader.ts`**, mirroring the classification already used in `backend/src/services/storageService.ts` (`isTransientStorageError`): network errors, 5xx, 408, 429 are transient; 400/413 are permanent; **401 is treated as "needs auth"**, not failure.
- **A durable queue.** The recording stays in IndexedDB until the server returns `202`. Failed uploads sit at `pending-upload` and are re-drained by a `chrome.alarms` tick (every 5 min), on popup open, and on browser startup. The popup surfaces "1 recording waiting to upload — Retry now", and for the 401 case, "Sign in to finish uploading". A failed upload therefore never destroys audio, which matters far more than making any single attempt succeed.

Resumable/chunked upload would need new backend endpoints and is deliberately out of scope; the durable queue covers the failure mode meanwhile.

### State

The popup closes on blur, so it owns no recording state. `chrome.storage.local` holds the active `{ recordingId, tabId, tabTitle, startedAt }`; the popup renders a timer from `startedAt`. `chrome.action.setBadgeText({ text: "REC" })` shows status when the popup is shut, and a distinct badge marks pending uploads.

### Stable extension ID

`CLIENT_ORIGIN` and `host_permissions` both reference `chrome-extension://<id>`, so the ID must not change between loads. Pin it by committing a `"key"` in `manifest.json`:

```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out extension-key.pem   # keep OUT of git
openssl rsa -in extension-key.pem -pubout -outform DER | base64 -w0          # → manifest "key"
```

The public key in `manifest.json` is safe to commit and fixes the ID across machines and reloads; `extension-key.pem` is only needed for `.crx` packaging and goes in `.gitignore`. The derived ID gets written into `backend/.env.example` as a comment next to `CLIENT_ORIGIN`.

---

## Files to create

| File | Purpose |
|---|---|
| `frontend/extension/manifest.json` | MV3. Permissions: `tabCapture`, `offscreen`, `storage`, `alarms`, `activeTab`, `notifications`. `host_permissions` for the API origin. Pinned `"key"`. `action.default_popup`, `options_page`, `background.service_worker` (`"type": "module"`). |
| `frontend/extension/background.ts` | Message router; `getMediaStreamId`; offscreen create/close; badge; alarm-driven queue drain; `onStartup` recovery. |
| `frontend/extension/offscreen.ts` | Capture/mix graphs, chunk batching to IndexedDB, WAV header writer, duration guard. |
| `frontend/extension/recordingStore.ts` | IndexedDB wrapper: `appendChunk`, `listPending`, `assembleBlob`, `discard`. |
| `frontend/extension/uploader.ts` | `FormData` POST with `credentials: "include"`, retry/backoff, error classification. |
| `frontend/extension/pcm-worklet.js` | `AudioWorkletProcessor` posting Float32 frames. Copied as a static asset — loaded by URL, not imported. |
| `frontend/extension/popup.tsx` | Record/stop + elapsed + size, pending-upload banner, `MeetingList` (reuse `src/components/meetings/MeetingList.tsx`), "Open full app" → `chrome.tabs.create({ url: chrome.runtime.getURL("app.html") })`. |
| `frontend/extension/options.tsx` | One-time `getUserMedia({ audio: true })` grant — offscreen documents cannot show the mic prompt, which is the only reason this page exists — plus the max-duration setting. |
| `frontend/vite.config.extension.ts` | `outDir: dist-extension`, multi-entry (`popup`, `options`, `offscreen`, `app`, `background`), `entryFileNames` pinned so `background.js` keeps a stable path, static copy of `manifest.json` + `pcm-worklet.js`. |

## Files to modify

**`frontend/src/components/ui/Provider.tsx`** — hardcodes `BrowserRouter`, which cannot work from `chrome-extension://`. Accept the router as a prop so popup/app entries pass `HashRouter` while `main.tsx` keeps `BrowserRouter`.

**`frontend/src/api.ts`** — the 401 interceptor does `window.location.replace("/login")`, which in an extension resolves to a nonexistent `chrome-extension://<id>/login`. Make the redirect target injectable, or detect `location.protocol === "chrome-extension:"` and use `#/login`.

**`frontend/package.json`** — add `build:extension` and `dev:extension`.

**`backend/src/config/session.ts`** — the one required backend change. The cookie is `sameSite: "lax"` outside production (line 23), so Chrome will **not** send it on requests originating from the extension, which are cross-site. Needs `SameSite=None; Secure`. Chrome treats `http://localhost` as trustworthy, so `Secure` still works in dev. Drive it from new env vars (`SESSION_COOKIE_SAMESITE`, `SESSION_COOKIE_SECURE`) validated in `backend/src/config/env.ts` rather than hardcoding.

**`backend/.env` / `.env.example`** — the new cookie vars, plus `chrome-extension://<id>` appended to `CLIENT_ORIGIN`. With `host_permissions` granted, MV3 exempts the extension's own requests from CORS, so this is belt-and-braces rather than strictly required.

### Optional hardening

`meetingController.ts:41` compares `req.file.mimetype` by exact match, so `audio/wav;codecs=1` would be rejected. Stripping parameters first is a two-line fix — not required here, since we set the Blob type to exactly `audio/wav`.

---

## Verification

1. `cd frontend && npm run build:extension`, load `dist-extension/` unpacked at `chrome://extensions`. Reload it twice and confirm **the ID does not change**.
2. Copy the ID into `CLIENT_ORIGIN`, set the cookie env vars, restart the backend. Confirm the **web** app at `:5173` can still log in — the cookie change affects it too.
3. Open the options page, grant the mic.
4. Log in through the popup; the meetings list rendering proves cookie auth works cross-origin.
5. Record ~30 s of a tab with audio while speaking into the mic.
   - **Confirm the tab is still audible during capture** — the single easiest thing to get wrong.
6. Close the popup mid-recording, reopen: timer still running, badge shows `REC`.
7. Stop. Expect `202`, a `meeting.upload.received` log with `mimeType: audio/wav`, then the worker progressing `uploaded → transcribing → … → completed`. Open the meeting in the web app and confirm the transcript contains **both** the tab audio and your own voice.
8. Durability checks:
   - **Backend down:** stop the backend, record, stop. Expect the popup to show a pending upload; restart the backend and confirm the alarm drain (or "Retry now") completes it.
   - **Signed out:** clear the session cookie and stop a recording. Expect "Sign in to finish uploading", and completion after logging in — not a discarded recording.
   - **Crash recovery:** record for a minute, kill the offscreen document from `chrome://extensions` (or restart Chrome), and confirm `onStartup` finds the partial recording and uploads it.
   - **Tab closed mid-call:** should auto-stop and still upload.
   - **Duration guard:** set max duration to 1 minute in options and confirm auto-stop and upload.
9. Record a ~20 min call and watch Chrome's task manager — heap should stay flat, confirming chunks are going to disk rather than memory.

---

## Risks

- **Popup size ceiling.** Chrome caps popups at 800×600, so the popup carries record controls, the pending-upload banner, and the list only; the full two-column meeting UI lives in the tab view via "Open full app".
- **Not covered:** Firefox/Safari (no `chrome.offscreen`/`tabCapture` equivalents), Chrome Web Store packaging and review, and speaker diarization to separate your voice from the tab audio — both sources land in one mixed mono track.
