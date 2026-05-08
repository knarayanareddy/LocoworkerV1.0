# Desktop Stack Decision: Electron (not Tauri)

## Summary
The desktop app uses **Electron**, not Tauri. This document explains why.

## Background
The original design spec (`Completeproject.md`) described the desktop app as "Tauri (Rust) + React". However, during the Pass 1-18 scaffold generation (specifically Pass 13), the implementation used **Electron** instead.

## Rationale for Electron
1. **Native module compatibility**: The project requires `keytar` (for OS keychain access) and `better-sqlite3` (for local databases). Both are Node.js native modules with mature Electron integration. Tauri's Node.js compatibility layer is less mature.

2. **Embedded Node.js child processes**: The desktop app spawns the Gateway (a Node.js app) as a child process. Electron makes this trivial via `child_process.spawn(process.execPath, ...)`. Tauri would require bundling a separate Node.js runtime.

3. **IPC model alignment**: Electron's `ipcMain`/`ipcRenderer` + `contextBridge` model maps cleanly to the "OS integration only" IPC surface described in Pass 24. Tauri's command/invoke model is more opinionated.

4. **Dashboard reuse**: Pass 23's dashboard is a React + Vite web app. Electron loads it directly. Tauri would require adapting the build (though this is minor).

## Trade-offs accepted
- **Bundle size**: Electron apps are ~150-200MB due to bundled Chromium. Tauri apps are ~10-20MB. We accept this trade-off for faster development.
- **Memory footprint**: Electron uses more RAM. We accept this for the target audience (developers with modern machines).

## Canonical stack (Pass 24 final)

Electron 30+ ├─ Main process (Node.js) │ ├─ Spawns Gateway child process (Node.js) │ ├─ Manages OS keychain via keytar │ └─ Exposes minimal IPC surface via ipcMain ├─ Preload script │ └─ Exposes window.locoworker bridge via contextBridge └─ Renderer process └─ Loads Dashboard (Pass 23) at locoworker-app:// custom protocol


## Migration note
If you see references to Tauri in `Completeproject.md`, treat them as superseded by this decision.
