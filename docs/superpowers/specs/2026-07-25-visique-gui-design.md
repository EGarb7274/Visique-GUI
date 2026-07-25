# Visique-GUI — Design Spec (v1)

Date: 2026-07-25

## Purpose

Visique-GUI is a standalone native Electron + TypeScript desktop client for
the Visique financial-analysis platform. A user picks an Excel (`.xlsx`)
file, it's analyzed by Visique's existing backend services, and the result is
rendered as a local dashboard (health score, KPIs, trend charts) — without
embedding the Visique website.

This is distinct from the existing `deploy/electron-scaffold` in
`Visique-Testing`, which is a demo-grade wrapper that simply loads the
deployed Next.js website (`https://visique.xyz/desktop-entry`) inside a
BrowserWindow. That scaffold has no independent UI code, no direct API
calls, and is explicitly labeled unsigned/non-notarized/demo-only. Long-term,
Visique-GUI is intended to replace that scaffold as the real downloadable
desktop product; it should borrow its packaging setup (electron-builder
config, mac/win icons in `resources/`) rather than reinvent it, once
Visique-GUI reaches feature parity.

## Context / Constraints Discovered

- The FastAPI backend's own `/api/v1/analysis/upload/xlsx` endpoint is
  **deprecated** (returns HTTP 410). The real parsing/analysis path is a
  Vercel serverless function, `frontend/api/analyze_xlsx.py`, exposed at
  `/api/analyze/xlsx` (see `frontend/vercel.json` rewrites), which has a
  hard **4MB payload ceiling** (`MAX_PAYLOAD_SIZE`).
- Retrieval happens on the FastAPI backend: `GET /api/v1/analysis/history`
  and `GET /api/v1/analysis/history/{id}`.
- **Persistence does NOT happen by calling the FastAPI backend's `/save`
  directly.** The raw endpoint (`POST /api/v1/analysis/save`) requires an
  HMAC `X-Service-Signature` header whenever `SERVICE_SECRET_KEY` is set on
  the backend — and it is, in production (`backend/render.yaml` auto-generates
  it). That secret must never be embedded in client software. The real save
  path is a Vercel proxy, `frontend/api/save_analysis.py`, exposed at
  `/api/save` (see `frontend/vercel.json` rewrites): it holds
  `SERVICE_SECRET_KEY` server-side, signs the payload, and forwards it to
  the Render backend with both the JWT and the signature attached. Clients
  (including Visique-GUI) must call `/api/save`, never the raw backend
  endpoint.
- Both Vercel functions (`analyze_xlsx.py`, `save_analysis.py`) hand-roll
  their own CORS with a hardcoded origin allowlist (`visique.xyz`,
  `visique-testing.vercel.app`, `localhost:3000`) that does **not** include
  Electron's `file://` origin (packaged app) or a dev server port like
  Vite's `localhost:5173`. A renderer calling `fetch()` directly against
  either endpoint would be blocked by CORS in both dev and packaged builds.
  (The main FastAPI backend's own CORS config is more permissive — it
  allows `file://.*` via regex — but that doesn't help here since these two
  calls target the Vercel functions, not the backend directly.)
- Backend and Vercel each maintain their own copy of the CSV/XLSX parsers
  today (`docs/current_issues.md`, Issue 4) — a known drift risk. Visique-GUI
  should not add a third copy by parsing locally.
- Auth is email+password → OTP → JWT (`/api/v1/auth/login`,
  `/api/v1/auth/verify-otp`), documented in `backend/app/api/auth.py`.

## Architecture

Three Electron layers:

- **Main process** — window lifecycle, native file-open dialog, a
  `safeStorage`-backed IPC bridge for token persistence (OS keychain, not
  `localStorage`), and executes the actual HTTP requests to Vercel/FastAPI
  (see correction below — this sidesteps a real CORS restriction on the
  Vercel endpoints). `contextIsolation: true`, `nodeIntegration: false`,
  matching the existing scaffold's `webPreferences` hardening.
- **Preload** — narrow `contextBridge` exposing `openFileDialog()`,
  `getToken()` / `setToken()` / `clearToken()`, and `apiRequest(method, url,
  body, token)`.
- **Renderer** — React + TypeScript, built with Vite. Charts via `recharts`
  (already used by the Next.js frontend — no new charting-library risk).

All backend interaction goes through one renderer-facing module,
`src/renderer/api/visiqueClient.ts`, exposing exactly:

- `analyzeFile(file)` — sends the file to `/api/analyze/xlsx` (Vercel),
  returns an unsaved `StandardizedDataPackage`.
- `saveAnalysis(pkg)` — sends the package to `/api/save` (Vercel proxy —
  **not** the raw FastAPI `/api/v1/analysis/save`, which requires an
  HMAC signature this client must never hold), returns `{ id }`.
- `getHistory()` — GET FastAPI `/api/v1/analysis/history`.
- `getAnalysis(id)` — GET FastAPI `/api/v1/analysis/history/{id}`.

**Correction after investigation:** these four functions do not call
`fetch()` themselves. Both Vercel endpoints (`/api/analyze/xlsx`,
`/api/save`) enforce a hardcoded CORS origin allowlist that excludes
Electron's `file://` origin and dev-server ports — a renderer-side `fetch()`
would be blocked in both dev and packaged builds. Instead, each function in
`visiqueClient.ts` calls a single generic IPC channel
(`window.visique.apiRequest(method, url, body, token)`) exposed by the
preload script. The main process executes the actual HTTP request in its
Node context, which isn't subject to browser CORS at all. This keeps the
renderer-facing API surface identical to a direct-fetch design (one module,
four functions) while avoiding the CORS gap entirely — a lighter version of
"Approach B" applied only to the mechanics of sending a request, not to
per-feature business logic.

Base URLs are configurable via env vars with production defaults, mirroring
the scaffold's `VISIQUE_APP_URL` pattern. This module is the intentional
seam: if backend parsing is ever consolidated off Vercel onto FastAPI (the
recommended long-term direction — see "Analysis Path" decision below), only
`analyzeFile()`'s target URL changes; no UI code does.

**Auth is mocked for v1**: a fixed fake user/token in an `AuthContext`,
built behind the same client-module boundary so the real login flow
(`/login` → OTP → `/verify-otp`) drops in later without touching UI code.

### Decision: Analysis path (Vercel now, not local parsing)

Three options were considered for how Visique-GUI performs XLSX
parsing/analysis:

1. **Call the existing Vercel serverless endpoint** (chosen for v1) — zero
   new backend risk, reuses tested code, matches what the web app does today.
2. Parse locally inside the Electron app (bundle/port `financial_model`) —
   rejected: would create a *third* copy of the parser alongside the
   backend's and Vercel's existing two, worsening the drift problem
   `docs/current_issues.md` already flags.
3. Add/restore a real endpoint on the FastAPI backend — the correct
   long-term fix (one canonical parser, no Vercel payload-size ceiling), but
   out of scope for the GUI project; it's a backend change. The
   `visiqueClient.ts` seam above makes this a future one-line swap.

### Decision: Electron internal structure

Three structures were considered:

- **(A, chosen, refined)** React+TS renderer's `visiqueClient.ts` calls a
  single generic IPC channel (`apiRequest`) rather than `fetch()` directly;
  the main process executes the HTTP request. This was refined after
  investigation showed the Vercel endpoints' hardcoded CORS allowlist would
  block a renderer-side `fetch()` outright (see "Context / Constraints
  Discovered"). JWT persistence goes through the same IPC bridge to
  `safeStorage` instead of `localStorage`.
- (B) All network calls routed through the main process via per-feature IPC
  handlers (a distinct main handler + preload method for each of
  analyze/save/history), so no business logic lives in the renderer at all.
  Rejected as unnecessary: the single generic `apiRequest` channel already
  gets the CORS and token-isolation benefits without per-feature plumbing.
  Worth revisiting only if the renderer ever needs to run untrusted/third-
  party code.
- (C) Vanilla TS renderer, no framework. Rejected: a dashboard + history
  list + growing feature set benefits from componentization; hand-rolling
  it manually doesn't pay off since bundle size isn't a real constraint for
  a desktop app.

## Components / Screens

- `AppShell` — top-level nav between Upload and History. No login screen in
  v1 (auth is mocked).
- `UploadScreen` — file picker + drag-and-drop. Client-side validation:
  `.xlsx` extension check, and a size check against the Vercel wrapper's
  4MB ceiling — reject oversized files before sending, with a clear
  message, rather than letting the request fail server-side.
- `DashboardScreen` — renders a `StandardizedDataPackage`: health score
  card, KPI grid (margins, liquidity ratios), 1–2 trend charts via
  `recharts`. Deliberately a subset of the full web dashboard's widgets —
  enough for v1, not full parity.
- `HistoryScreen` — table from `getHistory()`; selecting a row calls
  `getAnalysis(id)` and opens `DashboardScreen` for that record.
- `types.ts` — hand-ported TS interfaces for only the fields v1 renders
  (health score, KPI subset, trend series), ported from
  `financial_model/models.py`'s `StandardizedDataPackage` — not the full
  schema.

## Data Flow

1. App launches → mock-authenticated immediately → lands on `UploadScreen`.
2. User picks/drops an `.xlsx` → client-side validation → `analyzeFile()`
   → IPC `apiRequest` → main process POSTs to Vercel's `/api/analyze/xlsx`
   → returns an unsaved `StandardizedDataPackage`.
3. App immediately calls `saveAnalysis(pkg)` → IPC `apiRequest` → main
   process POSTs to Vercel's `/api/save` (proxy, HMAC-signed server-side,
   never the raw FastAPI endpoint) → returns `{ id }`.
4. App navigates to `DashboardScreen`, calls `getAnalysis(id)` → IPC
   `apiRequest` → main process GETs FastAPI `/history/{id}` → renders.
5. `HistoryScreen` calls `getHistory()` on demand; selecting a past entry
   re-fetches via `getAnalysis(id)` and reuses `DashboardScreen`.

## Error Handling

- Network/API failures surface as an inline banner on the current screen —
  never a silent failure.
- If a returned package's `analysis_status` is `"degraded"` or
  `"quarantined"` (a status the backend already computes), show a warning
  banner explaining which features are suppressed. The GUI only needs to
  display this flag, not detect data-quality issues itself.
- If `analyzeFile()` succeeds but `saveAnalysis()` fails (e.g., network
  drop), keep the parsed package in memory and offer a "retry save" action
  rather than forcing the user to re-upload and re-parse.

## Testing

- Unit tests for `visiqueClient.ts` against a mocked
  `window.visique.apiRequest` (not `fetch` — see the IPC correction above).
- Unit tests for the main process's IPC handler that executes `apiRequest`,
  against a mocked Node `fetch`.
- Component tests for `DashboardScreen` using a fixture
  `StandardizedDataPackage`. `backend/tests/data/` only contains PDF
  fixtures (`apple_2024_10k.pdf`, `sampledata.pdf`) — no ready-made JSON
  fixture matching this shape exists, so v1 creates its own synthetic
  fixture conforming to the `StandardizedDataPackage`/`types.ts` shape.
- Manual smoke test before calling v1 done: launch the app, upload a real
  sample `.xlsx`, confirm the dashboard renders end-to-end against the live
  Vercel/FastAPI services.

## Explicitly Out of Scope for v1

- Real login (mocked for now; real JWT/OTP flow is a follow-up).
- Exports (PDF/PPTX/CSV).
- AI CFO chat / Helios integration.
- Offline/local parsing.
- Code signing/notarization and distribution packaging (deferred until
  Visique-GUI is ready to replace the electron-scaffold).
