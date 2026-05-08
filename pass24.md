# Pass 24 — Desktop: Embedded Gateway + Keychain + IPC Re-architecture

> **"Later pass wins."** Every file path emitted here is the canonical final version.
> If a path appeared in Pass 1–23, this version replaces it entirely.

---

## Deliverables

| # | What | Where |
|---|------|--------|
| 1 | New `packages/keychain` package | OS-native secret storage (macOS Keychain / Windows Credential Store / Linux libsecret) |
| 2 | `apps/desktop/src/main/gatewayProcess.ts` | Spawn + manage embedded gateway child process, health-poll, port mapping |
| 3 | `apps/desktop/src/main/keychain.ts` | Wire `KeychainService` into desktop main, inject API keys into gateway env at spawn time |
| 4 | `apps/desktop/src/main/ipcHandlers.ts` | Re-arch all IPC handlers to be thin HTTP/SSE wrappers over embedded gateway |
| 5 | `apps/desktop/src/main/index.ts` | Rewrite: spawn gateway → poll health → load dashboard |
| 6 | `apps/desktop/src/main/tray.ts` | Update tray to reflect gateway status (starting / ready / error) |
| 7 | `apps/desktop/src/main/config.ts` | Update: strip secrets from disk config, delegate to keychain |
| 8 | `apps/desktop/src/preload/index.ts` | Expose `gatewayUrl`, `keychainGet/Set/Delete`, stream-bridge helpers |
| 9 | `apps/desktop/src/shared/ipcTypes.ts` | Add keychain channels + gateway-status channel |
| 10 | `apps/desktop/vite.main.config.ts` | Update aliases, ensure keychain native module is externalized |
| 11 | `apps/desktop/electron-builder.yml` | Include dashboard `dist/` as `extraResources`, add keychain native rebuild |
| 12 | `apps/desktop/package.json` | Add `keychain` workspace dep, `keytar`, `electron-rebuild`; remove direct engine deps |
| 13 | `apps/dashboard/vite.config.ts` | Add `base: "./"` so built assets load from `file://` inside Electron prod |

---

## Architectural shift (why Pass 24 matters)

Pass 13 ran the **full engine in-process** inside the Electron main process.
Pass 24 changes this to:
┌─────────────────────────────────────────────────┐ │ Electron main process │ │ │ │ ┌──────────────────┐ spawn ┌─────────────┐ │ │ │ GatewayProcess │──────────▶│ Gateway │ │ │ │ manager │ │ child proc │ │ │ │ (health poll, │◀──────────│ :3001 │ │ │ │ port mapping, │ ready/ │ (Pass 20 │ │ │ │ lifecycle) │ error │ runtime) │ │ │ └────────┬─────────┘ └──────┬──────┘ │ │ │ │ │ │ ┌────────▼─────────┐ │ │ │ │ ipcHandlers.ts │ HTTP calls │ │ │ │ (thin wrappers) │──────────────────┘ │ │ └────────┬─────────┘ │ │ │ ipcMain.handle / webContents.send │ └───────────┼─────────────────────────────────────┘ │ contextBridge ┌───────────▼─────────────────────────────────────┐ │ Renderer process (dashboard UI — Pass 23) │ │ loads http://127.0.0.1:5173 (dev) │ │ or file://.../dashboard/index.html (prod)│ │ │ │ Dashboard already speaks HTTP + EventSource │ │ → points VITE_LOCOWORKER_GATEWAY_URL at │ │ embedded gateway port (injected at runtime) │ └─────────────────────────────────────────────────┘

text


The dashboard renderer (Pass 23) **already knows how to talk to a gateway** via HTTP + SSE.
All Pass 24 has to do is:
1. guarantee a gateway is running at a known port,
2. tell the renderer what that URL is,
3. store/retrieve OS-level secrets via keychain,
4. keep the tray + lifecycle correct.

IPC handlers are reduced to **OS-integration surface only** (keychain, config, window management, tray).
Agent queries, sessions, memory, wiki, graph — all go through the gateway HTTP API as they would in any gateway client.

---

## Part 1 — `packages/keychain`

### `packages/keychain/package.json`

```json
{
  "name": "@locoworker/keychain",
  "version": "0.1.0",
  "description": "OS-native keychain storage for LocoWorker secrets",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "dev": "tsc -p tsconfig.json --watch",
    "clean": "rm -rf dist",
    "test": "vitest run"
  },
  "dependencies": {
    "keytar": "^7.9.0"
  },
  "devDependencies": {
    "@types/keytar": "^4.4.2",
    "typescript": "^5.4.5",
    "vitest": "^1.6.0"
  },
  "peerDependencies": {
    "electron": ">=28.0.0"
  },
  "peerDependenciesMeta": {
    "electron": { "optional": true }
  }
}
packages/keychain/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["dist", "node_modules"]
}
packages/keychain/src/types.ts
TypeScript

/**
 * A well-known keychain entry key understood by LocoWorker desktop.
 * Using string union so callers can also pass arbitrary strings.
 */
export type KeychainKey =
  | "anthropic-api-key"
  | "openai-api-key"
  | "google-api-key"
  | "locoworker-license"
  | (string & {});

export interface KeychainEntry {
  key: KeychainKey;
  value: string;
}

export interface IKeychainService {
  getPassword(key: KeychainKey): Promise<string | null>;
  setPassword(key: KeychainKey, value: string): Promise<void>;
  deletePassword(key: KeychainKey): Promise<boolean>;
  hasPassword(key: KeychainKey): Promise<boolean>;
  getAllKeys(): Promise<KeychainKey[]>;
}
packages/keychain/src/keychain.ts
TypeScript

import keytar from "keytar";
import type { IKeychainService, KeychainKey } from "./types.js";

/**
 * SERVICE_NAME used as the "service" identifier in the OS keychain.
 * All LocoWorker credentials are namespaced under this name so they
 * appear together in the OS credential manager.
 */
const SERVICE_NAME = "LocoWorker";

/**
 * KeychainService
 *
 * Thin wrapper around `keytar` that provides a stable interface across
 * macOS Keychain, Windows Credential Manager, and Linux libsecret.
 *
 * Every key is stored as:
 *   service = "LocoWorker"
 *   account = <key>      (e.g. "anthropic-api-key")
 *   password = <value>   (the secret string)
 */
export class KeychainService implements IKeychainService {
  async getPassword(key: KeychainKey): Promise<string | null> {
    try {
      return await keytar.getPassword(SERVICE_NAME, key);
    } catch (err) {
      console.error(`[keychain] getPassword failed for key="${key}":`, err);
      return null;
    }
  }

  async setPassword(key: KeychainKey, value: string): Promise<void> {
    try {
      await keytar.setPassword(SERVICE_NAME, key, value);
    } catch (err) {
      console.error(`[keychain] setPassword failed for key="${key}":`, err);
      throw new Error(`Failed to store secret "${key}" in OS keychain.`);
    }
  }

  async deletePassword(key: KeychainKey): Promise<boolean> {
    try {
      return await keytar.deletePassword(SERVICE_NAME, key);
    } catch (err) {
      console.error(`[keychain] deletePassword failed for key="${key}":`, err);
      return false;
    }
  }

  async hasPassword(key: KeychainKey): Promise<boolean> {
    const value = await this.getPassword(key);
    return value !== null && value.length > 0;
  }

  async getAllKeys(): Promise<KeychainKey[]> {
    try {
      const credentials = await keytar.findCredentials(SERVICE_NAME);
      return credentials.map((c) => c.account as KeychainKey);
    } catch (err) {
      console.error(`[keychain] getAllKeys failed:`, err);
      return [];
    }
  }
}
packages/keychain/src/index.ts
TypeScript

export { KeychainService } from "./keychain.js";
export type { IKeychainService, KeychainKey, KeychainEntry } from "./types.js";
Part 2 — Desktop package updates
apps/desktop/package.json
JSON

{
  "name": "@locoworker/desktop",
  "version": "0.1.0",
  "private": true,
  "main": "dist/main/index.js",
  "scripts": {
    "dev": "concurrently \"pnpm dev:vite\" \"pnpm dev:electron\"",
    "dev:vite": "vite build --watch --config vite.main.config.ts & vite build --watch --config vite.preload.config.ts",
    "dev:electron": "wait-on dist/main/index.js && electron .",
    "build": "pnpm build:vite && pnpm build:electron",
    "build:vite": "vite build --config vite.main.config.ts && vite build --config vite.preload.config.ts",
    "build:electron": "electron-builder --config electron-builder.yml",
    "rebuild-native": "electron-rebuild -f -w keytar",
    "postinstall": "electron-rebuild -f -w keytar",
    "clean": "rm -rf dist out"
  },
  "dependencies": {
    "@locoworker/keychain": "workspace:*",
    "electron-updater": "^6.1.8",
    "keytar": "^7.9.0"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "electron": "^30.0.0",
    "electron-builder": "^24.13.3",
    "electron-rebuild": "^3.2.9",
    "vite": "^5.2.11",
    "wait-on": "^7.2.0",
    "typescript": "^5.4.5"
  }
}
Note: Direct imports of @locoworker/core, @locoworker/memory, @locoworker/gateway etc. are removed from desktop's own dependencies. The engine stack runs inside the gateway child process; desktop no longer needs to link against it directly.

apps/desktop/electron-builder.yml
YAML

appId: dev.locoworker.desktop
productName: LocoWorker
copyright: Copyright © 2026 LocoWorker Contributors

# Where the built main/preload files live
directories:
  output: out
  buildResources: build

files:
  - dist/**/*
  - "!dist/renderer/**/*"   # renderer served by dashboard dev-server in dev / packed separately in prod

# In production, the dashboard's built dist/ is copied next to the app
# so Electron can load it via file://
extraResources:
  - from: "../dashboard/dist"
    to: "dashboard"
    filter:
      - "**/*"

# Gateway built output is also bundled so the child process can be spawned
  - from: "../gateway/dist"
    to: "gateway"
    filter:
      - "**/*"

# Also bundle gateway's node_modules (native addons etc.)
  - from: "../gateway/node_modules"
    to: "gateway/node_modules"
    filter:
      - "**/*"

nativeRebuilder:
  modules:
    - keytar

mac:
  category: public.app-category.developer-tools
  hardenedRuntime: true
  gatekeeperAssess: false
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.plist
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [x64, arm64]

win:
  target:
    - target: nsis
      arch: [x64]

linux:
  target:
    - target: AppImage
      arch: [x64]
  category: Development

publish:
  provider: github
  owner: knarayanareddy
  repo: LocoworkerV1.0
apps/desktop/vite.main.config.ts
TypeScript

import { defineConfig } from "vite";
import path from "path";

export default defineConfig({
  build: {
    lib: {
      entry: "src/main/index.ts",
      formats: ["cjs"],
      fileName: () => "index.js",
    },
    outDir: "dist/main",
    emptyOutDir: true,
    rollupOptions: {
      external: [
        "electron",
        "keytar",
        /^node:/,
        /^@locoworker\//,
        /^[a-z@][^/]*/,  // externalize all node_modules
      ],
    },
    sourcemap: true,
    minify: false,
  },
  resolve: {
    alias: {
      "@shared": path.resolve(__dirname, "src/shared"),
    },
  },
});
apps/desktop/vite.preload.config.ts
TypeScript

import { defineConfig } from "vite";
import path from "path";

export default defineConfig({
  build: {
    lib: {
      entry: "src/preload/index.ts",
      formats: ["cjs"],
      fileName: () => "preload.js",
    },
    outDir: "dist/preload",
    emptyOutDir: true,
    rollupOptions: {
      external: ["electron"],
    },
    sourcemap: true,
    minify: false,
  },
  resolve: {
    alias: {
      "@shared": path.resolve(__dirname, "src/shared"),
    },
  },
});
Part 3 — Shared IPC types (canonical, replaces Pass 13)
apps/desktop/src/shared/ipcTypes.ts
TypeScript

/**
 * Canonical IPC channel definitions for LocoWorker Desktop (Pass 24).
 *
 * ARCHITECTURE NOTE (Pass 24):
 *   Agent queries / sessions / memory / wiki / graph all go directly through
 *   the embedded Gateway's HTTP API — the dashboard renderer speaks HTTP+SSE
 *   natively (Pass 23).  IPC is now used only for OS-integration concerns:
 *   keychain, config, window management, gateway lifecycle, tray status.
 */

// ─── Gateway lifecycle ────────────────────────────────────────────────────────

export const IPC_GATEWAY_STATUS = "gateway:status";
export const IPC_GATEWAY_GET_URL = "gateway:getUrl";
export const IPC_GATEWAY_RESTART = "gateway:restart";

export type GatewayStatus =
  | { state: "starting" }
  | { state: "ready"; url: string }
  | { state: "error"; message: string }
  | { state: "stopped" };

// ─── Keychain ─────────────────────────────────────────────────────────────────

export const IPC_KEYCHAIN_GET = "keychain:get";
export const IPC_KEYCHAIN_SET = "keychain:set";
export const IPC_KEYCHAIN_DELETE = "keychain:delete";
export const IPC_KEYCHAIN_HAS = "keychain:has";
export const IPC_KEYCHAIN_LIST = "keychain:list";

export interface KeychainGetArgs { key: string }
export interface KeychainSetArgs { key: string; value: string }
export interface KeychainDeleteArgs { key: string }
export interface KeychainHasArgs { key: string }

// ─── Config ───────────────────────────────────────────────────────────────────

export const IPC_CONFIG_GET = "config:get";
export const IPC_CONFIG_SET = "config:set";
export const IPC_CONFIG_RESET = "config:reset";

export interface DesktopConfig {
  /**
   * Workspace directory. Passed to embedded gateway as LOCOWORKER_WORKSPACE.
   */
  workspace: string;
  /**
   * Override the embedded gateway port (default: 3001).
   * Maps to LOCOWORKER_GATEWAY_PORT in the gateway child process.
   */
  gatewayPort: number;
  /**
   * LLM provider selection for gateway.
   */
  provider: "anthropic" | "openai" | "google" | "openai-compatible";
  /**
   * Model override passed to gateway as LOCOWORKER_DEFAULT_MODEL.
   */
  model?: string;
  /**
   * Budget limit in USD, passed to gateway as LOCOWORKER_BUDGET_LIMIT.
   */
  budgetLimit?: number;
  /**
   * API keys are NOT stored here — they live in the OS keychain.
   * This comment is intentional: if you see API keys in the config file,
   * that is a bug.
   */
}

export const DEFAULT_DESKTOP_CONFIG: DesktopConfig = {
  workspace: "",          // resolved to homedir/locoworker at runtime if empty
  gatewayPort: 3001,
  provider: "anthropic",
};

// ─── Window ───────────────────────────────────────────────────────────────────

export const IPC_WINDOW_MINIMIZE = "window:minimize";
export const IPC_WINDOW_MAXIMIZE = "window:maximize";
export const IPC_WINDOW_CLOSE = "window:close";
export const IPC_WINDOW_IS_MAXIMIZED = "window:isMaximized";

// ─── App ──────────────────────────────────────────────────────────────────────

export const IPC_APP_GET_VERSION = "app:getVersion";
export const IPC_APP_GET_PLATFORM = "app:getPlatform";
export const IPC_APP_OPEN_EXTERNAL = "app:openExternal";
export const IPC_APP_OPEN_WORKSPACE = "app:openWorkspace";
Part 4 — Main process (full rewrite)
apps/desktop/src/main/gatewayProcess.ts
TypeScript

import { ChildProcess, spawn } from "child_process";
import { EventEmitter } from "events";
import path from "path";
import http from "http";
import { app } from "electron";
import type { GatewayStatus } from "../shared/ipcTypes.js";

export interface GatewayProcessOptions {
  port: number;
  workspace: string;
  env: Record<string, string | undefined>;
}

/**
 * GatewayProcessManager
 *
 * Spawns `apps/gateway/dist/main.js` (or its packaged equivalent under
 * resources/gateway/main.js) as a child process, polls /v1/health until
 * ready, emits status events, and handles clean shutdown.
 *
 * Events:
 *   "status"  GatewayStatus   fired whenever gateway status changes
 *   "log"     string          gateway stdout/stderr lines
 */
export class GatewayProcessManager extends EventEmitter {
  private child: ChildProcess | null = null;
  private _status: GatewayStatus = { state: "stopped" };
  private healthPollTimer: NodeJS.Timeout | null = null;
  private shuttingDown = false;

  get status(): GatewayStatus {
    return this._status;
  }

  get url(): string {
    return `http://127.0.0.1:${this.options.port}`;
  }

  constructor(private options: GatewayProcessOptions) {
    super();
  }

  /**
   * Resolve the gateway entry point.
   * - In production (packaged app): process.resourcesPath + /gateway/main.js
   * - In development: find the workspace root and use apps/gateway/dist/main.js
   */
  private resolveGatewayEntry(): string {
    if (app.isPackaged) {
      return path.join(process.resourcesPath, "gateway", "main.js");
    }
    // Dev: walk up from __dirname (dist/main/) to find monorepo root
    const devEntry = path.resolve(
      __dirname,
      "../../../../gateway/dist/main.js"
    );
    return devEntry;
  }

  async start(): Promise<void> {
    if (this.child) {
      await this.stop();
    }

    this.shuttingDown = false;
    this._setStatus({ state: "starting" });

    const gatewayEntry = this.resolveGatewayEntry();

    const env: NodeJS.ProcessEnv = {
      ...process.env,
      ...this.options.env,
      LOCOWORKER_GATEWAY_HOST: "127.0.0.1",
      LOCOWORKER_GATEWAY_PORT: String(this.options.port),
      LOCOWORKER_WORKSPACE: this.options.workspace,
      NODE_ENV: process.env.NODE_ENV ?? "production",
    };

    this.child = spawn(process.execPath, [gatewayEntry], {
      env,
      stdio: ["ignore", "pipe", "pipe"],
      detached: false,
    });

    this.child.stdout?.on("data", (chunk: Buffer) => {
      this.emit("log", chunk.toString());
    });

    this.child.stderr?.on("data", (chunk: Buffer) => {
      this.emit("log", `[stderr] ${chunk.toString()}`);
    });

    this.child.on("exit", (code, signal) => {
      if (!this.shuttingDown) {
        const message = `Gateway exited unexpectedly (code=${code}, signal=${signal})`;
        console.error(`[gatewayProcess] ${message}`);
        this._setStatus({ state: "error", message });
      }
    });

    this.child.on("error", (err) => {
      const message = `Failed to spawn gateway: ${err.message}`;
      console.error(`[gatewayProcess] ${message}`);
      this._setStatus({ state: "error", message });
    });

    // Poll health until ready or timeout (30 seconds)
    await this._pollHealth(30_000);
  }

  async stop(): Promise<void> {
    this.shuttingDown = true;
    this._clearHealthPoll();

    if (this.child && !this.child.killed) {
      this.child.kill("SIGTERM");
      await new Promise<void>((resolve) => {
        const t = setTimeout(() => {
          this.child?.kill("SIGKILL");
          resolve();
        }, 5_000);
        this.child?.once("exit", () => {
          clearTimeout(t);
          resolve();
        });
      });
    }

    this.child = null;
    this._setStatus({ state: "stopped" });
  }

  async restart(): Promise<void> {
    await this.stop();
    this.shuttingDown = false;
    await this.start();
  }

  private _setStatus(status: GatewayStatus): void {
    this._status = status;
    this.emit("status", status);
  }

  private _clearHealthPoll(): void {
    if (this.healthPollTimer) {
      clearTimeout(this.healthPollTimer);
      this.healthPollTimer = null;
    }
  }

  private async _pollHealth(timeoutMs: number): Promise<void> {
    const start = Date.now();
    const interval = 250; // ms between polls
    const url = `${this.url}/v1/health`;

    return new Promise((resolve, reject) => {
      const poll = () => {
        if (this.shuttingDown) {
          return resolve();
        }

        if (Date.now() - start > timeoutMs) {
          const message = `Gateway did not become healthy within ${timeoutMs}ms`;
          this._setStatus({ state: "error", message });
          return reject(new Error(message));
        }

        http
          .get(url, (res) => {
            if (res.statusCode === 200) {
              this._setStatus({ state: "ready", url: this.url });
              resolve();
            } else {
              this.healthPollTimer = setTimeout(poll, interval);
            }
            res.resume();
          })
          .on("error", () => {
            // Gateway not up yet — keep polling
            this.healthPollTimer = setTimeout(poll, interval);
          });
      };

      poll();
    });
  }
}
apps/desktop/src/main/keychain.ts
TypeScript

import { KeychainService } from "@locoworker/keychain";
import type { KeychainKey } from "@locoworker/keychain";
import { ipcMain } from "electron";
import {
  IPC_KEYCHAIN_GET,
  IPC_KEYCHAIN_SET,
  IPC_KEYCHAIN_DELETE,
  IPC_KEYCHAIN_HAS,
  IPC_KEYCHAIN_LIST,
  type KeychainGetArgs,
  type KeychainSetArgs,
  type KeychainDeleteArgs,
  type KeychainHasArgs,
} from "../shared/ipcTypes.js";

const service = new KeychainService();

/**
 * registerKeychainHandlers
 *
 * Wires keychain IPC handlers. Called once from main index.ts.
 */
export function registerKeychainHandlers(): void {
  ipcMain.handle(IPC_KEYCHAIN_GET, async (_event, { key }: KeychainGetArgs) => {
    return service.getPassword(key as KeychainKey);
  });

  ipcMain.handle(IPC_KEYCHAIN_SET, async (_event, { key, value }: KeychainSetArgs) => {
    await service.setPassword(key as KeychainKey, value);
    return true;
  });

  ipcMain.handle(IPC_KEYCHAIN_DELETE, async (_event, { key }: KeychainDeleteArgs) => {
    return service.deletePassword(key as KeychainKey);
  });

  ipcMain.handle(IPC_KEYCHAIN_HAS, async (_event, { key }: KeychainHasArgs) => {
    return service.hasPassword(key as KeychainKey);
  });

  ipcMain.handle(IPC_KEYCHAIN_LIST, async () => {
    return service.getAllKeys();
  });
}

/**
 * resolveApiKeys
 *
 * Read API keys from keychain and return them as a partial env-var record
 * suitable for merging into the gateway child process environment.
 */
export async function resolveApiKeys(): Promise<Record<string, string>> {
  const env: Record<string, string> = {};

  const anthropic = await service.getPassword("anthropic-api-key");
  if (anthropic) env.ANTHROPIC_API_KEY = anthropic;

  const openai = await service.getPassword("openai-api-key");
  if (openai) env.OPENAI_API_KEY = openai;

  const google = await service.getPassword("google-api-key");
  if (google) env.GOOGLE_API_KEY = google;

  return env;
}
apps/desktop/src/main/config.ts
TypeScript

import path from "path";
import fs from "fs";
import os from "os";
import { app } from "electron";
import {
  DEFAULT_DESKTOP_CONFIG,
  type DesktopConfig,
} from "../shared/ipcTypes.js";

function getConfigPath(): string {
  return path.join(app.getPath("userData"), "config.json");
}

function getDefaultWorkspace(): string {
  return path.join(os.homedir(), "locoworker");
}

/**
 * Load config from disk, merging with defaults.
 * API keys are NEVER stored in this file — they live in the OS keychain.
 */
export function loadConfig(): DesktopConfig {
  const configPath = getConfigPath();
  let saved: Partial<DesktopConfig> = {};

  if (fs.existsSync(configPath)) {
    try {
      const raw = fs.readFileSync(configPath, "utf8");
      saved = JSON.parse(raw) as Partial<DesktopConfig>;
    } catch {
      console.warn("[config] Failed to parse config.json — using defaults.");
    }
  }

  const merged: DesktopConfig = {
    ...DEFAULT_DESKTOP_CONFIG,
    ...saved,
    workspace: saved.workspace || getDefaultWorkspace(),
  };

  // Defensive: ensure workspace exists
  if (!fs.existsSync(merged.workspace)) {
    fs.mkdirSync(merged.workspace, { recursive: true });
  }

  return merged;
}

/**
 * Persist config to disk.
 * NEVER write API keys to the config file.
 */
export function saveConfig(config: DesktopConfig): void {
  const configPath = getConfigPath();
  const safe: DesktopConfig = { ...config };

  // Ensure no accidental secret leakage
  // (TypeScript already enforces this but belt-and-suspenders)
  const asAny = safe as Record<string, unknown>;
  delete asAny.anthropicApiKey;
  delete asAny.openaiApiKey;
  delete asAny.googleApiKey;

  fs.writeFileSync(configPath, JSON.stringify(safe, null, 2), "utf8");
}
apps/desktop/src/main/tray.ts
TypeScript

import { app, Menu, Tray, nativeImage, BrowserWindow, shell } from "electron";
import path from "path";
import type { GatewayStatus } from "../shared/ipcTypes.js";

let tray: Tray | null = null;

function getTrayIcon(status: GatewayStatus["state"]): Electron.NativeImage {
  // Attempt to load themed icons from build resources; fall back to empty image
  const iconFile = (() => {
    switch (status) {
      case "ready":    return "tray-ready.png";
      case "starting": return "tray-starting.png";
      case "error":    return "tray-error.png";
      default:         return "tray-stopped.png";
    }
  })();

  const iconPath = path.join(
    app.isPackaged ? process.resourcesPath : path.resolve(__dirname, "../../../build"),
    iconFile
  );

  try {
    return nativeImage.createFromPath(iconPath);
  } catch {
    return nativeImage.createEmpty();
  }
}

function buildContextMenu(
  status: GatewayStatus,
  win: BrowserWindow | null,
  onRestart: () => void
): Electron.Menu {
  const statusLabel = (() => {
    switch (status.state) {
      case "ready":    return `Gateway: ready (${(status as { url: string }).url})`;
      case "starting": return "Gateway: starting…";
      case "error":    return `Gateway: error — ${(status as { message: string }).message}`;
      default:         return "Gateway: stopped";
    }
  })();

  return Menu.buildFromTemplate([
    { label: "LocoWorker", enabled: false },
    { type: "separator" },
    { label: statusLabel, enabled: false },
    { type: "separator" },
    {
      label: "Show Window",
      click: () => {
        if (win) {
          win.show();
          win.focus();
        }
      },
    },
    {
      label: "Restart Gateway",
      enabled: status.state !== "starting",
      click: onRestart,
    },
    {
      label: "Open Workspace Folder",
      click: () => {
        // workspace path is exposed via ipc; open what we can
        shell.openPath(app.getPath("userData"));
      },
    },
    { type: "separator" },
    {
      label: "Quit",
      click: () => app.quit(),
    },
  ]);
}

export function createTray(
  win: BrowserWindow | null,
  initialStatus: GatewayStatus,
  onRestart: () => void
): Tray {
  tray = new Tray(getTrayIcon(initialStatus.state));
  tray.setToolTip("LocoWorker");
  tray.setContextMenu(buildContextMenu(initialStatus, win, onRestart));

  tray.on("click", () => {
    if (win) {
      if (win.isVisible()) {
        win.focus();
      } else {
        win.show();
        win.focus();
      }
    }
  });

  return tray;
}

export function updateTrayStatus(
  status: GatewayStatus,
  win: BrowserWindow | null,
  onRestart: () => void
): void {
  if (!tray) return;
  tray.setImage(getTrayIcon(status.state));
  tray.setContextMenu(buildContextMenu(status, win, onRestart));

  const tooltip = status.state === "ready"
    ? `LocoWorker — gateway ready`
    : status.state === "error"
    ? `LocoWorker — gateway error`
    : `LocoWorker — ${status.state}`;
  tray.setToolTip(tooltip);
}

export function destroyTray(): void {
  tray?.destroy();
  tray = null;
}
apps/desktop/src/main/ipcHandlers.ts
TypeScript

/**
 * ipcHandlers.ts (Pass 24 — OS-integration only)
 *
 * ALL agent / session / memory / wiki / graph operations go directly through
 * the embedded gateway's HTTP API (the dashboard renderer handles this via
 * GatewayClient from Pass 23).
 *
 * This file handles ONLY:
 *   - Gateway lifecycle queries
 *   - Config get/set
 *   - Window controls
 *   - App utilities
 *
 * Keychain handlers are registered separately in keychain.ts.
 */

import {
  app,
  ipcMain,
  shell,
  BrowserWindow,
  dialog,
} from "electron";
import {
  IPC_GATEWAY_STATUS,
  IPC_GATEWAY_GET_URL,
  IPC_GATEWAY_RESTART,
  IPC_CONFIG_GET,
  IPC_CONFIG_SET,
  IPC_CONFIG_RESET,
  IPC_WINDOW_MINIMIZE,
  IPC_WINDOW_MAXIMIZE,
  IPC_WINDOW_CLOSE,
  IPC_WINDOW_IS_MAXIMIZED,
  IPC_APP_GET_VERSION,
  IPC_APP_GET_PLATFORM,
  IPC_APP_OPEN_EXTERNAL,
  IPC_APP_OPEN_WORKSPACE,
  DEFAULT_DESKTOP_CONFIG,
  type DesktopConfig,
} from "../shared/ipcTypes.js";
import { loadConfig, saveConfig } from "./config.js";
import type { GatewayProcessManager } from "./gatewayProcess.js";

export function registerIpcHandlers(
  gateway: GatewayProcessManager,
  getWindow: () => BrowserWindow | null
): void {
  // ── Gateway lifecycle ──────────────────────────────────────────────────────

  ipcMain.handle(IPC_GATEWAY_STATUS, () => gateway.status);
  ipcMain.handle(IPC_GATEWAY_GET_URL, () => gateway.url);
  ipcMain.handle(IPC_GATEWAY_RESTART, async () => {
    await gateway.restart();
    return gateway.status;
  });

  // ── Config ─────────────────────────────────────────────────────────────────

  ipcMain.handle(IPC_CONFIG_GET, () => loadConfig());

  ipcMain.handle(IPC_CONFIG_SET, (_event, partial: Partial<DesktopConfig>) => {
    const current = loadConfig();
    const updated: DesktopConfig = { ...current, ...partial };
    saveConfig(updated);
    return updated;
  });

  ipcMain.handle(IPC_CONFIG_RESET, () => {
    saveConfig(DEFAULT_DESKTOP_CONFIG);
    return DEFAULT_DESKTOP_CONFIG;
  });

  // ── Window controls ────────────────────────────────────────────────────────

  ipcMain.on(IPC_WINDOW_MINIMIZE, () => getWindow()?.minimize());
  ipcMain.on(IPC_WINDOW_MAXIMIZE, () => {
    const win = getWindow();
    if (!win) return;
    win.isMaximized() ? win.unmaximize() : win.maximize();
  });
  ipcMain.on(IPC_WINDOW_CLOSE, () => getWindow()?.close());
  ipcMain.handle(IPC_WINDOW_IS_MAXIMIZED, () => getWindow()?.isMaximized() ?? false);

  // ── App utilities ──────────────────────────────────────────────────────────

  ipcMain.handle(IPC_APP_GET_VERSION, () => app.getVersion());
  ipcMain.handle(IPC_APP_GET_PLATFORM, () => process.platform);

  ipcMain.handle(IPC_APP_OPEN_EXTERNAL, async (_event, url: string) => {
    // Safety: only allow https:// and http:// links
    if (/^https?:\/\//i.test(url)) {
      await shell.openExternal(url);
      return true;
    }
    return false;
  });

  ipcMain.handle(IPC_APP_OPEN_WORKSPACE, async () => {
    const config = loadConfig();
    const result = await dialog.showOpenDialog({
      defaultPath: config.workspace,
      properties: ["openDirectory", "createDirectory"],
      title: "Select LocoWorker Workspace",
    });
    if (!result.canceled && result.filePaths[0]) {
      saveConfig({ ...config, workspace: result.filePaths[0] });
      return result.filePaths[0];
    }
    return null;
  });
}
apps/desktop/src/main/updater.ts
TypeScript

import { autoUpdater } from "electron-updater";
import { BrowserWindow } from "electron";

let updateCheckScheduled = false;

export function setupAutoUpdater(getWindow: () => BrowserWindow | null): void {
  autoUpdater.autoDownload = false;
  autoUpdater.autoInstallOnAppQuit = true;

  autoUpdater.on("update-available", (info) => {
    const win = getWindow();
    if (win) {
      win.webContents.send("updater:update-available", info);
    }
  });

  autoUpdater.on("update-downloaded", (info) => {
    const win = getWindow();
    if (win) {
      win.webContents.send("updater:update-downloaded", info);
    }
  });

  autoUpdater.on("error", (err) => {
    console.warn("[updater] Auto-update error:", err.message);
  });
}

export async function checkForUpdates(): Promise<void> {
  if (updateCheckScheduled) return;
  updateCheckScheduled = true;
  try {
    await autoUpdater.checkForUpdates();
  } catch (err) {
    console.warn("[updater] checkForUpdates failed:", err);
  }
}
apps/desktop/src/main/index.ts ← full rewrite
TypeScript

import {
  app,
  BrowserWindow,
  protocol,
  net,
  session,
} from "electron";
import path from "path";
import { pathToFileURL } from "url";
import { GatewayProcessManager } from "./gatewayProcess.js";
import { registerKeychainHandlers, resolveApiKeys } from "./keychain.js";
import { registerIpcHandlers } from "./ipcHandlers.js";
import { createTray, updateTrayStatus, destroyTray } from "./tray.js";
import { setupAutoUpdater, checkForUpdates } from "./updater.js";
import { loadConfig } from "./config.js";
import {
  IPC_GATEWAY_STATUS,
  type GatewayStatus,
} from "../shared/ipcTypes.js";

// ── Globals ──────────────────────────────────────────────────────────────────

let mainWindow: BrowserWindow | null = null;
let gatewayManager: GatewayProcessManager | null = null;

// ── Dev / Prod helpers ───────────────────────────────────────────────────────

const IS_DEV = !app.isPackaged;

/**
 * In development the dashboard is served by the Vite dev server on :5173.
 * In production the dashboard is copied to resources/dashboard/ by
 * electron-builder (see extraResources in electron-builder.yml).
 */
function getDashboardUrl(): string {
  if (IS_DEV) {
    return "http://127.0.0.1:5173";
  }
  return "locoworker-app://dashboard/index.html";
}

// ── Custom protocol (production only) ────────────────────────────────────────

/**
 * Register a custom protocol `locoworker-app://` that serves files from the
 * packaged resources directory.  This avoids file:// origin issues (CORS,
 * localStorage partitioning, EventSource same-origin checks).
 */
function registerAppProtocol(): void {
  protocol.registerSchemesAsPrivileged([
    {
      scheme: "locoworker-app",
      privileges: {
        standard: true,
        secure: true,
        supportFetchAPI: true,
        stream: true,
        allowServiceWorkers: true,
      },
    },
  ]);
}

function handleAppProtocol(): void {
  protocol.handle("locoworker-app", (request) => {
    const url = new URL(request.url);
    // url.pathname is like /dashboard/index.html
    const filePath = path.join(
      process.resourcesPath,
      url.host,         // "dashboard"
      url.pathname      // "/index.html"
    );
    return net.fetch(pathToFileURL(filePath).toString());
  });
}

// ── Window ───────────────────────────────────────────────────────────────────

function createWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1280,
    height: 800,
    minWidth: 900,
    minHeight: 600,
    title: "LocoWorker",
    show: false,                   // show after gateway is ready
    backgroundColor: "#0f0f0f",
    titleBarStyle: process.platform === "darwin" ? "hiddenInset" : "default",
    webPreferences: {
      preload: path.join(__dirname, "../preload/preload.js"),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      // Allow EventSource to connect to the embedded gateway from the
      // locoworker-app:// origin in production.
      webSecurity: !IS_DEV,
    },
  });

  win.on("ready-to-show", () => {
    win.show();
    if (IS_DEV) {
      win.webContents.openDevTools({ mode: "detach" });
    }
  });

  win.on("closed", () => {
    mainWindow = null;
  });

  return win;
}

// ── Boot sequence ─────────────────────────────────────────────────────────────

async function boot(): Promise<void> {
  const config = loadConfig();

  // 1) Resolve API keys from keychain to inject into gateway env
  const apiKeyEnv = await resolveApiKeys();

  // 2) Build gateway manager
  gatewayManager = new GatewayProcessManager({
    port: config.gatewayPort,
    workspace: config.workspace,
    env: {
      ...apiKeyEnv,
      LOCOWORKER_DEFAULT_MODEL: config.model ?? "",
      LOCOWORKER_BUDGET_LIMIT: config.budgetLimit ? String(config.budgetLimit) : "",
      LOCOWORKER_CORS_ORIGINS: getDashboardUrl(),
    },
  });

  // 3) Register IPC
  registerKeychainHandlers();
  registerIpcHandlers(gatewayManager, () => mainWindow);

  // 4) Create window (hidden until gateway ready)
  mainWindow = createWindow();

  if (!IS_DEV) {
    handleAppProtocol();
    // Permit EventSource from locoworker-app:// to connect to gateway on 127.0.0.1
    session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
      callback({
        responseHeaders: {
          ...details.responseHeaders,
          "Access-Control-Allow-Origin": ["*"],
        },
      });
    });
  }

  // 5) Subscribe to gateway status for tray + window + renderer updates
  gatewayManager.on("status", (status: GatewayStatus) => {
    updateTrayStatus(status, mainWindow, () => gatewayManager?.restart());

    // Forward status to renderer so ConnectionPanel (Pass 23) can react
    mainWindow?.webContents.send(IPC_GATEWAY_STATUS, status);

    if (status.state === "ready" && mainWindow) {
      // Load the dashboard pointing at the embedded gateway
      const url = getDashboardUrl();
      mainWindow.loadURL(url).catch((err) => {
        console.error("[desktop] Failed to load dashboard URL:", err);
      });
    }
  });

  // 6) Create tray (before gateway starts — shows "starting" state)
  createTray(mainWindow, { state: "starting" }, () =>
    gatewayManager?.restart()
  );

  // 7) Start gateway (async — status events drive the rest)
  gatewayManager.start().catch((err) => {
    console.error("[desktop] Gateway failed to start:", err);
  });

  // 8) Auto-updater (check 10s after boot to not block startup)
  setupAutoUpdater(() => mainWindow);
  setTimeout(() => checkForUpdates(), 10_000);
}

// ── App lifecycle ─────────────────────────────────────────────────────────────

registerAppProtocol(); // must be called before app.whenReady()

app.on("window-all-closed", () => {
  // On macOS keep the app running in the tray when all windows are closed
  if (process.platform !== "darwin") {
    app.quit();
  }
});

app.on("activate", () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    mainWindow = createWindow();
    if (gatewayManager?.status.state === "ready") {
      mainWindow.loadURL(getDashboardUrl()).catch(console.error);
    }
  } else {
    mainWindow?.show();
  }
});

app.on("before-quit", async () => {
  destroyTray();
  await gatewayManager?.stop();
});

app.whenReady().then(boot).catch((err) => {
  console.error("[desktop] Boot failed:", err);
  app.quit();
});
Part 5 — Preload (canonical rewrite)
apps/desktop/src/preload/index.ts
TypeScript

/**
 * LocoWorker Desktop Preload (Pass 24)
 *
 * Exposes a minimal, typed `window.locoworker` API to the renderer via
 * contextBridge.  All agent/session/memory operations go through the
 * embedded gateway's HTTP API directly — the dashboard renderer (Pass 23)
 * already does this via GatewayClient.
 *
 * This preload exposes only OS-integration surface:
 *   - gateway URL + status
 *   - keychain read/write/delete
 *   - config get/set
 *   - window controls
 *   - app utilities
 *   - IPC event subscription helpers
 */

import { contextBridge, ipcRenderer } from "electron";
import {
  IPC_GATEWAY_STATUS,
  IPC_GATEWAY_GET_URL,
  IPC_GATEWAY_RESTART,
  IPC_KEYCHAIN_GET,
  IPC_KEYCHAIN_SET,
  IPC_KEYCHAIN_DELETE,
  IPC_KEYCHAIN_HAS,
  IPC_KEYCHAIN_LIST,
  IPC_CONFIG_GET,
  IPC_CONFIG_SET,
  IPC_CONFIG_RESET,
  IPC_WINDOW_MINIMIZE,
  IPC_WINDOW_MAXIMIZE,
  IPC_WINDOW_CLOSE,
  IPC_WINDOW_IS_MAXIMIZED,
  IPC_APP_GET_VERSION,
  IPC_APP_GET_PLATFORM,
  IPC_APP_OPEN_EXTERNAL,
  IPC_APP_OPEN_WORKSPACE,
  type GatewayStatus,
  type DesktopConfig,
} from "../shared/ipcTypes.js";

export type LocoWorkerBridge = typeof bridge;

const bridge = {
  // ── Gateway ────────────────────────────────────────────────────────────────

  gateway: {
    getStatus: (): Promise<GatewayStatus> =>
      ipcRenderer.invoke(IPC_GATEWAY_STATUS),

    getUrl: (): Promise<string> =>
      ipcRenderer.invoke(IPC_GATEWAY_GET_URL),

    restart: (): Promise<GatewayStatus> =>
      ipcRenderer.invoke(IPC_GATEWAY_RESTART),

    /**
     * Subscribe to gateway status changes.
     * Returns an unsubscribe function.
     */
    onStatus: (cb: (status: GatewayStatus) => void): (() => void) => {
      const handler = (_event: Electron.IpcRendererEvent, status: GatewayStatus) =>
        cb(status);
      ipcRenderer.on(IPC_GATEWAY_STATUS, handler);
      return () => ipcRenderer.removeListener(IPC_GATEWAY_STATUS, handler);
    },
  },

  // ── Keychain ───────────────────────────────────────────────────────────────

  keychain: {
    get: (key: string): Promise<string | null> =>
      ipcRenderer.invoke(IPC_KEYCHAIN_GET, { key }),

    set: (key: string, value: string): Promise<boolean> =>
      ipcRenderer.invoke(IPC_KEYCHAIN_SET, { key, value }),

    delete: (key: string): Promise<boolean> =>
      ipcRenderer.invoke(IPC_KEYCHAIN_DELETE, { key }),

    has: (key: string): Promise<boolean> =>
      ipcRenderer.invoke(IPC_KEYCHAIN_HAS, { key }),

    list: (): Promise<string[]> =>
      ipcRenderer.invoke(IPC_KEYCHAIN_LIST),
  },

  // ── Config ─────────────────────────────────────────────────────────────────

  config: {
    get: (): Promise<DesktopConfig> =>
      ipcRenderer.invoke(IPC_CONFIG_GET),

    set: (partial: Partial<DesktopConfig>): Promise<DesktopConfig> =>
      ipcRenderer.invoke(IPC_CONFIG_SET, partial),

    reset: (): Promise<DesktopConfig> =>
      ipcRenderer.invoke(IPC_CONFIG_RESET),
  },

  // ── Window controls ────────────────────────────────────────────────────────

  window: {
    minimize: () => ipcRenderer.send(IPC_WINDOW_MINIMIZE),
    maximize: () => ipcRenderer.send(IPC_WINDOW_MAXIMIZE),
    close: () => ipcRenderer.send(IPC_WINDOW_CLOSE),
    isMaximized: (): Promise<boolean> =>
      ipcRenderer.invoke(IPC_WINDOW_IS_MAXIMIZED),
  },

  // ── App utilities ──────────────────────────────────────────────────────────

  app: {
    getVersion: (): Promise<string> =>
      ipcRenderer.invoke(IPC_APP_GET_VERSION),

    getPlatform: (): Promise<string> =>
      ipcRenderer.invoke(IPC_APP_GET_PLATFORM),

    openExternal: (url: string): Promise<boolean> =>
      ipcRenderer.invoke(IPC_APP_OPEN_EXTERNAL, url),

    openWorkspace: (): Promise<string | null> =>
      ipcRenderer.invoke(IPC_APP_OPEN_WORKSPACE),
  },

  // ── Generic event subscription (for updater events etc.) ──────────────────

  on: (
    channel: string,
    cb: (...args: unknown[]) => void
  ): (() => void) => {
    const ALLOWED_CHANNELS = [
      IPC_GATEWAY_STATUS,
      "updater:update-available",
      "updater:update-downloaded",
    ];
    if (!ALLOWED_CHANNELS.includes(channel)) {
      console.warn(`[preload] Blocked subscription to unlisted channel: ${channel}`);
      return () => {};
    }
    const handler = (_event: Electron.IpcRendererEvent, ...args: unknown[]) =>
      cb(...args);
    ipcRenderer.on(channel, handler);
    return () => ipcRenderer.removeListener(channel, handler);
  },
};

contextBridge.exposeInMainWorld("locoworker", bridge);
Part 6 — Dashboard: vite.config.ts patch
apps/dashboard/vite.config.ts ← patch: add base: "./" for production Electron loading
TypeScript

import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig(({ mode }) => ({
  plugins: [react()],

  /**
   * base: "./" is required so that in production, when the dashboard is
   * loaded from the locoworker-app:// custom protocol (or file://), all
   * asset paths are relative rather than absolute.
   *
   * In dev, Vite's dev server handles this transparently regardless.
   */
  base: "./",

  server: {
    port: 5173,
    host: "127.0.0.1",
    strictPort: true,
  },

  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },

  build: {
    outDir: "dist",
    emptyOutDir: true,
    sourcemap: mode === "development",
    rollupOptions: {
      output: {
        manualChunks: {
          react: ["react", "react-dom"],
          zustand: ["zustand"],
        },
      },
    },
  },

  define: {
    /**
     * In production desktop builds, VITE_LOCOWORKER_GATEWAY_URL is NOT
     * injected at Vite build time (the gateway port is only known at
     * Electron runtime).  Instead, the dashboard's GatewayClient reads
     * window.__LOCOWORKER_GATEWAY_URL__ which is set by a small inline
     * script injected by the Electron main process.
     *
     * In dev and standalone web builds, fall back to the standard env var.
     */
  },
}));
Runtime gateway URL injection: because the gateway port is only known at Electron runtime, the Electron main process injects it into the renderer after the page loads via webContents.executeJavaScript. This means GatewayClient in Pass 23's store must check window.__LOCOWORKER_GATEWAY_URL__ before import.meta.env.VITE_LOCOWORKER_GATEWAY_URL.

Part 7 — Dashboard store patch (gateway URL injection hook)
apps/dashboard/src/store/gatewaySlice.ts ← patch: read injected URL
Add to the top of createGatewaySlice initial state resolution:

TypeScript

/**
 * Resolve the gateway URL at runtime.
 * Priority order:
 *   1. window.__LOCOWORKER_GATEWAY_URL__  (injected by Electron main at runtime)
 *   2. import.meta.env.VITE_LOCOWORKER_GATEWAY_URL  (build-time env, dev/standalone)
 *   3. http://127.0.0.1:8787  (fallback default from Pass 23 gateway config)
 */
function resolveInitialGatewayUrl(): string {
  if (
    typeof window !== "undefined" &&
    (window as unknown as Record<string, unknown>).__LOCOWORKER_GATEWAY_URL__
  ) {
    return String(
      (window as unknown as Record<string, unknown>).__LOCOWORKER_GATEWAY_URL__
    );
  }
  return import.meta.env.VITE_LOCOWORKER_GATEWAY_URL ?? "http://127.0.0.1:8787";
}
Replace the hardcoded initial gatewayUrl in the slice with:

TypeScript

gatewayUrl: resolveInitialGatewayUrl(),
And in apps/desktop/src/main/index.ts, after mainWindow.loadURL(url) resolves in the "ready" handler, inject the runtime gateway URL:

TypeScript

// Inject gateway URL into renderer so GatewayClient uses the right port
mainWindow.webContents.executeJavaScript(
  `window.__LOCOWORKER_GATEWAY_URL__ = "${gatewayManager.url}";`
).catch(console.error);
Part 8 — Build script wiring
apps/desktop/scripts/copy-dashboard.sh
Bash

#!/usr/bin/env bash
# Copies the built dashboard dist into apps/desktop/dist/dashboard
# for dev inspection. Electron-builder handles the real copy for
# packaged builds via extraResources in electron-builder.yml.

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DESKTOP_DIR="$(dirname "$SCRIPT_DIR")"
REPO_ROOT="$(dirname "$(dirname "$DESKTOP_DIR")")"

DASHBOARD_DIST="$REPO_ROOT/apps/dashboard/dist"
TARGET="$DESKTOP_DIR/dist/dashboard"

if [ ! -d "$DASHBOARD_DIST" ]; then
  echo "[copy-dashboard] Building dashboard first..."
  (cd "$REPO_ROOT/apps/dashboard" && pnpm build)
fi

echo "[copy-dashboard] Copying $DASHBOARD_DIST → $TARGET"
rm -rf "$TARGET"
cp -r "$DASHBOARD_DIST" "$TARGET"
echo "[copy-dashboard] Done."
turbo.json patch (root level — add desktop build dependency on dashboard build)
In the existing turbo.json pipeline, ensure desktop's build step depends on dashboard's build:

JSON

{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "@locoworker/desktop#build": {
      "dependsOn": ["@locoworker/dashboard#build", "@locoworker/gateway#build", "^build"],
      "outputs": ["dist/**", "out/**"]
    }
  }
}
Part 9 — packages/keychain test
packages/keychain/src/keychain.test.ts
TypeScript

import { describe, it, expect, vi, beforeEach } from "vitest";
import { KeychainService } from "./keychain.js";

// Mock keytar so tests don't require OS keychain access
vi.mock("keytar", () => {
  const store = new Map<string, Map<string, string>>();

  return {
    default: {
      getPassword: vi.fn(async (service: string, account: string) => {
        return store.get(service)?.get(account) ?? null;
      }),
      setPassword: vi.fn(async (service: string, account: string, password: string) => {
        if (!store.has(service)) store.set(service, new Map());
        store.get(service)!.set(account, password);
      }),
      deletePassword: vi.fn(async (service: string, account: string) => {
        return store.get(service)?.delete(account) ?? false;
      }),
      findCredentials: vi.fn(async (service: string) => {
        const accounts = store.get(service);
        if (!accounts) return [];
        return Array.from(accounts.entries()).map(([account, password]) => ({
          account,
          password,
        }));
      }),
    },
  };
});

describe("KeychainService", () => {
  let svc: KeychainService;

  beforeEach(() => {
    svc = new KeychainService();
    vi.clearAllMocks();
  });

  it("returns null for unknown key", async () => {
    const result = await svc.getPassword("anthropic-api-key");
    expect(result).toBeNull();
  });

  it("stores and retrieves a secret", async () => {
    await svc.setPassword("anthropic-api-key", "sk-test-1234");
    const result = await svc.getPassword("anthropic-api-key");
    expect(result).toBe("sk-test-1234");
  });

  it("deletes a secret", async () => {
    await svc.setPassword("openai-api-key", "sk-openai-5678");
    const deleted = await svc.deletePassword("openai-api-key");
    expect(deleted).toBe(true);
    const result = await svc.getPassword("openai-api-key");
    expect(result).toBeNull();
  });

  it("hasPassword returns true when secret exists", async () => {
    await svc.setPassword("google-api-key", "google-abc");
    expect(await svc.hasPassword("google-api-key")).toBe(true);
  });

  it("hasPassword returns false when secret does not exist", async () => {
    expect(await svc.hasPassword("google-api-key")).toBe(false);
  });

  it("getAllKeys returns all stored keys", async () => {
    await svc.setPassword("anthropic-api-key", "a");
    await svc.setPassword("openai-api-key", "b");
    const keys = await svc.getAllKeys();
    expect(keys).toContain("anthropic-api-key");
    expect(keys).toContain("openai-api-key");
  });
});
Summary: Pass 24 canonical file map
text

packages/
  keychain/
    package.json                     ← NEW
    tsconfig.json                    ← NEW
    src/
      types.ts                       ← NEW
      keychain.ts                    ← NEW (KeychainService)
      index.ts                       ← NEW (exports)
      keychain.test.ts               ← NEW

apps/
  desktop/
    package.json                     ← REWRITE (remove engine deps, add keychain/keytar)
    electron-builder.yml             ← REWRITE (extraResources: dashboard + gateway)
    vite.main.config.ts              ← REWRITE (externalize keytar + node_modules)
    vite.preload.config.ts           ← REWRITE (clean, mirrors main)
    scripts/
      copy-dashboard.sh              ← NEW
    src/
      shared/
        ipcTypes.ts                  ← REWRITE (keychain + gateway-status channels)
      main/
        index.ts                     ← REWRITE (spawn gateway, load dashboard)
        gatewayProcess.ts            ← NEW (GatewayProcessManager)
        keychain.ts                  ← NEW (registerKeychainHandlers + resolveApiKeys)
        ipcHandlers.ts               ← REWRITE (OS-integration only, no engine code)
        config.ts                    ← REWRITE (no secrets on disk)
        tray.ts                      ← REWRITE (gateway-status-aware)
        updater.ts                   ← UPDATE (minor, getWindow pattern)
      preload/
        index.ts                     ← REWRITE (gateway + keychain + config + window + app)

  dashboard/
    vite.config.ts                   ← PATCH (add base: "./")
    src/store/gatewaySlice.ts        ← PATCH (resolveInitialGatewayUrl)

turbo.json                           ← PATCH (desktop depends on dashboard + gateway builds)
What Pass 24 closes
Gap from missingspecelements.md	Closed by
Desktop IPC handler bodies were hollow / engine-in-process	ipcHandlers.ts rewrite; engine moved to gateway child process
No OS keychain implementation	packages/keychain + main/keychain.ts
Embedded gateway never actually spawned	GatewayProcessManager + main/index.ts boot sequence
Desktop had no tray status reflecting real system state	tray.ts rewrite with GatewayStatus integration
Dashboard file:// loading broke asset paths	base: "./" in dashboard vite.config.ts
Gateway URL unknown at dashboard build time	window.__LOCOWORKER_GATEWAY_URL__ runtime injection + resolveInitialGatewayUrl in store
API keys stored on disk	config.ts never writes secrets; resolveApiKeys reads from OS keychain and injects into gateway child env at spawn time
Desktop build had no declared dependency on dashboard/gateway builds	turbo.json pipeline dependency added
