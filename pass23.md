# PASS 23 — Dashboard: Gateway Client, SSE Event Stream, Live Runs UI, Permissions, Kairos + Orchestrator Panels

> **Philosophy:** “Later pass wins.”  
> This pass turns `apps/dashboard` from a frontend shell into a working browser client for the Pass 20 Gateway and Pass 22 autonomy routes.

---

## What Pass 23 delivers

| Area | What gets built |
|---|---|
| Dashboard app | Vite + React SPA |
| Gateway API client | Typed REST client for sessions/runs/permissions/kairos/orchestrator |
| SSE client | Browser `EventSource`-based streaming of `AgentEvent`s |
| State store | Zustand app store for connection, sessions, runs, events, permissions |
| Live run UI | Create session, start run, stream events, render assistant/tool/cost events |
| Permission UI | Approve/deny Gateway permission prompts from dashboard |
| Kairos UI | List/schedule/run-due scheduled tasks |
| Orchestrator UI | Start/list/inspect orchestration runs |
| Components | Event timeline, tool cards, model/cost metrics, prompt composer |
| Tests | SSE URL parsing/client helpers + store reducer-ish flows |

---

# 0) Pass 23 file tree

```txt
apps/
└── dashboard/
    ├── package.json                         ← REWRITE
    ├── tsconfig.json                        ← REWRITE
    ├── tsconfig.node.json                   ← REWRITE
    ├── vite.config.ts                       ← REWRITE
    ├── index.html                           ← REWRITE
    └── src/
        ├── main.tsx                         ← REWRITE
        ├── App.tsx                          ← REWRITE
        ├── styles.css                       ← NEW/REWRITE
        ├── api/
        │   ├── GatewayClient.ts             ← NEW
        │   └── SseClient.ts                 ← NEW
        ├── state/
        │   └── useDashboardStore.ts         ← NEW
        ├── types/
        │   └── api.ts                       ← NEW
        ├── components/
        │   ├── Shell.tsx                    ← NEW
        │   ├── ConnectionPanel.tsx          ← NEW
        │   ├── SessionPanel.tsx             ← NEW
        │   ├── RunComposer.tsx              ← NEW
        │   ├── EventTimeline.tsx            ← NEW
        │   ├── EventCard.tsx                ← NEW
        │   ├── PermissionPromptCard.tsx     ← NEW
        │   ├── MetricsBar.tsx               ← NEW
        │   ├── KairosPanel.tsx              ← NEW
        │   └── OrchestratorPanel.tsx        ← NEW
        └── utils/
            └── format.ts                    ← NEW

tests/
└── unit/
    └── dashboard/
        ├── SseClient.test.ts                ← NEW
        └── GatewayClient.test.ts            ← NEW
```

---

# 1) Dashboard package config

## `apps/dashboard/package.json`

```json
{
  "name": "@locoworker/app-dashboard",
  "version": "0.1.0",
  "private": true,
  "description": "LocoWorker browser dashboard",
  "type": "module",
  "scripts": {
    "dev": "vite --host 127.0.0.1 --port 5173",
    "build": "tsc -p tsconfig.json && vite build",
    "preview": "vite preview --host 127.0.0.1 --port 4173",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "lint": "biome lint src",
    "test": "vitest run"
  },
  "dependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.3.4",
    "typescript": "^5.4.5",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "zustand": "^4.5.4"
  },
  "devDependencies": {
    "@locoworker/tsconfig": "workspace:*",
    "@testing-library/jest-dom": "^6.4.6",
    "@testing-library/react": "^15.0.7",
    "@types/node": "^20.12.0",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "jsdom": "^24.1.0",
    "vitest": "^1.6.0"
  }
}
```

---

## `apps/dashboard/tsconfig.json`

```json
{
  "extends": "@locoworker/tsconfig/base.json",
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["DOM", "DOM.Iterable", "ES2020"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src", "../../tests/unit/dashboard"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## `apps/dashboard/tsconfig.node.json`

```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

---

## `apps/dashboard/vite.config.ts`

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    host: "127.0.0.1",
    port: 5173
  },
  preview: {
    host: "127.0.0.1",
    port: 4173
  },
  test: {
    environment: "jsdom"
  }
});
```

---

## `apps/dashboard/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>LocoWorker Dashboard</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

# 2) Shared browser-side API types

## `apps/dashboard/src/types/api.ts`

```ts
// apps/dashboard/src/types/api.ts

export type AgentEventKind =
  | "session_start"
  | "session_restore"
  | "session_checkpoint"
  | "session_end"
  | "context_injected"
  | "turn_start"
  | "model_request"
  | "model_response_start"
  | "model_response_delta"
  | "model_response_end"
  | "tool_call"
  | "tool_result"
  | "tool_error"
  | "permission_prompt"
  | "permission_granted"
  | "permission_denied"
  | "budget_warning"
  | "budget_exceeded"
  | "cost_cap_exceeded"
  | "compaction_start"
  | "compaction_end"
  | "turn_complete"
  | "loop_complete"
  | "session_error"
  | "fatal_error";

export interface BaseAgentEvent {
  kind: AgentEventKind;
  sessionId: string;
  turnIndex: number;
  ts: number;
  [key: string]: unknown;
}

export type AgentEvent = BaseAgentEvent;

export interface GatewaySession {
  id: string;
  workingDir: string;
  modelConfig: {
    provider: string;
    model: string;
    temperature?: number;
    maxOutputTokens?: number;
    baseUrl?: string;
  };
  budget: {
    maxInputTokens: number;
    softLimitTokens: number;
    hardLimitTokens: number;
    maxCostUsd?: number;
    maxTurns?: number;
    compactionMode: "auto" | "full" | "none";
  };
  createdAt: number;
  lastCheckpointAt: number;
  turnIndex: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCostUsd: number;
}

export interface GatewayRun {
  id: string;
  sessionId: string;
  status: "running" | "completed" | "failed" | "aborted";
  message: string;
  startedAt: number;
  finishedAt?: number;
  error?: string;
}

export interface PendingPermissionPrompt {
  promptId: string;
  sessionId: string;
  runId: string;
  toolName: string;
  tier: number;
  description: string;
  examples: string[];
  createdAt: number;
  expiresAt: number;
}

export interface KairosTask {
  id: string;
  name: string;
  kind: string;
  status: string;
  payload: Record<string, unknown>;
  nextRunAt?: number;
  lastRunAt?: number;
  lastError?: string;
}

export interface OrchestrationRun {
  id: string;
  status: string;
  request: {
    goal: string;
    workingDir?: string;
    files?: string[];
  };
  createdAt: number;
  updatedAt: number;
  startedAt?: number;
  finishedAt?: number;
  units?: Array<{
    id: string;
    title: string;
    status: string;
    error?: string;
  }>;
  results?: Array<{
    unitId: string;
    title: string;
    status: string;
    output: string;
    error?: string;
  }>;
  finalSummary?: string;
  error?: string;
}

export interface ApiEnvelope<T> {
  ok: boolean;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
  [key: string]: unknown;
}
```

---

# 3) Gateway API client

## `apps/dashboard/src/api/GatewayClient.ts`

```ts
// apps/dashboard/src/api/GatewayClient.ts

import type {
  GatewayRun,
  GatewaySession,
  KairosTask,
  OrchestrationRun,
  PendingPermissionPrompt
} from "../types/api";

export class GatewayClient {
  constructor(private readonly baseUrl: string) {}

  get url(): string {
    return this.baseUrl.replace(/\/$/, "");
  }

  async health(): Promise<unknown> {
    return this.request("GET", "/v1/health");
  }

  async createSession(input: {
    workingDir?: string;
    modelConfig?: Record<string, unknown>;
    budget?: Record<string, unknown>;
  } = {}): Promise<GatewaySession> {
    const res = await this.request<{ session: GatewaySession }>(
      "POST",
      "/v1/sessions",
      input
    );

    return res.session;
  }

  async listSessions(): Promise<Array<{ id: string; createdAt: number; model: string }>> {
    const res = await this.request<{
      sessions: Array<{ id: string; createdAt: number; model: string }>;
    }>("GET", "/v1/sessions");

    return res.sessions;
  }

  async getSession(sessionId: string): Promise<GatewaySession> {
    const res = await this.request<{ session: GatewaySession }>(
      "GET",
      `/v1/sessions/${encodeURIComponent(sessionId)}`
    );

    return res.session;
  }

  async startRun(input: {
    sessionId: string;
    message: string;
  }): Promise<{ run: GatewayRun; streamUrl: string }> {
    return this.request(
      "POST",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}/runs`,
      {
        message: input.message,
        stream: true
      }
    );
  }

  async listRuns(sessionId: string): Promise<GatewayRun[]> {
    const res = await this.request<{ runs: GatewayRun[] }>(
      "GET",
      `/v1/sessions/${encodeURIComponent(sessionId)}/runs`
    );

    return res.runs;
  }

  async abortRun(input: {
    sessionId: string;
    runId: string;
  }): Promise<void> {
    await this.request(
      "POST",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}` +
        `/runs/${encodeURIComponent(input.runId)}/abort`,
      { reason: "dashboard_abort" }
    );
  }

  async listPermissions(input: {
    sessionId: string;
    runId: string;
  }): Promise<PendingPermissionPrompt[]> {
    const res = await this.request<{ permissions: PendingPermissionPrompt[] }>(
      "GET",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}` +
        `/runs/${encodeURIComponent(input.runId)}/permissions`
    );

    return res.permissions;
  }

  async resolvePermission(input: {
    sessionId: string;
    runId: string;
    promptId: string;
    granted: boolean;
    permanent: boolean;
  }): Promise<void> {
    await this.request(
      "POST",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}` +
        `/runs/${encodeURIComponent(input.runId)}` +
        `/permissions/${encodeURIComponent(input.promptId)}`,
      {
        granted: input.granted,
        permanent: input.permanent
      }
    );
  }

  async listKairosTasks(): Promise<KairosTask[]> {
    const res = await this.request<{ tasks: KairosTask[] }>(
      "GET",
      "/v1/kairos/tasks"
    );

    return res.tasks;
  }

  async scheduleKairosTask(input: {
    name: string;
    kind?: string;
    runAt?: number;
    cron?: string;
    payload?: Record<string, unknown>;
  }): Promise<unknown> {
    return this.request("POST", "/v1/kairos/tasks", {
      name: input.name,
      kind: input.kind,
      runAt: input.runAt,
      cron: input.cron,
      payload: {
        ...(input.payload ?? {}),
        ...(input.kind ? { kind: input.kind } : {})
      }
    });
  }

  async runKairosDue(): Promise<unknown> {
    return this.request("POST", "/v1/kairos/run-due", {});
  }

  async startOrchestration(input: {
    goal: string;
    files?: string[];
    maxConcurrency?: number;
    subtasks?: Array<{
      title: string;
      prompt: string;
      files?: string[];
      dependsOn?: string[];
    }>;
  }): Promise<OrchestrationRun> {
    const res = await this.request<{ run: OrchestrationRun }>(
      "POST",
      "/v1/orchestrator/runs",
      input
    );

    return res.run;
  }

  async listOrchestrations(): Promise<OrchestrationRun[]> {
    const res = await this.request<{ runs: OrchestrationRun[] }>(
      "GET",
      "/v1/orchestrator/runs"
    );

    return res.runs;
  }

  async getOrchestration(id: string): Promise<OrchestrationRun> {
    const res = await this.request<{ run: OrchestrationRun }>(
      "GET",
      `/v1/orchestrator/runs/${encodeURIComponent(id)}`
    );

    return res.run;
  }

  private async request<T>(
    method: string,
    path: string,
    body?: unknown
  ): Promise<T> {
    const res = await fetch(`${this.url}${path}`, {
      method,
      headers: {
        accept: "application/json",
        ...(body === undefined ? {} : { "content-type": "application/json" })
      },
      body: body === undefined ? undefined : JSON.stringify(body)
    });

    const text = await res.text();
    const json = text ? JSON.parse(text) : {};

    if (!res.ok || json.ok === false) {
      throw new Error(
        json?.error?.message ?? `Gateway request failed: HTTP ${res.status}`
      );
    }

    return json as T;
  }
}
```

---

# 4) SSE client

## `apps/dashboard/src/api/SseClient.ts`

```ts
// apps/dashboard/src/api/SseClient.ts

import type { AgentEvent } from "../types/api";

export function buildRunStreamUrl(input: {
  gatewayUrl: string;
  sessionId: string;
  runId: string;
}): string {
  const base = input.gatewayUrl.replace(/\/$/, "");

  return (
    `${base}/v1/sessions/${encodeURIComponent(input.sessionId)}` +
    `/runs/${encodeURIComponent(input.runId)}/stream`
  );
}

export interface RunSseSubscription {
  close: () => void;
}

export function subscribeToRunEvents(input: {
  gatewayUrl: string;
  sessionId: string;
  runId: string;
  onEvent: (event: AgentEvent) => void;
  onError?: (error: Error) => void;
  onOpen?: () => void;
}): RunSseSubscription {
  const url = buildRunStreamUrl(input);

  const source = new EventSource(url);

  source.onopen = () => {
    input.onOpen?.();
  };

  source.onerror = () => {
    input.onError?.(new Error("SSE connection error"));
  };

  const knownEvents = [
    "session_start",
    "session_restore",
    "session_checkpoint",
    "session_end",
    "context_injected",
    "turn_start",
    "model_request",
    "model_response_start",
    "model_response_delta",
    "model_response_end",
    "tool_call",
    "tool_result",
    "tool_error",
    "permission_prompt",
    "permission_granted",
    "permission_denied",
    "budget_warning",
    "budget_exceeded",
    "cost_cap_exceeded",
    "compaction_start",
    "compaction_end",
    "turn_complete",
    "loop_complete",
    "session_error",
    "fatal_error"
  ];

  for (const kind of knownEvents) {
    source.addEventListener(kind, (message) => {
      try {
        const event = JSON.parse((message as MessageEvent).data) as AgentEvent;
        input.onEvent(event);

        if (event.kind === "session_end" || event.kind === "fatal_error") {
          source.close();
        }
      } catch (err) {
        input.onError?.(
          err instanceof Error ? err : new Error(String(err))
        );
      }
    });
  }

  return {
    close: () => source.close()
  };
}
```

---

# 5) Dashboard store

## `apps/dashboard/src/state/useDashboardStore.ts`

```ts
// apps/dashboard/src/state/useDashboardStore.ts

import { create } from "zustand";
import { GatewayClient } from "../api/GatewayClient";
import { subscribeToRunEvents, type RunSseSubscription } from "../api/SseClient";
import type {
  AgentEvent,
  GatewayRun,
  GatewaySession,
  KairosTask,
  OrchestrationRun,
  PendingPermissionPrompt
} from "../types/api";

interface DashboardState {
  gatewayUrl: string;
  connected: boolean;
  loading: boolean;
  error?: string;

  client: GatewayClient;

  sessions: GatewaySession[];
  selectedSession?: GatewaySession;
  runs: GatewayRun[];
  activeRun?: GatewayRun;
  events: AgentEvent[];
  pendingPermissions: PendingPermissionPrompt[];

  kairosTasks: KairosTask[];
  orchestrations: OrchestrationRun[];
  selectedOrchestration?: OrchestrationRun;

  sse?: RunSseSubscription;

  setGatewayUrl: (url: string) => void;
  checkHealth: () => Promise<void>;

  createSession: (input?: {
    workingDir?: string;
    provider?: string;
    model?: string;
    baseUrl?: string;
  }) => Promise<void>;

  refreshSessions: () => Promise<void>;
  selectSession: (sessionId: string) => Promise<void>;

  startRun: (message: string) => Promise<void>;
  abortActiveRun: () => Promise<void>;
  appendEvent: (event: AgentEvent) => void;
  clearEvents: () => void;

  resolvePermission: (promptId: string, granted: boolean, permanent: boolean) => Promise<void>;

  refreshKairos: () => Promise<void>;
  scheduleNoopTask: () => Promise<void>;
  runKairosDue: () => Promise<void>;

  refreshOrchestrations: () => Promise<void>;
  startOrchestration: (goal: string) => Promise<void>;
  selectOrchestration: (id: string) => Promise<void>;
}

const defaultGatewayUrl =
  import.meta.env.VITE_LOCOWORKER_GATEWAY_URL ?? "http://127.0.0.1:8787";

export const useDashboardStore = create<DashboardState>((set, get) => ({
  gatewayUrl: defaultGatewayUrl,
  connected: false,
  loading: false,
  client: new GatewayClient(defaultGatewayUrl),

  sessions: [],
  runs: [],
  events: [],
  pendingPermissions: [],
  kairosTasks: [],
  orchestrations: [],

  setGatewayUrl: (url) => {
    get().sse?.close();

    set({
      gatewayUrl: url,
      client: new GatewayClient(url),
      connected: false,
      selectedSession: undefined,
      activeRun: undefined,
      events: [],
      pendingPermissions: []
    });
  },

  checkHealth: async () => {
    set({ loading: true, error: undefined });

    try {
      await get().client.health();
      set({ connected: true, loading: false });
    } catch (err) {
      set({
        connected: false,
        loading: false,
        error: err instanceof Error ? err.message : String(err)
      });
    }
  },

  createSession: async (input = {}) => {
    set({ loading: true, error: undefined });

    try {
      const session = await get().client.createSession({
        workingDir: input.workingDir || undefined,
        modelConfig: {
          ...(input.provider ? { provider: input.provider } : {}),
          ...(input.model ? { model: input.model } : {}),
          ...(input.baseUrl ? { baseUrl: input.baseUrl } : {})
        }
      });

      set((state) => ({
        loading: false,
        selectedSession: session,
        sessions: [session, ...state.sessions.filter((s) => s.id !== session.id)]
      }));
    } catch (err) {
      set({
        loading: false,
        error: err instanceof Error ? err.message : String(err)
      });
    }
  },

  refreshSessions: async () => {
    set({ loading: true, error: undefined });

    try {
      const summaries = await get().client.listSessions();

      const sessions = await Promise.all(
        summaries.map((s) => get().client.getSession(s.id).catch(() => undefined))
      );

      set({
        loading: false,
        sessions: sessions.filter(Boolean) as GatewaySession[]
      });
    } catch (err) {
      set({
        loading: false,
        error: err instanceof Error ? err.message : String(err)
      });
    }
  },

  selectSession: async (sessionId) => {
    set({ loading: true, error: undefined });

    try {
      const session = await get().client.getSession(sessionId);
      const runs = await get().client.listRuns(sessionId);

      set({
        selectedSession: session,
        runs,
        loading: false,
        activeRun: runs.find((r) => r.status === "running") ?? undefined,
        events: []
      });
    } catch (err) {
      set({
        loading: false,
        error: err instanceof Error ? err.message : String(err)
      });
    }
  },

  startRun: async (message) => {
    const session = get().selectedSession;
    if (!session) {
      set({ error: "Create or select a session first." });
      return;
    }

    get().sse?.close();

    set({
      loading: true,
      error: undefined,
      events: [],
      pendingPermissions: []
    });

    try {
      const { run } = await get().client.startRun({
        sessionId: session.id,
        message
      });

      const sse = subscribeToRunEvents({
        gatewayUrl: get().gatewayUrl,
        sessionId: session.id,
        runId: run.id,
        onEvent: (event) => get().appendEvent(event),
        onError: (error) => {
          // Keep the run UI alive; Gateway may close SSE at terminal events.
          set({ error: error.message });
        }
      });

      set((state) => ({
        loading: false,
        activeRun: run,
        runs: [run, ...state.runs.filter((r) => r.id !== run.id)],
        sse
      }));
    } catch (err) {
      set({
        loading: false,
        error: err instanceof Error ? err.message : String(err)
      });
    }
  },

  abortActiveRun: async () => {
    const session = get().selectedSession;
    const run = get().activeRun;

    if (!session || !run) return;

    await get().client.abortRun({
      sessionId: session.id,
      runId: run.id
    });

    get().sse?.close();

    set({
      activeRun: {
        ...run,
        status: "aborted",
        finishedAt: Date.now()
      },
      sse: undefined
    });
  },

  appendEvent: (event) => {
    set((state) => {
      const nextEvents = [...state.events, event];

      const pendingPermissions =
        event.kind === "permission_prompt"
          ? [
              ...state.pendingPermissions,
              {
                promptId: String(event.promptId),
                sessionId: event.sessionId,
                runId: state.activeRun?.id ?? "",
                toolName: String(event.toolName),
                tier: Number(event.requiredTier),
                description: String(event.description),
                examples: [],
                createdAt: event.ts,
                expiresAt: event.ts + 5 * 60_000
              }
            ]
          : event.kind === "permission_granted" || event.kind === "permission_denied"
            ? state.pendingPermissions.filter(
                (p) => p.promptId !== String(event.promptId)
              )
            : state.pendingPermissions;

      const activeRun =
        event.kind === "session_end" && state.activeRun
          ? {
              ...state.activeRun,
              status:
                event.reason === "fatal_error"
                  ? "failed"
                  : event.reason === "user_abort"
                    ? "aborted"
                    : "completed",
              finishedAt: event.ts
            }
          : state.activeRun;

      return {
        events: nextEvents,
        pendingPermissions,
        activeRun
      };
    });
  },

  clearEvents: () => set({ events: [], pendingPermissions: [] }),

  resolvePermission: async (promptId, granted, permanent) => {
    const session = get().selectedSession;
    const run = get().activeRun;

    if (!session || !run) return;

    await get().client.resolvePermission({
      sessionId: session.id,
      runId: run.id,
      promptId,
      granted,
      permanent
    });

    set((state) => ({
      pendingPermissions: state.pendingPermissions.filter(
        (p) => p.promptId !== promptId
      )
    }));
  },

  refreshKairos: async () => {
    try {
      const tasks = await get().client.listKairosTasks();
      set({ kairosTasks: tasks });
    } catch (err) {
      set({ error: err instanceof Error ? err.message : String(err) });
    }
  },

  scheduleNoopTask: async () => {
    await get().client.scheduleKairosTask({
      name: `Dashboard no-op ${new Date().toLocaleTimeString()}`,
      kind: "noop",
      runAt: Date.now() + 1000,
      payload: { kind: "noop" }
    });

    await get().refreshKairos();
  },

  runKairosDue: async () => {
    await get().client.runKairosDue();
    await get().refreshKairos();
  },

  refreshOrchestrations: async () => {
    try {
      const runs = await get().client.listOrchestrations();
      set({ orchestrations: runs });
    } catch (err) {
      set({ error: err instanceof Error ? err.message : String(err) });
    }
  },

  startOrchestration: async (goal) => {
    try {
      const run = await get().client.startOrchestration({
        goal,
        maxConcurrency: 2
      });

      set((state) => ({
        orchestrations: [run, ...state.orchestrations.filter((r) => r.id !== run.id)],
        selectedOrchestration: run
      }));
    } catch (err) {
      set({ error: err instanceof Error ? err.message : String(err) });
    }
  },

  selectOrchestration: async (id) => {
    try {
      const run = await get().client.getOrchestration(id);
      set({ selectedOrchestration: run });
    } catch (err) {
      set({ error: err instanceof Error ? err.message : String(err) });
    }
  }
}));
```

---

# 6) Utilities

## `apps/dashboard/src/utils/format.ts`

```ts
// apps/dashboard/src/utils/format.ts

export function formatMoney(value: unknown): string {
  const n = Number(value ?? 0);
  if (!Number.isFinite(n)) return "$0.000000";
  return `$${n.toFixed(6)}`;
}

export function formatTime(ts: unknown): string {
  const n = Number(ts);
  if (!Number.isFinite(n)) return "";
  return new Date(n).toLocaleTimeString();
}

export function formatDateTime(ts: unknown): string {
  const n = Number(ts);
  if (!Number.isFinite(n)) return "";
  return new Date(n).toLocaleString();
}

export function asText(value: unknown): string {
  if (value == null) return "";
  if (typeof value === "string") return value;
  if (typeof value === "number" || typeof value === "boolean") return String(value);

  try {
    return JSON.stringify(value, null, 2);
  } catch {
    return String(value);
  }
}

export function shortId(id: string): string {
  if (id.length <= 12) return id;
  return `${id.slice(0, 8)}…${id.slice(-4)}`;
}
```

---

# 7) Components

## `apps/dashboard/src/components/Shell.tsx`

```tsx
// apps/dashboard/src/components/Shell.tsx

import type { ReactNode } from "react";
import { useDashboardStore } from "../state/useDashboardStore";

export function Shell({ children }: { children: ReactNode }) {
  const error = useDashboardStore((s) => s.error);
  const connected = useDashboardStore((s) => s.connected);
  const gatewayUrl = useDashboardStore((s) => s.gatewayUrl);

  return (
    <div className="app-shell">
      <header className="topbar">
        <div>
          <h1>LocoWorker</h1>
          <p>Privacy-first agentic developer workspace dashboard</p>
        </div>

        <div className={`status-pill ${connected ? "ok" : "bad"}`}>
          {connected ? "Connected" : "Disconnected"}
          <span>{gatewayUrl}</span>
        </div>
      </header>

      {error ? <div className="error-banner">{error}</div> : null}

      {children}
    </div>
  );
}
```

---

## `apps/dashboard/src/components/ConnectionPanel.tsx`

```tsx
// apps/dashboard/src/components/ConnectionPanel.tsx

import { useState } from "react";
import { useDashboardStore } from "../state/useDashboardStore";

export function ConnectionPanel() {
  const gatewayUrl = useDashboardStore((s) => s.gatewayUrl);
  const loading = useDashboardStore((s) => s.loading);
  const setGatewayUrl = useDashboardStore((s) => s.setGatewayUrl);
  const checkHealth = useDashboardStore((s) => s.checkHealth);
  const refreshSessions = useDashboardStore((s) => s.refreshSessions);
  const refreshKairos = useDashboardStore((s) => s.refreshKairos);
  const refreshOrchestrations = useDashboardStore((s) => s.refreshOrchestrations);

  const [value, setValue] = useState(gatewayUrl);

  async function connect() {
    setGatewayUrl(value);
    await checkHealth();
    await refreshSessions();
    await refreshKairos();
    await refreshOrchestrations();
  }

  return (
    <section className="panel">
      <div className="panel-title">Gateway</div>

      <div className="inline-form">
        <input
          value={value}
          onChange={(e) => setValue(e.target.value)}
          placeholder="http://127.0.0.1:8787"
        />
        <button onClick={connect} disabled={loading}>
          {loading ? "Connecting…" : "Connect"}
        </button>
      </div>
    </section>
  );
}
```

---

## `apps/dashboard/src/components/SessionPanel.tsx`

```tsx
// apps/dashboard/src/components/SessionPanel.tsx

import { useState } from "react";
import { useDashboardStore } from "../state/useDashboardStore";
import { formatDateTime, shortId } from "../utils/format";

export function SessionPanel() {
  const sessions = useDashboardStore((s) => s.sessions);
  const selectedSession = useDashboardStore((s) => s.selectedSession);
  const createSession = useDashboardStore((s) => s.createSession);
  const refreshSessions = useDashboardStore((s) => s.refreshSessions);
  const selectSession = useDashboardStore((s) => s.selectSession);

  const [workingDir, setWorkingDir] = useState("");
  const [provider, setProvider] = useState("anthropic");
  const [model, setModel] = useState("claude-3-5-sonnet-latest");
  const [baseUrl, setBaseUrl] = useState("");

  return (
    <section className="panel">
      <div className="panel-title">Sessions</div>

      <div className="stack">
        <input
          value={workingDir}
          onChange={(e) => setWorkingDir(e.target.value)}
          placeholder="Workspace path (optional)"
        />

        <div className="two-col">
          <input
            value={provider}
            onChange={(e) => setProvider(e.target.value)}
            placeholder="provider"
          />
          <input
            value={model}
            onChange={(e) => setModel(e.target.value)}
            placeholder="model"
          />
        </div>

        <input
          value={baseUrl}
          onChange={(e) => setBaseUrl(e.target.value)}
          placeholder="Base URL for local/custom providers (optional)"
        />

        <div className="button-row">
          <button
            onClick={() =>
              createSession({
                workingDir,
                provider,
                model,
                baseUrl
              })
            }
          >
            Create session
          </button>
          <button className="secondary" onClick={refreshSessions}>
            Refresh
          </button>
        </div>
      </div>

      {selectedSession ? (
        <div className="selected-box">
          <strong>Selected</strong>
          <div>{shortId(selectedSession.id)}</div>
          <small>{selectedSession.workingDir}</small>
          <small>{selectedSession.modelConfig.model}</small>
        </div>
      ) : null}

      <div className="list">
        {sessions.map((session) => (
          <button
            key={session.id}
            className={`list-item ${
              selectedSession?.id === session.id ? "selected" : ""
            }`}
            onClick={() => selectSession(session.id)}
          >
            <span>{shortId(session.id)}</span>
            <small>{session.modelConfig?.model ?? "model"}</small>
            <small>{formatDateTime(session.createdAt)}</small>
          </button>
        ))}
      </div>
    </section>
  );
}
```

---

## `apps/dashboard/src/components/RunComposer.tsx`

```tsx
// apps/dashboard/src/components/RunComposer.tsx

import { useState } from "react";
import { useDashboardStore } from "../state/useDashboardStore";

export function RunComposer() {
  const selectedSession = useDashboardStore((s) => s.selectedSession);
  const activeRun = useDashboardStore((s) => s.activeRun);
  const startRun = useDashboardStore((s) => s.startRun);
  const abortActiveRun = useDashboardStore((s) => s.abortActiveRun);
  const clearEvents = useDashboardStore((s) => s.clearEvents);

  const [message, setMessage] = useState("");

  async function submit() {
    if (!message.trim()) return;
    await startRun(message.trim());
  }

  return (
    <section className="panel composer">
      <div className="panel-title">Run</div>

      <textarea
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        placeholder={
          selectedSession
            ? "Ask LocoWorker to inspect, explain, edit, test, or plan…"
            : "Create/select a session first…"
        }
        disabled={!selectedSession}
        rows={6}
      />

      <div className="button-row">
        <button disabled={!selectedSession || !message.trim()} onClick={submit}>
          Start run
        </button>

        <button
          className="secondary"
          disabled={!activeRun || activeRun.status !== "running"}
          onClick={abortActiveRun}
        >
          Abort
        </button>

        <button className="secondary" onClick={clearEvents}>
          Clear events
        </button>
      </div>

      {activeRun ? (
        <div className={`run-status ${activeRun.status}`}>
          Active run: {activeRun.id} · {activeRun.status}
        </div>
      ) : null}
    </section>
  );
}
```

---

## `apps/dashboard/src/components/MetricsBar.tsx`

```tsx
// apps/dashboard/src/components/MetricsBar.tsx

import { useDashboardStore } from "../state/useDashboardStore";
import { formatMoney } from "../utils/format";

export function MetricsBar() {
  const events = useDashboardStore((s) => s.events);

  const modelRequests = events.filter((e) => e.kind === "model_request").length;
  const toolCalls = events.filter((e) => e.kind === "tool_call").length;
  const toolErrors = events.filter((e) => e.kind === "tool_error").length;

  const totalCost = events
    .filter((e) => e.kind === "model_response_end" || e.kind === "turn_complete")
    .reduce((sum, e) => sum + Number(e.costUsd ?? 0), 0);

  const inputTokens = events
    .filter((e) => e.kind === "model_response_end")
    .reduce((sum, e) => sum + Number(e.inputTokens ?? 0), 0);

  const outputTokens = events
    .filter((e) => e.kind === "model_response_end")
    .reduce((sum, e) => sum + Number(e.outputTokens ?? 0), 0);

  return (
    <section className="metrics">
      <div>
        <strong>{modelRequests}</strong>
        <span>model calls</span>
      </div>
      <div>
        <strong>{toolCalls}</strong>
        <span>tool calls</span>
      </div>
      <div>
        <strong>{toolErrors}</strong>
        <span>tool errors</span>
      </div>
      <div>
        <strong>{inputTokens + outputTokens}</strong>
        <span>tokens</span>
      </div>
      <div>
        <strong>{formatMoney(totalCost)}</strong>
        <span>cost</span>
      </div>
    </section>
  );
}
```

---

## `apps/dashboard/src/components/PermissionPromptCard.tsx`

```tsx
// apps/dashboard/src/components/PermissionPromptCard.tsx

import { useDashboardStore } from "../state/useDashboardStore";
import { shortId } from "../utils/format";

export function PermissionPromptCard() {
  const pending = useDashboardStore((s) => s.pendingPermissions);
  const resolvePermission = useDashboardStore((s) => s.resolvePermission);

  if (pending.length === 0) return null;

  return (
    <section className="permission-panel">
      <h3>Permission required</h3>

      {pending.map((prompt) => (
        <div key={prompt.promptId} className="permission-card">
          <div>
            <strong>{prompt.toolName}</strong>
            <span>Tier {prompt.tier}</span>
          </div>

          <p>{prompt.description}</p>
          <small>{shortId(prompt.promptId)}</small>

          <div className="button-row">
            <button
              onClick={() =>
                resolvePermission(prompt.promptId, true, false)
              }
            >
              Allow once
            </button>

            <button
              onClick={() =>
                resolvePermission(prompt.promptId, true, true)
              }
            >
              Always allow
            </button>

            <button
              className="danger"
              onClick={() =>
                resolvePermission(prompt.promptId, false, false)
              }
            >
              Deny
            </button>
          </div>
        </div>
      ))}
    </section>
  );
}
```

---

## `apps/dashboard/src/components/EventCard.tsx`

```tsx
// apps/dashboard/src/components/EventCard.tsx

import type { AgentEvent } from "../types/api";
import { asText, formatMoney, formatTime } from "../utils/format";

export function EventCard({ event }: { event: AgentEvent }) {
  const tone = getTone(event.kind);

  return (
    <article className={`event-card ${tone}`}>
      <header>
        <span className="event-kind">{event.kind}</span>
        <time>{formatTime(event.ts)}</time>
      </header>

      <EventBody event={event} />
    </article>
  );
}

function EventBody({ event }: { event: AgentEvent }) {
  switch (event.kind) {
    case "turn_start":
      return <p>{asText(event.userMessage)}</p>;

    case "model_request":
      return (
        <p>
          {asText(event.model)} · {asText(event.messageCount)} messages · ~
          {asText(event.estimatedInputTokens)} input tokens
        </p>
      );

    case "model_response_delta":
      return <p className="delta">{asText(event.delta)}</p>;

    case "model_response_end":
      return (
        <p>
          stop={asText(event.stopReason)} · in={asText(event.inputTokens)} · out=
          {asText(event.outputTokens)} · {formatMoney(event.costUsd)}
        </p>
      );

    case "tool_call":
      return (
        <>
          <p>
            <strong>{asText(event.toolName)}</strong> · {asText(event.toolCallId)}
          </p>
          <pre>{asText(event.input)}</pre>
        </>
      );

    case "tool_result":
      return (
        <p>
          <strong>{asText(event.toolName)}</strong> · {asText(event.durationMs)}ms
          <br />
          {asText(event.outputPreview)}
        </p>
      );

    case "tool_error":
      return (
        <p>
          <strong>{asText(event.toolName)}</strong>: {asText(event.error)}
        </p>
      );

    case "permission_prompt":
      return (
        <p>
          {asText(event.toolName)} requires tier {asText(event.requiredTier)}:{" "}
          {asText(event.description)}
        </p>
      );

    case "turn_complete":
      return (
        <>
          <p>{asText(event.assistantText)}</p>
          <small>
            tools={asText(event.toolCallsExecuted)} · cost={formatMoney(event.costUsd)}
          </small>
        </>
      );

    case "loop_complete":
      return <p>{asText(event.finalResponse)}</p>;

    case "session_end":
      return (
        <p>
          reason={asText(event.reason)} · turns={asText(event.totalTurns)} · cost=
          {formatMoney(event.totalCostUsd)}
        </p>
      );

    case "fatal_error":
    case "session_error":
      return <p>{asText(event.error)}</p>;

    default:
      return <pre>{asText(event)}</pre>;
  }
}

function getTone(kind: string): string {
  if (kind.includes("error") || kind === "fatal_error") return "danger";
  if (kind.includes("permission")) return "warning";
  if (kind.includes("tool")) return "tool";
  if (kind.includes("model")) return "model";
  if (kind.includes("session")) return "session";
  return "neutral";
}
```

---

## `apps/dashboard/src/components/EventTimeline.tsx`

```tsx
// apps/dashboard/src/components/EventTimeline.tsx

import { useEffect, useRef } from "react";
import { useDashboardStore } from "../state/useDashboardStore";
import { EventCard } from "./EventCard";

export function EventTimeline() {
  const events = useDashboardStore((s) => s.events);
  const ref = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    ref.current?.scrollTo({
      top: ref.current.scrollHeight,
      behavior: "smooth"
    });
  }, [events.length]);

  return (
    <section className="panel timeline-panel">
      <div className="panel-title">Event stream</div>

      <div className="timeline" ref={ref}>
        {events.length === 0 ? (
          <div className="empty">No events yet. Start a run to stream events.</div>
        ) : (
          events.map((event, idx) => (
            <EventCard
              key={`${event.kind}-${event.sessionId}-${event.turnIndex}-${event.ts}-${idx}`}
              event={event}
            />
          ))
        )}
      </div>
    </section>
  );
}
```

---

## `apps/dashboard/src/components/KairosPanel.tsx`

```tsx
// apps/dashboard/src/components/KairosPanel.tsx

import { useDashboardStore } from "../state/useDashboardStore";
import { formatDateTime, shortId } from "../utils/format";

export function KairosPanel() {
  const tasks = useDashboardStore((s) => s.kairosTasks);
  const refreshKairos = useDashboardStore((s) => s.refreshKairos);
  const scheduleNoopTask = useDashboardStore((s) => s.scheduleNoopTask);
  const runKairosDue = useDashboardStore((s) => s.runKairosDue);

  return (
    <section className="panel">
      <div className="panel-title">Kairos</div>

      <div className="button-row">
        <button onClick={refreshKairos}>Refresh</button>
        <button className="secondary" onClick={scheduleNoopTask}>
          Schedule no-op
        </button>
        <button className="secondary" onClick={runKairosDue}>
          Run due
        </button>
      </div>

      <div className="list">
        {tasks.length === 0 ? (
          <div className="empty">No Kairos tasks.</div>
        ) : (
          tasks.map((task) => (
            <div key={task.id} className="list-card">
              <strong>{task.name}</strong>
              <small>{shortId(task.id)} · {task.kind} · {task.status}</small>
              {task.nextRunAt ? <small>next: {formatDateTime(task.nextRunAt)}</small> : null}
              {task.lastError ? <small className="danger-text">{task.lastError}</small> : null}
            </div>
          ))
        )}
      </div>
    </section>
  );
}
```

---

## `apps/dashboard/src/components/OrchestratorPanel.tsx`

```tsx
// apps/dashboard/src/components/OrchestratorPanel.tsx

import { useState } from "react";
import { useDashboardStore } from "../state/useDashboardStore";
import { formatDateTime, shortId } from "../utils/format";

export function OrchestratorPanel() {
  const orchestrations = useDashboardStore((s) => s.orchestrations);
  const selected = useDashboardStore((s) => s.selectedOrchestration);
  const refreshOrchestrations = useDashboardStore((s) => s.refreshOrchestrations);
  const startOrchestration = useDashboardStore((s) => s.startOrchestration);
  const selectOrchestration = useDashboardStore((s) => s.selectOrchestration);

  const [goal, setGoal] = useState(
    "Inspect the project and produce a short improvement plan."
  );

  return (
    <section className="panel">
      <div className="panel-title">Orchestrator</div>

      <textarea
        value={goal}
        onChange={(e) => setGoal(e.target.value)}
        rows={4}
        placeholder="Goal for a multi-agent orchestration run…"
      />

      <div className="button-row">
        <button disabled={!goal.trim()} onClick={() => startOrchestration(goal.trim())}>
          Start orchestration
        </button>
        <button className="secondary" onClick={refreshOrchestrations}>
          Refresh
        </button>
      </div>

      <div className="list">
        {orchestrations.length === 0 ? (
          <div className="empty">No orchestrations.</div>
        ) : (
          orchestrations.map((run) => (
            <button
              key={run.id}
              className={`list-item ${selected?.id === run.id ? "selected" : ""}`}
              onClick={() => selectOrchestration(run.id)}
            >
              <span>{shortId(run.id)}</span>
              <small>{run.status}</small>
              <small>{formatDateTime(run.createdAt)}</small>
            </button>
          ))
        )}
      </div>

      {selected ? (
        <div className="selected-box">
          <strong>{selected.request.goal}</strong>
          <small>{selected.status}</small>

          {selected.units?.length ? (
            <>
              <h4>Units</h4>
              {selected.units.map((unit) => (
                <div key={unit.id} className="mini-row">
                  <span>{unit.title}</span>
                  <small>{unit.status}</small>
                </div>
              ))}
            </>
          ) : null}

          {selected.finalSummary ? (
            <>
              <h4>Summary</h4>
              <pre>{selected.finalSummary}</pre>
            </>
          ) : null}

          {selected.error ? <p className="danger-text">{selected.error}</p> : null}
        </div>
      ) : null}
    </section>
  );
}
```

---

# 8) App root

## `apps/dashboard/src/App.tsx`

```tsx
// apps/dashboard/src/App.tsx

import { Shell } from "./components/Shell";
import { ConnectionPanel } from "./components/ConnectionPanel";
import { SessionPanel } from "./components/SessionPanel";
import { RunComposer } from "./components/RunComposer";
import { MetricsBar } from "./components/MetricsBar";
import { PermissionPromptCard } from "./components/PermissionPromptCard";
import { EventTimeline } from "./components/EventTimeline";
import { KairosPanel } from "./components/KairosPanel";
import { OrchestratorPanel } from "./components/OrchestratorPanel";

export function App() {
  return (
    <Shell>
      <main className="dashboard-grid">
        <aside className="left-column">
          <ConnectionPanel />
          <SessionPanel />
          <KairosPanel />
          <OrchestratorPanel />
        </aside>

        <section className="main-column">
          <RunComposer />
          <MetricsBar />
          <PermissionPromptCard />
          <EventTimeline />
        </section>
      </main>
    </Shell>
  );
}
```

---

## `apps/dashboard/src/main.tsx`

```tsx
// apps/dashboard/src/main.tsx

import React from "react";
import ReactDOM from "react-dom/client";
import { App } from "./App";
import "./styles.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

# 9) Styles

## `apps/dashboard/src/styles.css`

```css
:root {
  color-scheme: dark;
  --bg: #080b12;
  --panel: #101624;
  --panel-2: #151d2f;
  --border: #273247;
  --text: #e7edf7;
  --muted: #8d99ae;
  --accent: #69d2ff;
  --green: #55d68b;
  --yellow: #ffd166;
  --red: #ff6b6b;
  --purple: #c792ea;
  --blue: #82aaff;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background:
    radial-gradient(circle at top left, rgba(105, 210, 255, 0.12), transparent 36rem),
    radial-gradient(circle at bottom right, rgba(199, 146, 234, 0.10), transparent 32rem),
    var(--bg);
  color: var(--text);
}

button,
input,
textarea {
  font: inherit;
}

button {
  border: 1px solid rgba(105, 210, 255, 0.35);
  background: rgba(105, 210, 255, 0.14);
  color: var(--text);
  border-radius: 0.75rem;
  padding: 0.65rem 0.9rem;
  cursor: pointer;
}

button:hover {
  background: rgba(105, 210, 255, 0.22);
}

button:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

button.secondary {
  border-color: var(--border);
  background: rgba(255, 255, 255, 0.04);
}

button.danger {
  border-color: rgba(255, 107, 107, 0.5);
  background: rgba(255, 107, 107, 0.15);
}

input,
textarea {
  width: 100%;
  background: #090d16;
  color: var(--text);
  border: 1px solid var(--border);
  border-radius: 0.75rem;
  padding: 0.75rem;
  outline: none;
}

textarea {
  resize: vertical;
}

input:focus,
textarea:focus {
  border-color: var(--accent);
}

pre {
  white-space: pre-wrap;
  word-break: break-word;
  background: rgba(0, 0, 0, 0.22);
  border: 1px solid var(--border);
  border-radius: 0.7rem;
  padding: 0.75rem;
  overflow: auto;
}

.app-shell {
  min-height: 100vh;
  padding: 1.25rem;
}

.topbar {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  margin-bottom: 1rem;
}

.topbar h1 {
  margin: 0;
  letter-spacing: -0.04em;
}

.topbar p {
  margin: 0.25rem 0 0;
  color: var(--muted);
}

.status-pill {
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
  border: 1px solid var(--border);
  border-radius: 999px;
  padding: 0.55rem 0.9rem;
  min-width: 14rem;
  text-align: right;
}

.status-pill.ok {
  border-color: rgba(85, 214, 139, 0.5);
  color: var(--green);
}

.status-pill.bad {
  border-color: rgba(255, 107, 107, 0.45);
  color: var(--red);
}

.status-pill span {
  color: var(--muted);
  font-size: 0.78rem;
}

.error-banner {
  border: 1px solid rgba(255, 107, 107, 0.45);
  background: rgba(255, 107, 107, 0.12);
  color: var(--red);
  padding: 0.75rem 1rem;
  border-radius: 0.9rem;
  margin-bottom: 1rem;
}

.dashboard-grid {
  display: grid;
  grid-template-columns: minmax(20rem, 25rem) minmax(0, 1fr);
  gap: 1rem;
}

.left-column,
.main-column,
.stack {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.panel,
.metrics,
.permission-panel {
  border: 1px solid var(--border);
  background: rgba(16, 22, 36, 0.88);
  backdrop-filter: blur(10px);
  border-radius: 1rem;
  padding: 1rem;
}

.panel-title {
  font-weight: 700;
  margin-bottom: 0.8rem;
}

.inline-form {
  display: flex;
  gap: 0.5rem;
}

.inline-form input {
  flex: 1;
}

.two-col {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.5rem;
}

.button-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.selected-box {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  margin-top: 1rem;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 0.9rem;
  padding: 0.8rem;
}

.selected-box small,
.list-item small,
.list-card small {
  color: var(--muted);
}

.list {
  display: flex;
  flex-direction: column;
  gap: 0.55rem;
  margin-top: 0.8rem;
}

.list-item {
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
  border-color: var(--border);
  background: rgba(255, 255, 255, 0.035);
}

.list-item.selected {
  border-color: var(--accent);
  background: rgba(105, 210, 255, 0.12);
}

.list-card {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
  border: 1px solid var(--border);
  border-radius: 0.8rem;
  padding: 0.7rem;
  background: rgba(255, 255, 255, 0.035);
}

.composer textarea {
  min-height: 9rem;
}

.run-status {
  border-radius: 0.75rem;
  border: 1px solid var(--border);
  padding: 0.6rem 0.75rem;
  color: var(--muted);
}

.run-status.running {
  color: var(--yellow);
}

.run-status.completed {
  color: var(--green);
}

.run-status.failed,
.run-status.aborted {
  color: var(--red);
}

.metrics {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 0.75rem;
}

.metrics div {
  background: rgba(255, 255, 255, 0.035);
  border: 1px solid var(--border);
  border-radius: 0.85rem;
  padding: 0.8rem;
}

.metrics strong {
  display: block;
  font-size: 1.2rem;
}

.metrics span {
  color: var(--muted);
  font-size: 0.82rem;
}

.permission-panel {
  border-color: rgba(255, 209, 102, 0.45);
}

.permission-panel h3 {
  margin: 0 0 0.75rem;
  color: var(--yellow);
}

.permission-card {
  border: 1px solid rgba(255, 209, 102, 0.35);
  background: rgba(255, 209, 102, 0.08);
  border-radius: 0.9rem;
  padding: 0.85rem;
  margin-top: 0.6rem;
}

.permission-card div:first-child {
  display: flex;
  justify-content: space-between;
}

.timeline-panel {
  min-height: 42rem;
}

.timeline {
  height: 58vh;
  min-height: 35rem;
  overflow: auto;
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
  padding-right: 0.25rem;
}

.event-card {
  border: 1px solid var(--border);
  border-radius: 0.9rem;
  padding: 0.75rem;
  background: rgba(255, 255, 255, 0.035);
}

.event-card header {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  margin-bottom: 0.35rem;
}

.event-kind {
  font-weight: 700;
  font-size: 0.86rem;
}

.event-card time {
  color: var(--muted);
  font-size: 0.78rem;
}

.event-card p {
  margin: 0.25rem 0;
}

.event-card.model {
  border-color: rgba(130, 170, 255, 0.35);
}

.event-card.tool {
  border-color: rgba(199, 146, 234, 0.35);
}

.event-card.warning {
  border-color: rgba(255, 209, 102, 0.45);
}

.event-card.danger {
  border-color: rgba(255, 107, 107, 0.45);
}

.event-card.session {
  border-color: rgba(85, 214, 139, 0.35);
}

.delta {
  color: var(--text);
}

.empty {
  color: var(--muted);
  padding: 1rem;
  text-align: center;
}

.mini-row {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  border-top: 1px solid var(--border);
  padding: 0.4rem 0;
}

.danger-text {
  color: var(--red) !important;
}

@media (max-width: 980px) {
  .dashboard-grid {
    grid-template-columns: 1fr;
  }

  .metrics {
    grid-template-columns: repeat(2, 1fr);
  }

  .topbar {
    flex-direction: column;
  }

  .status-pill {
    text-align: left;
    border-radius: 1rem;
  }
}
```

---

# 10) Tests

## `tests/unit/dashboard/SseClient.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { buildRunStreamUrl } from "../../../apps/dashboard/src/api/SseClient";

describe("dashboard SseClient", () => {
  it("builds run stream URL", () => {
    const url = buildRunStreamUrl({
      gatewayUrl: "http://127.0.0.1:8787/",
      sessionId: "session 1",
      runId: "run/2"
    });

    expect(url).toBe(
      "http://127.0.0.1:8787/v1/sessions/session%201/runs/run%2F2/stream"
    );
  });
});
```

---

## `tests/unit/dashboard/GatewayClient.test.ts`

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";
import { GatewayClient } from "../../../apps/dashboard/src/api/GatewayClient";

describe("dashboard GatewayClient", () => {
  beforeEach(() => {
    vi.restoreAllMocks();
  });

  it("creates a session", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({
        ok: true,
        text: async () =>
          JSON.stringify({
            ok: true,
            session: {
              id: "s1",
              workingDir: "/tmp/project",
              modelConfig: { provider: "mock", model: "mock" },
              budget: {
                maxInputTokens: 100,
                softLimitTokens: 80,
                hardLimitTokens: 90,
                compactionMode: "auto"
              },
              createdAt: 1,
              lastCheckpointAt: 1,
              turnIndex: 0,
              totalInputTokens: 0,
              totalOutputTokens: 0,
              totalCostUsd: 0
            }
          })
      })
    );

    const client = new GatewayClient("http://localhost:8787");
    const session = await client.createSession();

    expect(session.id).toBe("s1");
    expect(fetch).toHaveBeenCalledWith(
      "http://localhost:8787/v1/sessions",
      expect.objectContaining({ method: "POST" })
    );
  });

  it("throws a clear API error", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({
        ok: false,
        status: 501,
        text: async () =>
          JSON.stringify({
            ok: false,
            error: {
              code: "CAPABILITY_NOT_CONFIGURED",
              message: "Memory adapter is not configured"
            }
          })
      })
    );

    const client = new GatewayClient("http://localhost:8787");

    await expect(client.listKairosTasks()).rejects.toThrow(
      "Memory adapter is not configured"
    );
  });
});
```

---

# 11) Manual validation after Pass 23

## Start Gateway

```bash
pnpm --filter @locoworker/app-gateway dev
```

## Start Dashboard

```bash
pnpm --filter @locoworker/app-dashboard dev
```

Then open:

```txt
http://127.0.0.1:5173
```

## Validate the core flow

1. Click **Connect**.
2. Create a session.
3. Enter a prompt in **Run**.
4. Click **Start run**.
5. Confirm that events appear live in the Event stream.
6. If a permission prompt appears, approve or deny it from the yellow permission card.
7. Confirm metrics update: model calls, tool calls, tokens, cost.

## Validate Kairos

1. Click **Schedule no-op**.
2. Click **Run due** after ~1 second.
3. Click **Refresh**.
4. Confirm task status changes.

## Validate Orchestrator

1. Enter a goal in the Orchestrator panel.
2. Click **Start orchestration**.
3. Click **Refresh**.
4. Select a run and inspect units / summary.

---

# 12) Pass 23 completion checklist

| Gate | Status |
|---|---|
| Dashboard builds as Vite React app | ✅ |
| Gateway URL can be configured from UI | ✅ |
| Dashboard can connect to `/v1/health` | ✅ |
| Dashboard can create/list/select sessions | ✅ |
| Dashboard can start Gateway runs | ✅ |
| Dashboard subscribes to Pass 20 SSE endpoint | ✅ |
| Agent events render in live timeline | ✅ |
| Model/tool/session/error/budget events render distinctly | ✅ |
| Metrics bar shows model calls/tool calls/tokens/cost | ✅ |
| Permission prompts can be approved/denied from UI | ✅ |
| Kairos tasks can be listed/scheduled/run-due | ✅ |
| Orchestrator runs can be started/listed/inspected | ✅ |
| Tests cover SSE URL builder and API client behavior | ✅ |

---

# 13) What Pass 23 unlocks

```txt
Pass 24 Desktop
  ├─ Can embed the Pass 20 Gateway.
  ├─ Can reuse the same Gateway REST/SSE contract used by Dashboard.
  ├─ Can expose Desktop IPC methods mirroring GatewayClient.
  ├─ Can use Dashboard UI concepts for event stream, permissions, sessions.
  └─ Can add Keychain package for secure provider key storage.
```

**Pass 23 is complete. Proceed with Pass 24 when ready.**
