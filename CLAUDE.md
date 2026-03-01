# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bottleneck is a fast GitHub PR review and branch management Electron desktop app for AI-native teams. It reproduces the core GitHub PR experience while being dramatically faster and optimized for parallelized background agents (Claude Code, Cursor, Devin, Codex).

**Must use npm** — bun is known not to work.

## Development Commands

```bash
npm run dev          # Full dev environment with hot reload (recommended)
npm run build        # Build all components for production
npm run dist         # Package the app for distribution (build + electron-builder)
npm start            # Build and run the electron app

npm test             # vitest run (single pass)
npm run test:watch   # vitest in watch mode
npx vitest run src/renderer/utils/__tests__/PrefixTrie.test.ts  # Run a single test file
```

Component-specific (rarely needed — `npm run dev` runs all three):
- `npm run dev:main` / `dev:preload` / `dev:renderer`

**No lint command exists.** ESLint deps are installed but there's no `.eslintrc` config file.

## Architecture

### Electron Multi-Process Model

Three separate TypeScript compilation targets, each with its own tsconfig:

| Process | Source | tsconfig | Output | Module System |
|---------|--------|----------|--------|---------------|
| Main | `src/main/` | `tsconfig.main.json` | `dist/main/` | CommonJS |
| Preload | `src/preload/` | `tsconfig.preload.json` | `dist/preload/` | CommonJS |
| Renderer | `src/renderer/` | `tsconfig.json` | `dist/renderer/` | ESM (Vite) |

Path aliases (configured in all tsconfigs + `vite.config.ts`):
- `@/*` → `src/renderer/*`
- `@main/*` → `src/main/*`
- `@shared/*` → `src/shared/*` (alias configured but directory does not exist yet)

### IPC Communication

All renderer↔main communication goes through the preload's `contextBridge` (`window.electron`). Context isolation is enabled.

**Renderer → Main** (request/response via `ipcRenderer.invoke` / `ipcMain.handle`):
All handlers return `{ success: boolean, data?: T, error?: string }` — never throw across IPC.

Key channels: `auth:*`, `git:*` (clone/checkout/pull/fetch/branches), `app:*` (directory picker, zoom, version), `settings:*` (get/set/clear via electron-store), `updater:*`.

**Main → Renderer** (one-way via `webContents.send`):
Menu-triggered actions, navigation commands, review actions, auto-updater events. The preload has an allowlist of subscribable channels.

### State Management (Zustand Stores)

Six stores with cross-store dependencies:

- **authStore** — auth state, token, user profile. Dev mode uses `"dev-token"` to route all API calls to mock data
- **prStore** — `Map<string, PullRequest>`, repository selection, filters, groups. Persists `selectedRepo`/`recentlyViewedRepos` to electron-store manually
- **syncStore** — orchestrates fetching repos + PRs. Imports and calls `prStore` directly via `getState()`. Has debounce timer
- **uiStore** — sidebar/panel toggles, theme, PR selection. `persist` middleware → localStorage (`"ui-storage"`)
- **settingsStore** — full settings + teams CRUD. `persist` → localStorage (`"settings-storage"`), also syncs to electron-store
- **branchStore** — three-tier cache: in-memory Map (5min TTL) → electron-store → GitHub API

Cross-store calls use `useXStore.getState()` to avoid React subscription overhead.

### Dev Mode / Mock Data

Setting `token === "dev-token"` (via `authStore.setUser()`) routes all API calls to `src/renderer/mockData.ts` with simulated delays. This lets the renderer run in a browser without Electron.

### Data Persistence

- **electron-store** (main process, JSON file in app userData): auth token, selected repo, branch cache, teams, settings
- **localStorage** (renderer): zustand persisted stores, lastSyncTime

### Build Pipeline (`scripts/dev.js`)

`npm run dev` does:
1. Initial tsc compile of main + preload (blocking)
2. Starts three watchers in parallel: tsc main, tsc preload, Vite dev server (port 3000)
3. Waits for Vite to print `"Local:"`, then launches Electron after 500ms delay
4. Kills all processes when Electron closes

Environment: reads `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` from `.env` (loaded via dotenv), passes them to Electron. Main process also checks `GITHUB_TOKEN` / `GITHUB_PERSONAL_ACCESS_TOKEN` env vars as PAT shortcuts.

### Code Splitting

All view components in `App.tsx` use `React.lazy()` + `Suspense`. Vite config splits Monaco Editor into its own chunk (`monaco`), with a separate `vendor` chunk for other node_modules.

### Performance Tooling

Renderer exposes global dev helpers: `window.perfSummary()`, `perfDetails()`, `perfTimeline()`, `perfExport()`, `perfMark()` via `PerfLogger` (uses `performance.now()`).

### PR Grouping

`PrefixTrie` in `src/renderer/utils/` normalizes branch names by stripping agent-generated suffixes (hex hashes, model names) to group related PRs. `prGrouping.ts` annotates PRs with `agent`, `titlePrefix`, and `labelNames` metadata.
