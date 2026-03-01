# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bottleneck is a fast GitHub PR review and branch management Electron desktop application. It reproduces the core GitHub PR experience while being dramatically faster and optimized for power-review workflows.

## Development Commands

### Core Commands
- `npm run dev` - Start development server with hot reload (recommended for development)
- `npm run build` - Build all components for production
- `npm run dist` - Package the app for distribution
- `npm start` - Build and run the electron app

### Component-specific Development
- `npm run dev:main` - Watch and compile main process only
- `npm run dev:preload` - Watch and compile preload script only
- `npm run dev:renderer` - Start Vite dev server for renderer only
- `npm run electron` - Run the built app

## Architecture

The application follows Electron's multi-process architecture:

### Main Process (`src/main/`)
- **index.ts** - Electron main process entry, window management, IPC handlers
- **auth.ts** - GitHub OAuth device flow authentication
- **database.ts** - SQLite database operations for local caching
- **git.ts** - Git operations using simple-git
- **menu.ts** - Application menu configuration

### Renderer Process (`src/renderer/`)
- **views/** - Main application views (PRListView, PRDetailView, BranchesView, SettingsView)
- **components/** - Reusable UI components
- **stores/** - Zustand state management (authStore, prStore, syncStore, uiStore)
- **services/** - API services for GitHub integration
- **utils/** - Utility functions

### Preload Script (`src/preload/`)
- Exposes safe IPC channels to renderer process

## Key Technologies

- **Electron 28** - Desktop app framework
- **React 18 + TypeScript** - UI framework
- **Monaco Editor** - VSCode-powered diff viewer
- **SQLite3** - Local data persistence
- **Zustand** - State management
- **React Query** - API state management
- **Tailwind CSS** - Styling
- **Vite** - Build tool for renderer process

## TypeScript Configuration

The project uses path aliases:
- `@/*` → `src/renderer/*`
- `@main/*` → `src/main/*`
- `@shared/*` → `src/shared/*`

Three TypeScript configs:
- `tsconfig.json` - Renderer process
- `tsconfig.main.json` - Main process
- `tsconfig.preload.json` - Preload script

## GitHub Integration

Authentication requires a GitHub OAuth app. The app uses device flow authentication (no backend required). Required environment variables in `.env`:
- `GITHUB_CLIENT_ID`
- `GITHUB_CLIENT_SECRET`

## Performance Considerations

The app is optimized for:
- PR list render: <300ms from cache, <1.5s cold fetch
- First diff paint: <150ms for typical files
- Handling 1k+ files / 50k+ changed lines smoothly
- 60 FPS scrolling in all views

Key optimizations include virtualized lists, web workers for diff computation, incremental syntax highlighting, smart caching with ETags, and concurrent API requests with rate limiting.