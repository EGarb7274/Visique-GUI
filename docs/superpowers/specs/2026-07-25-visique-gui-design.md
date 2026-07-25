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
  Vercel serverless function, `frontend/api/analyze_xlsx.py`, which has a
  hard **4MB payload ceiling** (`MAX_PAYLOAD_SIZE`).
- Persistence and retrieval happen on the FastAPI backend, separately:
  `POST /api/v1/analysis/save` and `GET /api/v1/analysis/history` /
  `GET /api/v1/analysis/history/{id}`.
- Backend and Vercel each maintain their own copy of the CSV/XLSX parsers
  today (`docs/current_issues.md`, Issue 4) — a known drift risk. Visique-GUI
  should not add a third copy by parsing locally.
- Auth is email+password → OTP → JWT (`/api/v1/auth/login`,
  `/api/v1/auth/verify-otp`), documented in `backend/app/api/auth.py`.

## Architecture

Three Electron layers:

- **Main process** — window lifecycle, native file-open dialog, and a
  `safeStorage`-backed IPC bridge for token persistence (OS keychain, not
  `localStorage`). `contextIsolation: true`, `nodeIntegration: false`,
  matching the existing scaffold's `webPreferences` hardening.
- **Preload** — narrow `contextBridge` exposing only `openFileDialog()` and
  `getToken()` / `setToken()` / `clearToken()`.
- **Renderer** — React + TypeScript, built with Vite. Charts via `recharts`
  (already used by the Next.js frontend — no new charting-library risk).

All backend interaction goes through one module,
`src/renderer/api/visiqueClient.ts`, exposing exactly:

- `analyzeFile(file)` — POSTs to the Vercel analyze endpoint, returns an
  unsaved `StandardizedDataPackage`.
- `saveAnalysis(pkg)` — POSTs to FastAPI `/api/v1/analysis/save`, returns
  `{ id }`.
- `getHistory()` — GET FastAPI `/api/v1/analysis/history`.
- `getAnalysis(id)` — GET FastAPI `/api/v1/analysis/history/{id}`.

Base URLs are configurable via env vars with production defaults, mirroring
the scaffold's `VISIQUE_APP_URL` pattern. This module is the intentional
seam: if backend parsing is ever consolidated off Vercel onto FastAPI (the
recommended long-term direction — see "Analysis Path" decision below), only
`analyzeFile()`'s implementation changes; no UI code does.

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

- **(A, chosen)** React+TS renderer makes API calls directly via `fetch`;
  JWT persistence goes through a narrow IPC bridge to `safeStorage` instead
  of `localStorage`.
- (B) All network calls routed through the main process via IPC, so the
  JWT never enters the renderer's JS context at all. Stronger isolation,
  but every new API call requires wiring three places (main handler, preload
  bridge, renderer caller) instead of one `fetch` call. Worth revisiting if
  the renderer ever needs to run untrusted/third-party code; not a v1
  concern.
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
   POSTs to the Vercel function → returns an unsaved
   `StandardizedDataPackage`.
3. App immediately calls `saveAnalysis(pkg)` → FastAPI `/save` → returns
   `{ id }`.
4. App navigates to `DashboardScreen`, calls `getAnalysis(id)` → FastAPI
   `/history/{id}` → renders.
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

- Unit tests for `visiqueClient.ts` against mocked `fetch`.
- Component tests for `DashboardScreen` using a fixture
  `StandardizedDataPackage` (reuse an existing golden fixture from
  `backend/tests/data/` if the shape lines up).
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
