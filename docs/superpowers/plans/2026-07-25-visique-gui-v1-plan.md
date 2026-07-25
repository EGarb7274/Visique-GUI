# Visique-GUI v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build v1 of Visique-GUI — an Electron + TypeScript desktop app where a user picks an `.xlsx` file, it's analyzed and saved via Visique's existing Vercel/FastAPI services, and the result renders as a local dashboard, with a history list of past analyses.

**Architecture:** Electron main/preload/renderer split. Renderer is React + TypeScript (Vite via `electron-vite`), charts via `recharts`. All actual HTTP calls execute in the main process (Node `fetch`) behind two IPC channels (`api:request` for JSON calls, `api:uploadFile` for the multipart analyze call) — this sidesteps a real CORS restriction on the Vercel endpoints that would block renderer-side `fetch()`. Auth is mocked (fixed token) for v1.

**Tech Stack:** Electron ^33, electron-vite ^2, React ^18, TypeScript ^5 (strict), Vite ^5, recharts ^3.7.0, Vitest ^2, @testing-library/react ^16.

## Global Constraints

- Node/Electron: Electron 33 bundles a Node runtime with native `fetch`, `FormData`, and `Blob` in the main process — no extra HTTP client library needed.
- TypeScript strict mode (`"strict": true`) in all `tsconfig*.json`.
- No runtime dependencies beyond `electron`, `react`, `react-dom`, `recharts`. No new dependency may be added without updating this plan.
- **Never call FastAPI's raw `/api/v1/analysis/save` directly.** Always call the Vercel proxy at `/api/save`. The raw endpoint requires an HMAC `X-Service-Signature` signed with `SERVICE_SECRET_KEY`, a server-only secret that must never exist in this repo or this app's bundle.
- All backend base URLs are configurable via Vite env vars (`VITE_VISIQUE_WEB_BASE_URL` for the Vercel-hosted functions, `VITE_VISIQUE_API_BASE_URL` for the FastAPI backend) with hardcoded production defaults as fallback (`https://visique.xyz`, `https://visique-backend.onrender.com`).
- Every `BrowserWindow` uses `contextIsolation: true`, `nodeIntegration: false`, `sandbox: false` — matches `Visique-Testing/deploy/electron-scaffold/main.js`.
- File layout follows `electron-vite`'s convention: `src/main/`, `src/preload/`, `src/renderer/` (with `src/renderer/src/` for the React app itself), plus `src/shared/` for code imported by more than one process.

---

### Task 1: Project scaffolding

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `tsconfig.node.json`
- Create: `electron.vite.config.ts`
- Create: `.gitignore`
- Create: `src/main/index.ts`
- Create: `src/preload/index.ts`
- Create: `src/renderer/index.html`
- Create: `src/renderer/src/main.tsx`
- Create: `src/renderer/src/App.tsx`

**Interfaces:**
- Produces: an `electron-vite` project that builds to `out/main/index.js`, `out/preload/index.js`, `out/renderer/index.html`, and a `npm run dev` command that opens a blank window. Later tasks replace the placeholder `App.tsx` content.

- [ ] **Step 1: Write `package.json`**

```json
{
  "name": "visique-gui",
  "version": "0.1.0",
  "private": true,
  "main": "out/main/index.js",
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build",
    "preview": "electron-vite preview",
    "typecheck": "tsc --noEmit -p tsconfig.json && tsc --noEmit -p tsconfig.node.json",
    "test": "vitest run"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "recharts": "^3.7.0"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "^6.6.3",
    "@testing-library/react": "^16.0.1",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.4",
    "electron": "^33.4.11",
    "electron-vite": "^2.3.0",
    "jsdom": "^25.0.1",
    "typescript": "^5.7.2",
    "vite": "^5.4.11",
    "vitest": "^2.1.8"
  }
}
```

- [ ] **Step 2: Write `tsconfig.node.json`** (main + preload)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["electron", "node"]
  },
  "include": ["src/main/**/*.ts", "src/preload/**/*.ts", "src/shared/**/*.ts"]
}
```

- [ ] **Step 3: Write `tsconfig.json`** (renderer)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["vite/client"]
  },
  "include": ["src/renderer/src/**/*.ts", "src/renderer/src/**/*.tsx", "src/shared/**/*.ts"]
}
```

- [ ] **Step 4: Write `electron.vite.config.ts`**

```ts
import { defineConfig, externalizeDepsPlugin } from 'electron-vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  main: {
    plugins: [externalizeDepsPlugin()],
  },
  preload: {
    plugins: [externalizeDepsPlugin()],
  },
  renderer: {
    root: 'src/renderer',
    plugins: [react()],
  },
});
```

- [ ] **Step 5: Write `.gitignore`**

```
node_modules/
out/
dist/
*.log
.env.local
```

- [ ] **Step 6: Write `src/main/index.ts`** (placeholder — Task 5 replaces this with IPC wiring)

```ts
import { app, BrowserWindow } from 'electron';
import { join } from 'node:path';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    show: false,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: false,
    },
  });

  mainWindow.on('ready-to-show', () => mainWindow.show());

  if (process.env['ELECTRON_RENDERER_URL']) {
    void mainWindow.loadURL(process.env['ELECTRON_RENDERER_URL']);
  } else {
    void mainWindow.loadFile(join(__dirname, '../renderer/index.html'));
  }
}

app.whenReady().then(() => {
  createWindow();

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
```

- [ ] **Step 7: Write `src/preload/index.ts`** (placeholder — Task 6 replaces this)

```ts
// Placeholder — Task 6 exposes the window.visique bridge here.
export {};
```

- [ ] **Step 8: Write `src/renderer/index.html`**

```html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Visique</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 9: Write `src/renderer/src/App.tsx`** (placeholder — Task 14 replaces this)

```tsx
export function App() {
  return <div>Visique-GUI</div>;
}
```

- [ ] **Step 10: Write `src/renderer/src/main.tsx`**

```tsx
import { createRoot } from 'react-dom/client';
import { App } from './App';

const container = document.getElementById('root');
if (!container) throw new Error('Root element not found');
createRoot(container).render(<App />);
```

- [ ] **Step 11: Install dependencies and verify the build**

```bash
npm install
npm run build
```

Expected: both commands exit 0, and `out/main/index.js`, `out/preload/index.js`, `out/renderer/index.html` exist.

- [ ] **Step 12: Commit**

```bash
git add package.json tsconfig.json tsconfig.node.json electron.vite.config.ts .gitignore src
git commit -m "feat: scaffold electron-vite project"
```

---

### Task 2: Shared types and test fixture

**Files:**
- Create: `src/shared/types.ts`
- Create: `test/fixtures/sample-package.json`
- Test: `test/shared/types.test.ts`

**Interfaces:**
- Produces: `StandardizedDataPackage`, `HealthScoreBreakdown`, `KPIMetrics`, `RiskAnalysis`, `FinancialReportSummary`, `AnalysisStatus`, `AnalysisHistoryEntry`, `PickedFile`, `ApiRequestArgs`, `ApiResponse`, `UploadFileArgs` — used by every subsequent task.

- [ ] **Step 1: Write `src/shared/types.ts`**

```ts
// --- Domain types (subset of Visique-Testing's financial_model/models.py
// StandardizedDataPackage — only the fields v1 renders) ---

export type AnalysisStatus = 'reliable' | 'degraded' | 'quarantined';

export interface FinancialReportSummary {
  company_name: string;
  period_end: string;
  period_type: string;
}

export interface KPIMetrics {
  current_ratio: number | null;
  quick_ratio: number | null;
  gross_margin: number | null;
  operating_margin: number | null;
  net_margin: number | null;
  ebitda_margin: number | null;
  debt_to_equity: number | null;
  roa: number | null;
  roe: number | null;
}

export interface RiskAnalysis {
  risk_score: number;
  risk_factors: string[];
  liquidity_risk: string;
  solvency_risk: string;
}

export interface HealthScoreBreakdown {
  stability: number;
  profitability: number;
  growth: number;
  efficiency: number;
  total_score: number;
}

export interface StandardizedDataPackage {
  raw_data: FinancialReportSummary;
  kpis: KPIMetrics;
  risk_analysis: RiskAnalysis;
  health_score: HealthScoreBreakdown;
  insights: string[];
  analysis_status: AnalysisStatus;
  analysis_id?: number;
  timestamp?: string;
}

export interface AnalysisHistoryEntry {
  id: number;
  company_name: string;
  filename: string;
  timestamp: string;
  runner_name: string;
}

// --- IPC contract types (shared between preload, main, and renderer) ---

export interface PickedFile {
  path: string;
  name: string;
  size: number;
}

export interface ApiRequestArgs {
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  url: string;
  token?: string | null;
  json?: unknown;
}

export interface UploadFileArgs {
  url: string;
  token?: string | null;
  filePath: string;
  fileName: string;
}

export interface ApiResponse {
  status: number;
  ok: boolean;
  body: unknown;
}
```

- [ ] **Step 2: Write `test/fixtures/sample-package.json`**

`backend/tests/data/` in Visique-Testing only has PDF fixtures — no ready-made JSON matching this shape exists, so this is a hand-built synthetic fixture.

```json
{
  "raw_data": {
    "company_name": "Acme Robotics, Inc.",
    "period_end": "2025-12-31",
    "period_type": "annual"
  },
  "kpis": {
    "current_ratio": 1.8,
    "quick_ratio": 1.2,
    "gross_margin": 0.42,
    "operating_margin": 0.15,
    "net_margin": 0.09,
    "ebitda_margin": 0.21,
    "debt_to_equity": 0.65,
    "roa": 0.11,
    "roe": 0.18
  },
  "risk_analysis": {
    "risk_score": 32,
    "risk_factors": ["Elevated accounts receivable days"],
    "liquidity_risk": "Low",
    "solvency_risk": "Moderate"
  },
  "health_score": {
    "stability": 74,
    "profitability": 68,
    "growth": 81,
    "efficiency": 70,
    "total_score": 73
  },
  "insights": [
    "Revenue grew 18% year-over-year, outpacing operating expense growth.",
    "Gross margin improved 3 points versus the prior period."
  ],
  "analysis_status": "reliable",
  "analysis_id": 101,
  "timestamp": "2026-07-20T14:32:00Z"
}
```

- [ ] **Step 3: Write the failing test**

```ts
// test/shared/types.test.ts
import { describe, it, expect } from 'vitest';
import type { StandardizedDataPackage } from '../../src/shared/types';
import fixture from '../fixtures/sample-package.json';

describe('sample-package fixture', () => {
  it('matches the StandardizedDataPackage shape', () => {
    const pkg: StandardizedDataPackage = fixture as StandardizedDataPackage;
    expect(pkg.raw_data.company_name).toBe('Acme Robotics, Inc.');
    expect(pkg.health_score.total_score).toBe(73);
    expect(pkg.analysis_status).toBe('reliable');
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `npx vitest run test/shared/types.test.ts`
Expected: FAIL — `Cannot find module '../../src/shared/types'` (file doesn't exist yet if Step 1 hasn't run) or module resolution error. Run this before Step 1's file is saved, or simply confirm the test passes only after both Step 1 and Step 2 are in place (this task's "red" state is "files don't exist yet").

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run test/shared/types.test.ts`
Expected: PASS (1 test).

- [ ] **Step 6: Commit**

```bash
git add src/shared/types.ts test/fixtures/sample-package.json test/shared/types.test.ts
git commit -m "feat: add shared domain/IPC types and test fixture"
```

---

### Task 3: Secure token storage (main process)

**Files:**
- Create: `src/main/tokenStore.ts`
- Test: `test/main/tokenStore.test.ts`

**Interfaces:**
- Consumes: none.
- Produces: `class TokenStore { constructor(storageDir: string); setToken(token: string): void; getToken(): string | null; clearToken(): void; }` — used by Task 5 (IPC registration).

- [ ] **Step 1: Write the failing test**

```ts
// test/main/tokenStore.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

vi.mock('electron', () => ({
  safeStorage: {
    isEncryptionAvailable: () => true,
    encryptString: (s: string) => Buffer.from(`enc:${s}`),
    decryptString: (b: Buffer) => b.toString('utf-8').replace(/^enc:/, ''),
  },
}));

import { TokenStore } from '../../src/main/tokenStore';

describe('TokenStore', () => {
  let dir: string;

  beforeEach(() => {
    dir = mkdtempSync(join(tmpdir(), 'visique-gui-test-'));
  });

  afterEach(() => {
    rmSync(dir, { recursive: true, force: true });
  });

  it('returns null when no token has been set', () => {
    const store = new TokenStore(dir);
    expect(store.getToken()).toBeNull();
  });

  it('round-trips a token through set/get', () => {
    const store = new TokenStore(dir);
    store.setToken('jwt-abc-123');
    expect(store.getToken()).toBe('jwt-abc-123');
  });

  it('clears a stored token', () => {
    const store = new TokenStore(dir);
    store.setToken('jwt-abc-123');
    store.clearToken();
    expect(store.getToken()).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/main/tokenStore.test.ts`
Expected: FAIL with "Cannot find module '../../src/main/tokenStore'".

- [ ] **Step 3: Write `src/main/tokenStore.ts`**

```ts
import { safeStorage } from 'electron';
import { existsSync, readFileSync, rmSync, writeFileSync } from 'node:fs';
import { join } from 'node:path';

export class TokenStore {
  private readonly filePath: string;

  constructor(storageDir: string) {
    this.filePath = join(storageDir, 'token.enc');
  }

  setToken(token: string): void {
    if (!safeStorage.isEncryptionAvailable()) {
      throw new Error('Secure storage is not available on this system.');
    }
    writeFileSync(this.filePath, safeStorage.encryptString(token));
  }

  getToken(): string | null {
    if (!existsSync(this.filePath)) return null;
    return safeStorage.decryptString(readFileSync(this.filePath));
  }

  clearToken(): void {
    if (existsSync(this.filePath)) rmSync(this.filePath);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/main/tokenStore.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/tokenStore.ts test/main/tokenStore.test.ts
git commit -m "feat: add safeStorage-backed token store"
```

---

### Task 4: Main-process request execution (JSON + file upload)

**Files:**
- Create: `src/main/apiRequestHandler.ts`
- Create: `src/main/uploadFileHandler.ts`
- Test: `test/main/apiRequestHandler.test.ts`
- Test: `test/main/uploadFileHandler.test.ts`

**Interfaces:**
- Consumes: `ApiRequestArgs`, `UploadFileArgs`, `ApiResponse` from `src/shared/types.ts` (Task 2).
- Produces: `executeApiRequest(args: ApiRequestArgs, fetchImpl?: typeof fetch): Promise<ApiResponse>` and `executeUploadFile(args: UploadFileArgs, fetchImpl?: typeof fetch): Promise<ApiResponse>` — used by Task 5 (IPC registration).

- [ ] **Step 1: Write the failing test for `apiRequestHandler`**

```ts
// test/main/apiRequestHandler.test.ts
import { describe, it, expect, vi } from 'vitest';
import { executeApiRequest } from '../../src/main/apiRequestHandler';

describe('executeApiRequest', () => {
  it('sends a Bearer token and JSON body, and parses a JSON response', async () => {
    const fetchMock = vi.fn(async (_url: string, init?: RequestInit) => {
      expect(init?.method).toBe('POST');
      expect((init?.headers as Record<string, string>).Authorization).toBe('Bearer tok-1');
      expect((init?.headers as Record<string, string>)['Content-Type']).toBe('application/json');
      expect(init?.body).toBe(JSON.stringify({ hello: 'world' }));
      return new Response(JSON.stringify({ id: 42 }), { status: 200 });
    });

    const result = await executeApiRequest(
      { method: 'POST', url: 'https://example.com/save', token: 'tok-1', json: { hello: 'world' } },
      fetchMock as unknown as typeof fetch,
    );

    expect(result).toEqual({ status: 200, ok: true, body: { id: 42 } });
  });

  it('omits the Authorization header when no token is provided', async () => {
    const fetchMock = vi.fn(async (_url: string, init?: RequestInit) => {
      expect((init?.headers as Record<string, string>).Authorization).toBeUndefined();
      return new Response('[]', { status: 200 });
    });

    const result = await executeApiRequest(
      { method: 'GET', url: 'https://example.com/history' },
      fetchMock as unknown as typeof fetch,
    );

    expect(result).toEqual({ status: 200, ok: true, body: [] });
  });

  it('surfaces non-2xx responses with ok: false', async () => {
    const fetchMock = vi.fn(
      async () => new Response(JSON.stringify({ detail: 'Not found' }), { status: 404 }),
    );

    const result = await executeApiRequest(
      { method: 'GET', url: 'https://example.com/history/9', token: 'tok-1' },
      fetchMock as unknown as typeof fetch,
    );

    expect(result).toEqual({ status: 404, ok: false, body: { detail: 'Not found' } });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/main/apiRequestHandler.test.ts`
Expected: FAIL with "Cannot find module '../../src/main/apiRequestHandler'".

- [ ] **Step 3: Write `src/main/apiRequestHandler.ts`**

```ts
import type { ApiRequestArgs, ApiResponse } from '../shared/types';

export async function executeApiRequest(
  args: ApiRequestArgs,
  fetchImpl: typeof fetch = fetch,
): Promise<ApiResponse> {
  const headers: Record<string, string> = {};
  if (args.token) headers.Authorization = `Bearer ${args.token}`;
  if (args.json !== undefined) headers['Content-Type'] = 'application/json';

  const response = await fetchImpl(args.url, {
    method: args.method,
    headers,
    body: args.json !== undefined ? JSON.stringify(args.json) : undefined,
  });

  const text = await response.text();
  let body: unknown = null;
  if (text) {
    try {
      body = JSON.parse(text);
    } catch {
      body = text;
    }
  }

  return { status: response.status, ok: response.ok, body };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/main/apiRequestHandler.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Write the failing test for `uploadFileHandler`**

```ts
// test/main/uploadFileHandler.test.ts
import { describe, it, expect, vi } from 'vitest';
import { mkdtempSync, writeFileSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { executeUploadFile } from '../../src/main/uploadFileHandler';

describe('executeUploadFile', () => {
  it('reads the file from disk and sends it as multipart form data', async () => {
    const dir = mkdtempSync(join(tmpdir(), 'visique-gui-upload-test-'));
    const filePath = join(dir, 'report.xlsx');
    writeFileSync(filePath, 'fake-xlsx-bytes');

    const fetchMock = vi.fn(async (_url: string, init?: RequestInit) => {
      expect(init?.method).toBe('POST');
      expect((init?.headers as Record<string, string>).Authorization).toBe('Bearer tok-1');
      expect(init?.body).toBeInstanceOf(FormData);
      return new Response(JSON.stringify({ analysis_status: 'reliable' }), { status: 200 });
    });

    const result = await executeUploadFile(
      { url: 'https://example.com/analyze', token: 'tok-1', filePath, fileName: 'report.xlsx' },
      fetchMock as unknown as typeof fetch,
    );

    expect(result).toEqual({ status: 200, ok: true, body: { analysis_status: 'reliable' } });
    rmSync(dir, { recursive: true, force: true });
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `npx vitest run test/main/uploadFileHandler.test.ts`
Expected: FAIL with "Cannot find module '../../src/main/uploadFileHandler'".

- [ ] **Step 7: Write `src/main/uploadFileHandler.ts`**

```ts
import { readFileSync } from 'node:fs';
import type { ApiResponse, UploadFileArgs } from '../shared/types';

export async function executeUploadFile(
  args: UploadFileArgs,
  fetchImpl: typeof fetch = fetch,
): Promise<ApiResponse> {
  const buffer = readFileSync(args.filePath);
  const form = new FormData();
  form.append('file', new Blob([buffer]), args.fileName);

  const headers: Record<string, string> = {};
  if (args.token) headers.Authorization = `Bearer ${args.token}`;

  const response = await fetchImpl(args.url, { method: 'POST', headers, body: form });

  const text = await response.text();
  let body: unknown = null;
  if (text) {
    try {
      body = JSON.parse(text);
    } catch {
      body = text;
    }
  }

  return { status: response.status, ok: response.ok, body };
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `npx vitest run test/main/uploadFileHandler.test.ts`
Expected: PASS (1 test).

- [ ] **Step 9: Commit**

```bash
git add src/main/apiRequestHandler.ts src/main/uploadFileHandler.ts test/main/apiRequestHandler.test.ts test/main/uploadFileHandler.test.ts
git commit -m "feat: execute API/upload requests in the main process"
```

---

### Task 5: IPC registration, file dialog, and main entry point

**Files:**
- Create: `src/main/ipc.ts`
- Modify: `src/main/index.ts` (replace Task 1's placeholder)
- Test: `test/main/ipc.test.ts`

**Interfaces:**
- Consumes: `TokenStore` (Task 3), `executeApiRequest`/`executeUploadFile` (Task 4), `PickedFile` (Task 2).
- Produces: `registerIpcHandlers(tokenStore: TokenStore): void`, registering IPC channels `token:get`, `token:set`, `token:clear`, `api:request`, `api:uploadFile`, `dialog:openFile` — consumed by the preload bridge (Task 6). Also `toPickedFile(filePath: string): PickedFile`, a pure helper used inside the `dialog:openFile` handler.

- [ ] **Step 1: Write the failing test**

```ts
// test/main/ipc.test.ts
import { describe, it, expect, vi } from 'vitest';
import { mkdtempSync, writeFileSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

vi.mock('electron', () => ({
  ipcMain: { handle: vi.fn() },
  dialog: { showOpenDialog: vi.fn() },
  BrowserWindow: { getFocusedWindow: vi.fn(() => null) },
  safeStorage: {
    isEncryptionAvailable: () => true,
    encryptString: (s: string) => Buffer.from(`enc:${s}`),
    decryptString: (b: Buffer) => b.toString('utf-8').replace(/^enc:/, ''),
  },
}));

import { ipcMain } from 'electron';
import { toPickedFile } from '../../src/main/ipc';

describe('toPickedFile', () => {
  it('builds a PickedFile descriptor with the real file size', () => {
    const dir = mkdtempSync(join(tmpdir(), 'visique-gui-ipc-test-'));
    const filePath = join(dir, 'q4.xlsx');
    writeFileSync(filePath, '12345');

    const picked = toPickedFile(filePath);

    expect(picked.path).toBe(filePath);
    expect(picked.name).toBe('q4.xlsx');
    expect(picked.size).toBe(5);
    rmSync(dir, { recursive: true, force: true });
  });
});

describe('registerIpcHandlers', () => {
  it('registers all six IPC channels', async () => {
    const { registerIpcHandlers } = await import('../../src/main/ipc');
    const { TokenStore } = await import('../../src/main/tokenStore');
    const dir = mkdtempSync(join(tmpdir(), 'visique-gui-ipc-test-'));
    registerIpcHandlers(new TokenStore(dir));

    const registeredChannels = (ipcMain.handle as ReturnType<typeof vi.fn>).mock.calls.map(
      (call) => call[0],
    );
    expect(registeredChannels).toEqual(
      expect.arrayContaining([
        'token:get',
        'token:set',
        'token:clear',
        'api:request',
        'api:uploadFile',
        'dialog:openFile',
      ]),
    );
    rmSync(dir, { recursive: true, force: true });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/main/ipc.test.ts`
Expected: FAIL with "Cannot find module '../../src/main/ipc'".

- [ ] **Step 3: Write `src/main/ipc.ts`**

```ts
import { BrowserWindow, dialog, ipcMain } from 'electron';
import { basename } from 'node:path';
import { statSync } from 'node:fs';
import type { TokenStore } from './tokenStore';
import { executeApiRequest } from './apiRequestHandler';
import { executeUploadFile } from './uploadFileHandler';
import type { ApiRequestArgs, PickedFile, UploadFileArgs } from '../shared/types';

export function toPickedFile(filePath: string): PickedFile {
  const stats = statSync(filePath);
  return { path: filePath, name: basename(filePath), size: stats.size };
}

export function registerIpcHandlers(tokenStore: TokenStore): void {
  ipcMain.handle('token:get', () => tokenStore.getToken());
  ipcMain.handle('token:set', (_event, token: string) => tokenStore.setToken(token));
  ipcMain.handle('token:clear', () => tokenStore.clearToken());

  ipcMain.handle('api:request', (_event, args: ApiRequestArgs) => executeApiRequest(args));
  ipcMain.handle('api:uploadFile', (_event, args: UploadFileArgs) => executeUploadFile(args));

  ipcMain.handle('dialog:openFile', async () => {
    const win = BrowserWindow.getFocusedWindow();
    const result = await dialog.showOpenDialog(win ?? undefined, {
      properties: ['openFile'],
      filters: [{ name: 'Excel Files', extensions: ['xlsx'] }],
    });
    if (result.canceled || result.filePaths.length === 0) return null;
    return toPickedFile(result.filePaths[0]);
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/main/ipc.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Replace `src/main/index.ts`**

```ts
import { app, BrowserWindow } from 'electron';
import { join } from 'node:path';
import { registerIpcHandlers } from './ipc';
import { TokenStore } from './tokenStore';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    show: false,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: false,
    },
  });

  mainWindow.on('ready-to-show', () => mainWindow.show());

  if (process.env['ELECTRON_RENDERER_URL']) {
    void mainWindow.loadURL(process.env['ELECTRON_RENDERER_URL']);
  } else {
    void mainWindow.loadFile(join(__dirname, '../renderer/index.html'));
  }
}

app.whenReady().then(() => {
  const tokenStore = new TokenStore(app.getPath('userData'));
  registerIpcHandlers(tokenStore);
  createWindow();

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
```

- [ ] **Step 6: Verify the build still passes**

Run: `npm run build`
Expected: exits 0.

- [ ] **Step 7: Commit**

```bash
git add src/main/ipc.ts src/main/index.ts test/main/ipc.test.ts
git commit -m "feat: register IPC handlers and wire main process entry point"
```

---

### Task 6: Preload bridge

**Files:**
- Modify: `src/preload/index.ts` (replace Task 1's placeholder)
- Create: `src/renderer/src/visique-global.d.ts`
- Test: `test/preload/index.test.ts`

**Interfaces:**
- Consumes: `ApiRequestArgs` from `src/shared/types.ts` (Task 2).
- Produces: a `window.visique` object with `openFileDialog`, `uploadFile`, `apiRequest`, `getToken`, `setToken`, `clearToken` — the only surface the renderer (Tasks 7+) touches to reach the main process.

- [ ] **Step 1: Write the failing test**

```ts
// test/preload/index.test.ts
import { describe, it, expect, vi } from 'vitest';

const exposeInMainWorld = vi.fn();
const invoke = vi.fn();

vi.mock('electron', () => ({
  contextBridge: { exposeInMainWorld },
  ipcRenderer: { invoke },
}));

describe('preload bridge', () => {
  it('exposes window.visique with the six expected methods', async () => {
    await import('../../src/preload/index');

    expect(exposeInMainWorld).toHaveBeenCalledWith('visique', expect.any(Object));
    const bridge = exposeInMainWorld.mock.calls[0][1];
    expect(Object.keys(bridge).sort()).toEqual(
      ['apiRequest', 'clearToken', 'getToken', 'openFileDialog', 'setToken', 'uploadFile'].sort(),
    );
  });

  it('forwards apiRequest to the api:request IPC channel', async () => {
    await import('../../src/preload/index');
    const bridge = exposeInMainWorld.mock.calls[0][1];

    const args = { method: 'GET' as const, url: 'https://example.com/history' };
    void bridge.apiRequest(args);

    expect(invoke).toHaveBeenCalledWith('api:request', args);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/preload/index.test.ts`
Expected: FAIL — `exposeInMainWorld` not called (placeholder preload has no logic).

- [ ] **Step 3: Write `src/preload/index.ts`**

```ts
import { contextBridge, ipcRenderer } from 'electron';
import type { ApiRequestArgs } from '../shared/types';

contextBridge.exposeInMainWorld('visique', {
  openFileDialog: () => ipcRenderer.invoke('dialog:openFile'),
  uploadFile: (url: string, token: string | null, filePath: string, fileName: string) =>
    ipcRenderer.invoke('api:uploadFile', { url, token, filePath, fileName }),
  apiRequest: (args: ApiRequestArgs) => ipcRenderer.invoke('api:request', args),
  getToken: () => ipcRenderer.invoke('token:get'),
  setToken: (token: string) => ipcRenderer.invoke('token:set', token),
  clearToken: () => ipcRenderer.invoke('token:clear'),
});
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/preload/index.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Write `src/renderer/src/visique-global.d.ts`** (renderer-side type declaration for `window.visique`)

```ts
import type { ApiRequestArgs, ApiResponse, PickedFile } from '../../shared/types';

export {};

declare global {
  interface Window {
    visique: {
      openFileDialog: () => Promise<PickedFile | null>;
      uploadFile: (
        url: string,
        token: string | null,
        filePath: string,
        fileName: string,
      ) => Promise<ApiResponse>;
      apiRequest: (args: ApiRequestArgs) => Promise<ApiResponse>;
      getToken: () => Promise<string | null>;
      setToken: (token: string) => Promise<void>;
      clearToken: () => Promise<void>;
    };
  }
}
```

- [ ] **Step 6: Commit**

```bash
git add src/preload/index.ts src/renderer/src/visique-global.d.ts test/preload/index.test.ts
git commit -m "feat: expose window.visique preload bridge"
```

---

### Task 7: Renderer API client

**Files:**
- Create: `src/renderer/src/api/visiqueClient.ts`
- Test: `test/renderer/visiqueClient.test.ts`

**Interfaces:**
- Consumes: `window.visique.apiRequest` / `window.visique.uploadFile` (Task 6), `StandardizedDataPackage`, `AnalysisHistoryEntry`, `PickedFile` (Task 2).
- Produces: `analyzeFile(token, file: PickedFile): Promise<StandardizedDataPackage>`, `saveAnalysis(token, pkg): Promise<{ id: number }>`, `getHistory(token): Promise<AnalysisHistoryEntry[]>`, `getAnalysis(token, id): Promise<StandardizedDataPackage>`, and `class ApiError extends Error { status: number }` — used by Tasks 12 and 13 (Upload/History screens).

- [ ] **Step 1: Write the failing test**

```ts
// test/renderer/visiqueClient.test.ts
/** @vitest-environment jsdom */
import { describe, it, expect, vi, beforeEach } from 'vitest';
import {
  analyzeFile,
  saveAnalysis,
  getHistory,
  getAnalysis,
  ApiError,
} from '../../src/renderer/src/api/visiqueClient';

const apiRequest = vi.fn();
const uploadFile = vi.fn();

beforeEach(() => {
  apiRequest.mockReset();
  uploadFile.mockReset();
  (window as unknown as { visique: unknown }).visique = { apiRequest, uploadFile };
});

describe('visiqueClient', () => {
  it('analyzeFile uploads the picked file and returns the package', async () => {
    uploadFile.mockResolvedValue({ status: 200, ok: true, body: { analysis_status: 'reliable' } });

    const result = await analyzeFile('tok-1', { path: '/tmp/q4.xlsx', name: 'q4.xlsx', size: 10 });

    expect(uploadFile).toHaveBeenCalledWith(
      expect.stringContaining('/api/analyze/xlsx'),
      'tok-1',
      '/tmp/q4.xlsx',
      'q4.xlsx',
    );
    expect(result).toEqual({ analysis_status: 'reliable' });
  });

  it('saveAnalysis posts to /api/save (never the raw backend endpoint)', async () => {
    apiRequest.mockResolvedValue({ status: 200, ok: true, body: { id: 7 } });

    const result = await saveAnalysis('tok-1', { analysis_status: 'reliable' } as never);

    expect(apiRequest).toHaveBeenCalledWith({
      method: 'POST',
      url: expect.stringContaining('/api/save'),
      token: 'tok-1',
      json: { analysis_status: 'reliable' },
    });
    expect(apiRequest.mock.calls[0][0].url).not.toContain('/api/v1/analysis/save');
    expect(result).toEqual({ id: 7 });
  });

  it('getHistory fetches the FastAPI history list', async () => {
    apiRequest.mockResolvedValue({ status: 200, ok: true, body: [{ id: 1 }] });

    const result = await getHistory('tok-1');

    expect(apiRequest).toHaveBeenCalledWith({
      method: 'GET',
      url: expect.stringContaining('/api/v1/analysis/history'),
      token: 'tok-1',
    });
    expect(result).toEqual([{ id: 1 }]);
  });

  it('getAnalysis fetches a single saved analysis by id', async () => {
    apiRequest.mockResolvedValue({ status: 200, ok: true, body: { analysis_id: 7 } });

    const result = await getAnalysis('tok-1', 7);

    expect(apiRequest).toHaveBeenCalledWith({
      method: 'GET',
      url: expect.stringContaining('/api/v1/analysis/history/7'),
      token: 'tok-1',
    });
    expect(result).toEqual({ analysis_id: 7 });
  });

  it('throws ApiError with the backend detail message on a non-ok response', async () => {
    apiRequest.mockResolvedValue({ status: 404, ok: false, body: { detail: 'Analysis not found' } });

    await expect(getAnalysis('tok-1', 999)).rejects.toThrow('Analysis not found');
    await expect(getAnalysis('tok-1', 999)).rejects.toBeInstanceOf(ApiError);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/visiqueClient.test.ts`
Expected: FAIL with "Cannot find module '../../src/renderer/src/api/visiqueClient'".

- [ ] **Step 3: Write `src/renderer/src/api/visiqueClient.ts`**

```ts
import type {
  AnalysisHistoryEntry,
  PickedFile,
  StandardizedDataPackage,
} from '../../../shared/types';

const WEB_BASE_URL = import.meta.env.VITE_VISIQUE_WEB_BASE_URL ?? 'https://visique.xyz';
const API_BASE_URL =
  import.meta.env.VITE_VISIQUE_API_BASE_URL ?? 'https://visique-backend.onrender.com';

export class ApiError extends Error {
  status: number;

  constructor(status: number, body: unknown) {
    const detail =
      body && typeof body === 'object' && 'detail' in (body as Record<string, unknown>)
        ? String((body as Record<string, unknown>).detail)
        : `Request failed with status ${status}`;
    super(detail);
    this.status = status;
  }
}

export async function analyzeFile(
  token: string,
  file: PickedFile,
): Promise<StandardizedDataPackage> {
  const res = await window.visique.uploadFile(
    `${WEB_BASE_URL}/api/analyze/xlsx`,
    token,
    file.path,
    file.name,
  );
  if (!res.ok) throw new ApiError(res.status, res.body);
  return res.body as StandardizedDataPackage;
}

export async function saveAnalysis(
  token: string,
  pkg: StandardizedDataPackage,
): Promise<{ id: number }> {
  // Deliberately /api/save (Vercel proxy), never the raw FastAPI
  // /api/v1/analysis/save — that endpoint requires an HMAC signature
  // signed with a server-only secret this client must never hold.
  const res = await window.visique.apiRequest({
    method: 'POST',
    url: `${WEB_BASE_URL}/api/save`,
    token,
    json: pkg,
  });
  if (!res.ok) throw new ApiError(res.status, res.body);
  return res.body as { id: number };
}

export async function getHistory(token: string): Promise<AnalysisHistoryEntry[]> {
  const res = await window.visique.apiRequest({
    method: 'GET',
    url: `${API_BASE_URL}/api/v1/analysis/history`,
    token,
  });
  if (!res.ok) throw new ApiError(res.status, res.body);
  return res.body as AnalysisHistoryEntry[];
}

export async function getAnalysis(
  token: string,
  id: number,
): Promise<StandardizedDataPackage> {
  const res = await window.visique.apiRequest({
    method: 'GET',
    url: `${API_BASE_URL}/api/v1/analysis/history/${id}`,
    token,
  });
  if (!res.ok) throw new ApiError(res.status, res.body);
  return res.body as StandardizedDataPackage;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/visiqueClient.test.ts`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add src/renderer/src/api/visiqueClient.ts test/renderer/visiqueClient.test.ts
git commit -m "feat: add renderer API client"
```

---

### Task 8: Mocked auth context

**Files:**
- Create: `src/renderer/src/auth/AuthContext.tsx`
- Test: `test/renderer/AuthContext.test.tsx`

**Interfaces:**
- Consumes: `window.visique.setToken` (Task 6).
- Produces: `AuthProvider` (React component), `useAuth(): { token: string; userEmail: string }` — used by Tasks 12, 13, 14.

- [ ] **Step 1: Write the failing test**

```tsx
// test/renderer/AuthContext.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';
import { AuthProvider, useAuth } from '../../src/renderer/src/auth/AuthContext';

const setToken = vi.fn().mockResolvedValue(undefined);

beforeEach(() => {
  setToken.mockClear();
  (window as unknown as { visique: unknown }).visique = { setToken };
});

function Probe() {
  const { token, userEmail } = useAuth();
  return <div>{token}:{userEmail}</div>;
}

describe('AuthContext', () => {
  it('provides a fixed mock token and email, and persists the token via IPC', () => {
    render(
      <AuthProvider>
        <Probe />
      </AuthProvider>,
    );

    expect(screen.getByText('mock-dev-token:dev@visique.ai')).toBeInTheDocument();
    expect(setToken).toHaveBeenCalledWith('mock-dev-token');
  });

  it('throws if useAuth is called outside an AuthProvider', () => {
    const consoleError = vi.spyOn(console, 'error').mockImplementation(() => {});
    expect(() => render(<Probe />)).toThrow('useAuth must be used within an AuthProvider');
    consoleError.mockRestore();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/AuthContext.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/auth/AuthContext'".

- [ ] **Step 3: Write `src/renderer/src/auth/AuthContext.tsx`**

```tsx
import { createContext, useContext, useEffect, useState, type ReactNode } from 'react';

interface AuthValue {
  token: string;
  userEmail: string;
}

// v1 auth is mocked. Real login (/login -> OTP -> /verify-otp) replaces
// this provider's internals later without changing useAuth()'s shape.
const MOCK_TOKEN = 'mock-dev-token';
const MOCK_EMAIL = 'dev@visique.ai';

const AuthContext = createContext<AuthValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [value] = useState<AuthValue>({ token: MOCK_TOKEN, userEmail: MOCK_EMAIL });

  useEffect(() => {
    void window.visique.setToken(value.token);
  }, [value.token]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within an AuthProvider');
  return ctx;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/AuthContext.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/renderer/src/auth/AuthContext.tsx test/renderer/AuthContext.test.tsx
git commit -m "feat: add mocked auth context"
```

---

### Task 9: File validation and error banner

**Files:**
- Create: `src/renderer/src/utils/validateFile.ts`
- Create: `src/renderer/src/components/ErrorBanner.tsx`
- Test: `test/renderer/validateFile.test.ts`
- Test: `test/renderer/ErrorBanner.test.tsx`

**Interfaces:**
- Consumes: `PickedFile` (Task 2).
- Produces: `validateFile(file: PickedFile): string | null`, `<ErrorBanner message={string} onRetry?={() => void} />` — used by Tasks 12, 13.

- [ ] **Step 1: Write the failing test for `validateFile`**

```ts
// test/renderer/validateFile.test.ts
import { describe, it, expect } from 'vitest';
import { validateFile } from '../../src/renderer/src/utils/validateFile';

describe('validateFile', () => {
  it('rejects non-.xlsx files', () => {
    const error = validateFile({ path: '/tmp/report.csv', name: 'report.csv', size: 100 });
    expect(error).toBe('Only .xlsx files are supported.');
  });

  it('rejects files over the 4MB Vercel wrapper ceiling', () => {
    const error = validateFile({
      path: '/tmp/big.xlsx',
      name: 'big.xlsx',
      size: 5 * 1024 * 1024,
    });
    expect(error).toContain('too large');
  });

  it('accepts a valid .xlsx file under the size limit', () => {
    const error = validateFile({ path: '/tmp/q4.xlsx', name: 'q4.xlsx', size: 1024 });
    expect(error).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/validateFile.test.ts`
Expected: FAIL with "Cannot find module '../../src/renderer/src/utils/validateFile'".

- [ ] **Step 3: Write `src/renderer/src/utils/validateFile.ts`**

```ts
import type { PickedFile } from '../../../shared/types';

const MAX_SIZE_BYTES = 4 * 1024 * 1024;

export function validateFile(file: PickedFile): string | null {
  if (!file.name.toLowerCase().endsWith('.xlsx')) {
    return 'Only .xlsx files are supported.';
  }
  if (file.size > MAX_SIZE_BYTES) {
    return `File is too large (${(file.size / 1024 / 1024).toFixed(1)}MB). Maximum size is 4MB.`;
  }
  return null;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/validateFile.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Write the failing test for `ErrorBanner`**

```tsx
// test/renderer/ErrorBanner.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ErrorBanner } from '../../src/renderer/src/components/ErrorBanner';

describe('ErrorBanner', () => {
  it('renders the message and no retry button when onRetry is omitted', () => {
    render(<ErrorBanner message="Failed to save analysis." />);
    expect(screen.getByRole('alert')).toHaveTextContent('Failed to save analysis.');
    expect(screen.queryByRole('button')).not.toBeInTheDocument();
  });

  it('calls onRetry when the retry button is clicked', async () => {
    const onRetry = vi.fn();
    render(<ErrorBanner message="Network error." onRetry={onRetry} />);
    await userEvent.click(screen.getByRole('button', { name: 'Retry' }));
    expect(onRetry).toHaveBeenCalledOnce();
  });
});
```

- [ ] **Step 6: Add `@testing-library/user-event` to `package.json` devDependencies**

```json
"@testing-library/user-event": "^14.5.2"
```

Run: `npm install`

- [ ] **Step 7: Run test to verify it fails**

Run: `npx vitest run test/renderer/ErrorBanner.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/components/ErrorBanner'".

- [ ] **Step 8: Write `src/renderer/src/components/ErrorBanner.tsx`**

```tsx
interface ErrorBannerProps {
  message: string;
  onRetry?: () => void;
}

export function ErrorBanner({ message, onRetry }: ErrorBannerProps) {
  return (
    <div role="alert" className="error-banner">
      <span>{message}</span>
      {onRetry ? <button onClick={onRetry}>Retry</button> : null}
    </div>
  );
}
```

- [ ] **Step 9: Run test to verify it passes**

Run: `npx vitest run test/renderer/ErrorBanner.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 10: Commit**

```bash
git add package.json src/renderer/src/utils/validateFile.ts src/renderer/src/components/ErrorBanner.tsx test/renderer/validateFile.test.ts test/renderer/ErrorBanner.test.tsx
git commit -m "feat: add file validation and error banner"
```

---

### Task 10: Dashboard chart components

**Files:**
- Create: `src/renderer/src/components/HealthScoreChart.tsx`
- Create: `src/renderer/src/components/KpiGrid.tsx`
- Create: `src/renderer/src/components/KpiBarChart.tsx`
- Test: `test/renderer/HealthScoreChart.test.tsx`
- Test: `test/renderer/KpiGrid.test.tsx`
- Test: `test/renderer/KpiBarChart.test.tsx`

**Interfaces:**
- Consumes: `HealthScoreBreakdown`, `KPIMetrics` (Task 2), fixture data (Task 2).
- Produces: `<HealthScoreChart score={HealthScoreBreakdown} />`, `<KpiGrid kpis={KPIMetrics} />`, `<KpiBarChart kpis={KPIMetrics} />` — used by Task 11 (DashboardScreen).

Note: the approved design spec called these "trend charts," but a single `StandardizedDataPackage` is a one-period snapshot — `financial_model/models.py`'s `FinancialReport` has one `period_end`, not a time series. There's no historical data to trend within one analysis. v1 instead visualizes the real fields that exist for a single analysis: the four `HealthScoreBreakdown` components, and the KPI margin values.

- [ ] **Step 1: Write the failing test for `HealthScoreChart`**

```tsx
// test/renderer/HealthScoreChart.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { HealthScoreChart } from '../../src/renderer/src/components/HealthScoreChart';
import fixture from '../fixtures/sample-package.json';

describe('HealthScoreChart', () => {
  it('renders the total score', () => {
    render(<HealthScoreChart score={fixture.health_score} />);
    expect(screen.getByText(/Health Score: 73/)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/HealthScoreChart.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/components/HealthScoreChart'".

- [ ] **Step 3: Write `src/renderer/src/components/HealthScoreChart.tsx`**

```tsx
import { Bar, BarChart, ResponsiveContainer, Tooltip, XAxis, YAxis } from 'recharts';
import type { HealthScoreBreakdown } from '../../../shared/types';

export function HealthScoreChart({ score }: { score: HealthScoreBreakdown }) {
  const data = [
    { name: 'Stability', value: score.stability },
    { name: 'Profitability', value: score.profitability },
    { name: 'Growth', value: score.growth },
    { name: 'Efficiency', value: score.efficiency },
  ];

  return (
    <div className="health-score-chart">
      <h3>Health Score: {score.total_score.toFixed(0)}</h3>
      <ResponsiveContainer width="100%" height={200}>
        <BarChart data={data}>
          <XAxis dataKey="name" />
          <YAxis domain={[0, 100]} />
          <Tooltip />
          <Bar dataKey="value" fill="#4f46e5" />
        </BarChart>
      </ResponsiveContainer>
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/HealthScoreChart.test.tsx`
Expected: PASS (1 test).

- [ ] **Step 5: Write the failing test for `KpiGrid`**

```tsx
// test/renderer/KpiGrid.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { KpiGrid } from '../../src/renderer/src/components/KpiGrid';
import fixture from '../fixtures/sample-package.json';

describe('KpiGrid', () => {
  it('renders a labeled value for each known KPI', () => {
    render(<KpiGrid kpis={fixture.kpis} />);
    expect(screen.getByText('Gross Margin')).toBeInTheDocument();
    expect(screen.getByText('0.42')).toBeInTheDocument();
  });

  it('renders a dash for a null KPI value', () => {
    render(<KpiGrid kpis={{ ...fixture.kpis, roe: null }} />);
    expect(screen.getByText('—')).toBeInTheDocument();
  });
});
```

- [ ] **Step 6: Run test to verify it fails**

Run: `npx vitest run test/renderer/KpiGrid.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/components/KpiGrid'".

- [ ] **Step 7: Write `src/renderer/src/components/KpiGrid.tsx`**

```tsx
import type { KPIMetrics } from '../../../shared/types';

const KPI_LABELS: Record<keyof KPIMetrics, string> = {
  current_ratio: 'Current Ratio',
  quick_ratio: 'Quick Ratio',
  gross_margin: 'Gross Margin',
  operating_margin: 'Operating Margin',
  net_margin: 'Net Margin',
  ebitda_margin: 'EBITDA Margin',
  debt_to_equity: 'Debt to Equity',
  roa: 'Return on Assets',
  roe: 'Return on Equity',
};

export function KpiGrid({ kpis }: { kpis: KPIMetrics }) {
  const keys = Object.keys(KPI_LABELS) as Array<keyof KPIMetrics>;

  return (
    <div className="kpi-grid">
      {keys.map((key) => {
        const value = kpis[key];
        return (
          <div className="kpi-card" key={key}>
            <div className="kpi-label">{KPI_LABELS[key]}</div>
            <div className="kpi-value">{value === null ? '—' : value.toFixed(2)}</div>
          </div>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `npx vitest run test/renderer/KpiGrid.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 9: Write the failing test for `KpiBarChart`**

```tsx
// test/renderer/KpiBarChart.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect } from 'vitest';
import { render } from '@testing-library/react';
import { KpiBarChart } from '../../src/renderer/src/components/KpiBarChart';
import fixture from '../fixtures/sample-package.json';

describe('KpiBarChart', () => {
  it('renders without throwing given a full KPI set', () => {
    expect(() => render(<KpiBarChart kpis={fixture.kpis} />)).not.toThrow();
  });

  it('renders without throwing when margin fields are null', () => {
    const kpis = { ...fixture.kpis, gross_margin: null, net_margin: null };
    expect(() => render(<KpiBarChart kpis={kpis} />)).not.toThrow();
  });
});
```

- [ ] **Step 10: Run test to verify it fails**

Run: `npx vitest run test/renderer/KpiBarChart.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/components/KpiBarChart'".

- [ ] **Step 11: Write `src/renderer/src/components/KpiBarChart.tsx`**

```tsx
import { Bar, BarChart, ResponsiveContainer, Tooltip, XAxis, YAxis } from 'recharts';
import type { KPIMetrics } from '../../../shared/types';

export function KpiBarChart({ kpis }: { kpis: KPIMetrics }) {
  const data = [
    { name: 'Gross', value: (kpis.gross_margin ?? 0) * 100 },
    { name: 'Operating', value: (kpis.operating_margin ?? 0) * 100 },
    { name: 'Net', value: (kpis.net_margin ?? 0) * 100 },
    { name: 'EBITDA', value: (kpis.ebitda_margin ?? 0) * 100 },
  ];

  return (
    <ResponsiveContainer width="100%" height={220}>
      <BarChart data={data}>
        <XAxis dataKey="name" />
        <YAxis unit="%" />
        <Tooltip formatter={(value: number) => `${value.toFixed(1)}%`} />
        <Bar dataKey="value" fill="#0ea5e9" />
      </BarChart>
    </ResponsiveContainer>
  );
}
```

- [ ] **Step 12: Run test to verify it passes**

Run: `npx vitest run test/renderer/KpiBarChart.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 13: Commit**

```bash
git add src/renderer/src/components/HealthScoreChart.tsx src/renderer/src/components/KpiGrid.tsx src/renderer/src/components/KpiBarChart.tsx test/renderer/HealthScoreChart.test.tsx test/renderer/KpiGrid.test.tsx test/renderer/KpiBarChart.test.tsx
git commit -m "feat: add dashboard chart components"
```

---

### Task 11: Dashboard screen

**Files:**
- Create: `src/renderer/src/screens/DashboardScreen.tsx`
- Test: `test/renderer/DashboardScreen.test.tsx`

**Interfaces:**
- Consumes: `HealthScoreChart`, `KpiGrid`, `KpiBarChart` (Task 10), `StandardizedDataPackage` (Task 2).
- Produces: `<DashboardScreen pkg={StandardizedDataPackage} />` — used by Task 14 (AppShell).

- [ ] **Step 1: Write the failing test**

```tsx
// test/renderer/DashboardScreen.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { DashboardScreen } from '../../src/renderer/src/screens/DashboardScreen';
import fixture from '../fixtures/sample-package.json';
import type { StandardizedDataPackage } from '../../src/shared/types';

describe('DashboardScreen', () => {
  it('renders the company name and no status banner when reliable', () => {
    render(<DashboardScreen pkg={fixture as StandardizedDataPackage} />);
    expect(screen.getByText('Acme Robotics, Inc.')).toBeInTheDocument();
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  });

  it('shows a warning banner when analysis_status is degraded', () => {
    const degraded: StandardizedDataPackage = { ...(fixture as StandardizedDataPackage), analysis_status: 'degraded' };
    render(<DashboardScreen pkg={degraded} />);
    expect(screen.getByRole('alert')).toHaveTextContent('degraded');
  });

  it('renders each insight', () => {
    render(<DashboardScreen pkg={fixture as StandardizedDataPackage} />);
    expect(screen.getByText(/Revenue grew 18%/)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/DashboardScreen.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/screens/DashboardScreen'".

- [ ] **Step 3: Write `src/renderer/src/screens/DashboardScreen.tsx`**

```tsx
import { HealthScoreChart } from '../components/HealthScoreChart';
import { KpiGrid } from '../components/KpiGrid';
import { KpiBarChart } from '../components/KpiBarChart';
import type { StandardizedDataPackage } from '../../../shared/types';

export function DashboardScreen({ pkg }: { pkg: StandardizedDataPackage }) {
  const isDegraded = pkg.analysis_status === 'degraded' || pkg.analysis_status === 'quarantined';

  return (
    <div className="dashboard-screen">
      <h2>{pkg.raw_data.company_name}</h2>
      <p className="period">
        {pkg.raw_data.period_end} ({pkg.raw_data.period_type})
      </p>
      {isDegraded ? (
        <div role="alert" className="status-banner">
          Data quality is {pkg.analysis_status}. Some features may be suppressed or less reliable.
        </div>
      ) : null}
      <HealthScoreChart score={pkg.health_score} />
      <KpiGrid kpis={pkg.kpis} />
      <KpiBarChart kpis={pkg.kpis} />
      {pkg.insights.length > 0 ? (
        <ul className="insights">
          {pkg.insights.map((insight) => (
            <li key={insight}>{insight}</li>
          ))}
        </ul>
      ) : null}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/DashboardScreen.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/renderer/src/screens/DashboardScreen.tsx test/renderer/DashboardScreen.test.tsx
git commit -m "feat: add dashboard screen"
```

---

### Task 12: Upload screen

**Files:**
- Create: `src/renderer/src/screens/UploadScreen.tsx`
- Test: `test/renderer/UploadScreen.test.tsx`

**Interfaces:**
- Consumes: `useAuth` (Task 8), `analyzeFile`/`saveAnalysis`/`ApiError` (Task 7), `validateFile` (Task 9), `ErrorBanner` (Task 9), `PickedFile`/`StandardizedDataPackage` (Task 2).
- Produces: `<UploadScreen onAnalyzed={(pkg: StandardizedDataPackage) => void} />` — used by Task 14 (AppShell).

- [ ] **Step 1: Write the failing test**

```tsx
// test/renderer/UploadScreen.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UploadScreen } from '../../src/renderer/src/screens/UploadScreen';
import { AuthProvider } from '../../src/renderer/src/auth/AuthContext';
import * as client from '../../src/renderer/src/api/visiqueClient';
import fixture from '../fixtures/sample-package.json';

vi.mock('../../src/renderer/src/api/visiqueClient', async () => {
  const actual = await vi.importActual<typeof import('../../src/renderer/src/api/visiqueClient')>(
    '../../src/renderer/src/api/visiqueClient',
  );
  return { ...actual, analyzeFile: vi.fn(), saveAnalysis: vi.fn() };
});

beforeEach(() => {
  vi.mocked(client.analyzeFile).mockReset();
  vi.mocked(client.saveAnalysis).mockReset();
  (window as unknown as { visique: unknown }).visique = {
    setToken: vi.fn().mockResolvedValue(undefined),
    openFileDialog: vi.fn().mockResolvedValue({ path: '/tmp/q4.xlsx', name: 'q4.xlsx', size: 1024 }),
  };
});

function renderUpload(onAnalyzed = vi.fn()) {
  render(
    <AuthProvider>
      <UploadScreen onAnalyzed={onAnalyzed} />
    </AuthProvider>,
  );
  return onAnalyzed;
}

describe('UploadScreen', () => {
  it('analyzes and saves the picked file, then calls onAnalyzed with the merged package', async () => {
    vi.mocked(client.analyzeFile).mockResolvedValue(fixture as never);
    vi.mocked(client.saveAnalysis).mockResolvedValue({ id: 55 });
    const onAnalyzed = renderUpload();

    await userEvent.click(screen.getByRole('button', { name: 'Choose File' }));

    await waitFor(() => {
      expect(onAnalyzed).toHaveBeenCalledWith(expect.objectContaining({ analysis_id: 55 }));
    });
  });

  it('shows a retry banner and keeps the analyzed package in memory when save fails', async () => {
    vi.mocked(client.analyzeFile).mockResolvedValue(fixture as never);
    vi.mocked(client.saveAnalysis)
      .mockRejectedValueOnce(new client.ApiError(500, { detail: 'Save failed' }))
      .mockResolvedValueOnce({ id: 56 });
    const onAnalyzed = renderUpload();

    await userEvent.click(screen.getByRole('button', { name: 'Choose File' }));

    await waitFor(() => expect(screen.getByRole('alert')).toHaveTextContent('Save failed'));
    expect(onAnalyzed).not.toHaveBeenCalled();

    await userEvent.click(screen.getByRole('button', { name: 'Retry' }));

    await waitFor(() => {
      expect(onAnalyzed).toHaveBeenCalledWith(expect.objectContaining({ analysis_id: 56 }));
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/UploadScreen.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/screens/UploadScreen'".

- [ ] **Step 3: Write `src/renderer/src/screens/UploadScreen.tsx`**

```tsx
import { useCallback, useState } from 'react';
import { useAuth } from '../auth/AuthContext';
import { ApiError, analyzeFile, saveAnalysis } from '../api/visiqueClient';
import { validateFile } from '../utils/validateFile';
import { ErrorBanner } from '../components/ErrorBanner';
import type { PickedFile, StandardizedDataPackage } from '../../../shared/types';

interface UploadScreenProps {
  onAnalyzed: (pkg: StandardizedDataPackage) => void;
}

type Stage = 'idle' | 'analyzing' | 'saving' | 'save-failed';

export function UploadScreen({ onAnalyzed }: UploadScreenProps) {
  const { token } = useAuth();
  const [stage, setStage] = useState<Stage>('idle');
  const [error, setError] = useState<string | null>(null);
  const [pendingPackage, setPendingPackage] = useState<StandardizedDataPackage | null>(null);

  const persist = useCallback(
    async (pkg: StandardizedDataPackage) => {
      setStage('saving');
      try {
        const { id } = await saveAnalysis(token, pkg);
        setStage('idle');
        setPendingPackage(null);
        onAnalyzed({ ...pkg, analysis_id: id });
      } catch (err) {
        setPendingPackage(pkg);
        setStage('save-failed');
        setError(err instanceof ApiError ? err.message : 'Failed to save analysis.');
      }
    },
    [token, onAnalyzed],
  );

  const runAnalysis = useCallback(
    async (file: PickedFile) => {
      const validationError = validateFile(file);
      if (validationError) {
        setError(validationError);
        return;
      }
      setError(null);
      setStage('analyzing');
      try {
        const pkg = await analyzeFile(token, file);
        await persist(pkg);
      } catch (err) {
        setStage('idle');
        setError(err instanceof ApiError ? err.message : 'Failed to analyze file.');
      }
    },
    [token, persist],
  );

  const handleRetrySave = useCallback(() => {
    if (pendingPackage) void persist(pendingPackage);
  }, [pendingPackage, persist]);

  const handlePick = useCallback(async () => {
    const file = await window.visique.openFileDialog();
    if (file) void runAnalysis(file);
  }, [runAnalysis]);

  const handleDrop = useCallback(
    (event: React.DragEvent<HTMLDivElement>) => {
      event.preventDefault();
      const dropped = event.dataTransfer.files[0] as (File & { path: string }) | undefined;
      if (!dropped) return;
      void runAnalysis({ path: dropped.path, name: dropped.name, size: dropped.size });
    },
    [runAnalysis],
  );

  const busy = stage === 'analyzing' || stage === 'saving';

  return (
    <div className="upload-screen">
      <div className="drop-zone" onDragOver={(e) => e.preventDefault()} onDrop={handleDrop}>
        <p>Drag an .xlsx file here, or</p>
        <button onClick={() => void handlePick()} disabled={busy}>
          Choose File
        </button>
      </div>
      {stage === 'analyzing' ? <p>Analyzing…</p> : null}
      {stage === 'saving' ? <p>Saving…</p> : null}
      {error ? (
        <ErrorBanner message={error} onRetry={stage === 'save-failed' ? handleRetrySave : undefined} />
      ) : null}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/UploadScreen.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/renderer/src/screens/UploadScreen.tsx test/renderer/UploadScreen.test.tsx
git commit -m "feat: add upload screen with validation and retry-save"
```

---

### Task 13: History screen

**Files:**
- Create: `src/renderer/src/screens/HistoryScreen.tsx`
- Test: `test/renderer/HistoryScreen.test.tsx`

**Interfaces:**
- Consumes: `useAuth` (Task 8), `getHistory`/`getAnalysis`/`ApiError` (Task 7), `ErrorBanner` (Task 9), `AnalysisHistoryEntry`/`StandardizedDataPackage` (Task 2).
- Produces: `<HistoryScreen onSelect={(pkg: StandardizedDataPackage) => void} />` — used by Task 14 (AppShell).

- [ ] **Step 1: Write the failing test**

```tsx
// test/renderer/HistoryScreen.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { HistoryScreen } from '../../src/renderer/src/screens/HistoryScreen';
import { AuthProvider } from '../../src/renderer/src/auth/AuthContext';
import * as client from '../../src/renderer/src/api/visiqueClient';
import fixture from '../fixtures/sample-package.json';

vi.mock('../../src/renderer/src/api/visiqueClient', async () => {
  const actual = await vi.importActual<typeof import('../../src/renderer/src/api/visiqueClient')>(
    '../../src/renderer/src/api/visiqueClient',
  );
  return { ...actual, getHistory: vi.fn(), getAnalysis: vi.fn() };
});

beforeEach(() => {
  vi.mocked(client.getHistory).mockReset();
  vi.mocked(client.getAnalysis).mockReset();
  (window as unknown as { visique: unknown }).visique = {
    setToken: vi.fn().mockResolvedValue(undefined),
  };
});

describe('HistoryScreen', () => {
  it('lists past analyses and opens one on click', async () => {
    vi.mocked(client.getHistory).mockResolvedValue([
      { id: 1, company_name: 'Acme Robotics, Inc.', filename: 'q4.xlsx', timestamp: '2026-07-20', runner_name: 'Dev User' },
    ]);
    vi.mocked(client.getAnalysis).mockResolvedValue(fixture as never);
    const onSelect = vi.fn();

    render(
      <AuthProvider>
        <HistoryScreen onSelect={onSelect} />
      </AuthProvider>,
    );

    await waitFor(() => expect(screen.getByText('Acme Robotics, Inc.')).toBeInTheDocument());

    await userEvent.click(screen.getByText('Acme Robotics, Inc.'));

    await waitFor(() => expect(onSelect).toHaveBeenCalledWith(fixture));
  });

  it('shows a retry banner when the history request fails', async () => {
    vi.mocked(client.getHistory).mockRejectedValue(new client.ApiError(500, { detail: 'Server error' }));

    render(
      <AuthProvider>
        <HistoryScreen onSelect={vi.fn()} />
      </AuthProvider>,
    );

    await waitFor(() => expect(screen.getByRole('alert')).toHaveTextContent('Server error'));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/HistoryScreen.test.tsx`
Expected: FAIL with "Cannot find module '../../src/renderer/src/screens/HistoryScreen'".

- [ ] **Step 3: Write `src/renderer/src/screens/HistoryScreen.tsx`**

```tsx
import { useCallback, useEffect, useState } from 'react';
import { useAuth } from '../auth/AuthContext';
import { ApiError, getAnalysis, getHistory } from '../api/visiqueClient';
import { ErrorBanner } from '../components/ErrorBanner';
import type { AnalysisHistoryEntry, StandardizedDataPackage } from '../../../shared/types';

interface HistoryScreenProps {
  onSelect: (pkg: StandardizedDataPackage) => void;
}

export function HistoryScreen({ onSelect }: HistoryScreenProps) {
  const { token } = useAuth();
  const [entries, setEntries] = useState<AnalysisHistoryEntry[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      setEntries(await getHistory(token));
    } catch (err) {
      setError(err instanceof ApiError ? err.message : 'Failed to load history.');
    } finally {
      setLoading(false);
    }
  }, [token]);

  useEffect(() => {
    void load();
  }, [load]);

  const handleSelect = useCallback(
    async (id: number) => {
      try {
        onSelect(await getAnalysis(token, id));
      } catch (err) {
        setError(err instanceof ApiError ? err.message : 'Failed to load analysis.');
      }
    },
    [token, onSelect],
  );

  if (loading) return <p>Loading history…</p>;

  return (
    <div className="history-screen">
      {error ? <ErrorBanner message={error} onRetry={load} /> : null}
      <table>
        <thead>
          <tr>
            <th>Company</th>
            <th>File</th>
            <th>Date</th>
          </tr>
        </thead>
        <tbody>
          {entries.map((entry) => (
            <tr key={entry.id} onClick={() => void handleSelect(entry.id)} style={{ cursor: 'pointer' }}>
              <td>{entry.company_name}</td>
              <td>{entry.filename}</td>
              <td>{entry.timestamp}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/HistoryScreen.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/renderer/src/screens/HistoryScreen.tsx test/renderer/HistoryScreen.test.tsx
git commit -m "feat: add history screen"
```

---

### Task 14: App shell and renderer entry point

**Files:**
- Modify: `src/renderer/src/App.tsx` (replace Task 1's placeholder)
- Test: `test/renderer/App.test.tsx`

**Interfaces:**
- Consumes: `AuthProvider` (Task 8), `UploadScreen` (Task 12), `DashboardScreen` (Task 11), `HistoryScreen` (Task 13).
- Produces: `<App />` — the renderer's root component, unchanged export name so `main.tsx` (already written in Task 1) needs no changes.

- [ ] **Step 1: Write the failing test**

```tsx
// test/renderer/App.test.tsx
/** @vitest-environment jsdom */
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { App } from '../../src/renderer/src/App';
import * as client from '../../src/renderer/src/api/visiqueClient';
import fixture from '../fixtures/sample-package.json';

vi.mock('../../src/renderer/src/api/visiqueClient', async () => {
  const actual = await vi.importActual<typeof import('../../src/renderer/src/api/visiqueClient')>(
    '../../src/renderer/src/api/visiqueClient',
  );
  return { ...actual, getHistory: vi.fn(), getAnalysis: vi.fn(), analyzeFile: vi.fn(), saveAnalysis: vi.fn() };
});

beforeEach(() => {
  vi.mocked(client.getHistory).mockResolvedValue([]);
  (window as unknown as { visique: unknown }).visique = {
    setToken: vi.fn().mockResolvedValue(undefined),
    openFileDialog: vi.fn(),
  };
});

describe('App', () => {
  it('starts on the Upload screen and can navigate to History', async () => {
    render(<App />);

    expect(screen.getByRole('button', { name: 'Choose File' })).toBeInTheDocument();

    await userEvent.click(screen.getByRole('button', { name: 'History' }));

    await waitFor(() => expect(client.getHistory).toHaveBeenCalled());
  });

  it('shows the dashboard after selecting a history entry', async () => {
    vi.mocked(client.getHistory).mockResolvedValue([
      { id: 1, company_name: 'Acme Robotics, Inc.', filename: 'q4.xlsx', timestamp: '2026-07-20', runner_name: 'Dev User' },
    ]);
    vi.mocked(client.getAnalysis).mockResolvedValue(fixture as never);

    render(<App />);
    await userEvent.click(screen.getByRole('button', { name: 'History' }));
    await waitFor(() => expect(screen.getByText('Acme Robotics, Inc.')).toBeInTheDocument());
    await userEvent.click(screen.getByText('Acme Robotics, Inc.'));

    await waitFor(() => expect(screen.getByText(/Health Score/)).toBeInTheDocument());
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/renderer/App.test.tsx`
Expected: FAIL — `screen.getByRole('button', { name: 'Choose File' })` not found (placeholder `App.tsx` just renders `<div>Visique-GUI</div>`).

- [ ] **Step 3: Replace `src/renderer/src/App.tsx`**

```tsx
import { useCallback, useState } from 'react';
import { AuthProvider } from './auth/AuthContext';
import { UploadScreen } from './screens/UploadScreen';
import { DashboardScreen } from './screens/DashboardScreen';
import { HistoryScreen } from './screens/HistoryScreen';
import type { StandardizedDataPackage } from '../../shared/types';

type View = 'upload' | 'dashboard' | 'history';

function AppShell() {
  const [view, setView] = useState<View>('upload');
  const [activePackage, setActivePackage] = useState<StandardizedDataPackage | null>(null);

  const handleAnalyzed = useCallback((pkg: StandardizedDataPackage) => {
    setActivePackage(pkg);
    setView('dashboard');
  }, []);

  const handleHistorySelect = useCallback((pkg: StandardizedDataPackage) => {
    setActivePackage(pkg);
    setView('dashboard');
  }, []);

  return (
    <div className="app-shell">
      <nav>
        <button onClick={() => setView('upload')}>Upload</button>
        <button onClick={() => setView('history')}>History</button>
      </nav>
      <main>
        {view === 'upload' ? <UploadScreen onAnalyzed={handleAnalyzed} /> : null}
        {view === 'dashboard' && activePackage ? <DashboardScreen pkg={activePackage} /> : null}
        {view === 'history' ? <HistoryScreen onSelect={handleHistorySelect} /> : null}
      </main>
    </div>
  );
}

export function App() {
  return (
    <AuthProvider>
      <AppShell />
    </AuthProvider>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/renderer/App.test.tsx`
Expected: PASS (2 tests).

- [ ] **Step 5: Run the full test suite and typecheck**

```bash
npm run test
npm run typecheck
```

Expected: all tests pass, no type errors.

- [ ] **Step 6: Commit**

```bash
git add src/renderer/src/App.tsx test/renderer/App.test.tsx
git commit -m "feat: wire app shell navigation between upload, dashboard, history"
```

---

### Task 15: Manual end-to-end smoke test

**Files:** none (verification only).

**Interfaces:** none — this task validates the whole v1 slice against the real, live Vercel/FastAPI services.

- [ ] **Step 1: Launch the app in dev mode**

```bash
npm run dev
```

Expected: an Electron window opens showing the Upload screen with a "Choose File" button and a drop zone.

- [ ] **Step 2: Upload a real sample `.xlsx`**

Click "Choose File", pick a real financial `.xlsx` workbook under 4MB. Expected: "Analyzing…" appears, then "Saving…", then the Dashboard screen renders with the company name, health score chart, KPI grid, and KPI bar chart populated from the real response.

- [ ] **Step 3: Confirm the analysis was actually persisted**

Click "History" in the nav. Expected: the just-uploaded analysis appears in the list with the correct company name and filename. Click it — the same dashboard reappears, now fetched via `GET /api/v1/analysis/history/{id}` rather than from local state.

- [ ] **Step 4: Confirm the 4MB validation message**

Attempt to pick or drop a file over 4MB (or a non-`.xlsx` file). Expected: an inline error appears immediately, with no network request made (confirm via a manual check of dev tools' Network tab, opened via `win.webContents.openDevTools()` if not already visible in dev mode).

- [ ] **Step 5: Record the result**

If any step fails, file it as a follow-up rather than editing this plan — the plan's job was to get to a working v1, not to anticipate every runtime surprise from live services outside this repo's control.
