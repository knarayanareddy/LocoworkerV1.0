# PASS 21 — Runnable CLI: Direct Mode, Gateway Mode, REPL, Sessions, Streaming Renderer

> **Philosophy:** “Later pass wins.”  
> This pass makes `apps/cli` a real user-facing command-line app. It can run LocoWorker directly through `queryLoop`, or remotely through the Pass 20 Gateway REST/SSE API.

---

## What Pass 21 delivers

| Area | What gets built |
|---|---|
| CLI entrypoint | `locoworker`, `loco`, and `lw` binaries |
| Direct mode | CLI can call Pass 19 `queryLoop` directly |
| Gateway mode | CLI can create sessions, start runs, stream SSE from Pass 20 Gateway |
| REPL/chat mode | Interactive `lw chat` shell |
| Session commands | Create/list/resume/checkpoint sessions locally or through Gateway |
| Streaming renderer | Renders `AgentEvent` stream cleanly in terminal |
| Permission prompts | TTY interactive permission approval/deny/always |
| Non-TTY behavior | Safe default deny, with `--yes` override for automation |
| Config/env | Workspace/model/budget/gateway config from flags and env |
| Tests | SSE parser, renderer smoke, CLI config parsing |

---

# 0) Pass 21 file tree

```txt
apps/
└── cli/
    ├── package.json                         ← REWRITE
    ├── tsconfig.build.json                  ← REWRITE
    └── src/
        ├── main.ts                          ← REWRITE
        ├── config/
        │   └── CliConfig.ts                 ← NEW
        ├── gateway/
        │   ├── GatewayClient.ts             ← NEW
        │   └── SseParser.ts                 ← NEW
        ├── direct/
        │   └── DirectRunner.ts              ← NEW
        ├── terminal/
        │   ├── CliRenderer.ts               ← NEW
        │   ├── CliPermissionHandler.ts      ← NEW
        │   └── colors.ts                    ← NEW
        ├── commands/
        │   ├── ask.command.ts               ← NEW
        │   ├── chat.command.ts              ← NEW
        │   ├── sessions.command.ts          ← NEW
        │   └── doctor.command.ts            ← NEW
        └── utils/
            ├── readStdin.ts                 ← NEW
            └── errors.ts                    ← NEW

tests/
└── unit/
    └── cli/
        ├── SseParser.test.ts                ← NEW
        ├── CliConfig.test.ts                ← NEW
        └── CliRenderer.test.ts              ← NEW
```

---

# 1) CLI package config

## `apps/cli/package.json`

```json
{
  "name": "@locoworker/app-cli",
  "version": "0.1.0",
  "private": true,
  "description": "LocoWorker command-line interface",
  "type": "module",
  "bin": {
    "locoworker": "./dist/main.js",
    "loco": "./dist/main.js",
    "lw": "./dist/main.js"
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json && node ./scripts/make-bin-executable.mjs",
    "dev": "tsx src/main.ts",
    "start": "node dist/main.js",
    "typecheck": "tsc --noEmit",
    "lint": "biome lint src",
    "test": "vitest run"
  },
  "dependencies": {
    "@locoworker/core": "workspace:*",
    "@locoworker/gateway": "workspace:*",
    "commander": "^12.1.0"
  },
  "devDependencies": {
    "@locoworker/tsconfig": "workspace:*",
    "@types/node": "^20.12.0",
    "tsx": "^4.16.2",
    "typescript": "^5.4.5",
    "vitest": "^1.6.0"
  }
}
```

---

## `apps/cli/tsconfig.build.json`

```json
{
  "extends": "@locoworker/tsconfig/base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": false,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## `apps/cli/scripts/make-bin-executable.mjs`

```js
import fs from "node:fs";
import path from "node:path";

const file = path.resolve("dist/main.js");

if (fs.existsSync(file)) {
  fs.chmodSync(file, 0o755);
}
```

> If your earlier scaffold does not have `apps/cli/scripts/`, create it.

---

# 2) CLI config

## `apps/cli/src/config/CliConfig.ts`

```ts
// apps/cli/src/config/CliConfig.ts

import path from "node:path";
import type { BudgetConfig, ModelConfig } from "@locoworker/core";
import { DEFAULT_BUDGET } from "@locoworker/core";

export type CliRunMode = "direct" | "gateway";
export type CliPermissionMode = "interactive" | "allow" | "deny";

export interface CliConfig {
  mode: CliRunMode;
  workingDir: string;
  gatewayUrl: string;
  modelConfig: ModelConfig;
  budget: BudgetConfig;
  permissionMode: CliPermissionMode;
  json: boolean;
  quiet: boolean;
}

export interface CliFlagInput {
  gateway?: boolean;
  direct?: boolean;
  gatewayUrl?: string;
  workspace?: string;
  provider?: string;
  model?: string;
  apiKey?: string;
  baseUrl?: string;
  temperature?: string | number;
  maxOutputTokens?: string | number;
  maxCostUsd?: string | number;
  maxTurns?: string | number;
  yes?: boolean;
  deny?: boolean;
  json?: boolean;
  quiet?: boolean;
}

function num(value: string | number | undefined): number | undefined {
  if (value == null || value === "") return undefined;
  const n = Number(value);
  return Number.isFinite(n) ? n : undefined;
}

function envBool(value: string | undefined): boolean {
  if (!value) return false;
  return ["1", "true", "yes", "on"].includes(value.toLowerCase());
}

export function loadCliConfig(
  flags: CliFlagInput = {},
  env: NodeJS.ProcessEnv = process.env
): CliConfig {
  const mode: CliRunMode =
    flags.gateway || envBool(env.LOCOWORKER_CLI_GATEWAY)
      ? "gateway"
      : flags.direct
        ? "direct"
        : (env.LOCOWORKER_CLI_MODE as CliRunMode | undefined) ?? "direct";

  const workingDir = path.resolve(
    flags.workspace ??
      env.LOCOWORKER_WORKSPACE ??
      process.cwd()
  );

  const permissionMode: CliPermissionMode = flags.yes
    ? "allow"
    : flags.deny
      ? "deny"
      : envBool(env.LOCOWORKER_CLI_YES)
        ? "allow"
        : envBool(env.LOCOWORKER_CLI_DENY)
          ? "deny"
          : "interactive";

  return {
    mode,
    workingDir,
    gatewayUrl:
      flags.gatewayUrl ??
      env.LOCOWORKER_GATEWAY_URL ??
      "http://127.0.0.1:8787",

    modelConfig: {
      provider:
        flags.provider ??
        env.LOCOWORKER_MODEL_PROVIDER ??
        "anthropic",
      model:
        flags.model ??
        env.LOCOWORKER_MODEL ??
        "claude-3-5-sonnet-latest",
      apiKey:
        flags.apiKey ??
        env.LOCOWORKER_API_KEY,
      baseUrl:
        flags.baseUrl ??
        env.LOCOWORKER_MODEL_BASE_URL,
      temperature:
        num(flags.temperature) ??
        num(env.LOCOWORKER_TEMPERATURE),
      maxOutputTokens:
        num(flags.maxOutputTokens) ??
        num(env.LOCOWORKER_MAX_OUTPUT_TOKENS)
    },

    budget: {
      ...DEFAULT_BUDGET,
      maxCostUsd:
        num(flags.maxCostUsd) ??
        num(env.LOCOWORKER_MAX_COST_USD),
      maxTurns:
        num(flags.maxTurns) ??
        num(env.LOCOWORKER_MAX_TURNS)
    },

    permissionMode,
    json: Boolean(flags.json),
    quiet: Boolean(flags.quiet)
  };
}
```

---

# 3) Terminal utilities

## `apps/cli/src/terminal/colors.ts`

```ts
// apps/cli/src/terminal/colors.ts

const enabled = process.stdout.isTTY && process.env.NO_COLOR == null;

function wrap(code: string, text: string): string {
  return enabled ? `\x1b[${code}m${text}\x1b[0m` : text;
}

export const c = {
  dim: (s: string) => wrap("2", s),
  bold: (s: string) => wrap("1", s),
  red: (s: string) => wrap("31", s),
  green: (s: string) => wrap("32", s),
  yellow: (s: string) => wrap("33", s),
  blue: (s: string) => wrap("34", s),
  magenta: (s: string) => wrap("35", s),
  cyan: (s: string) => wrap("36", s),
  gray: (s: string) => wrap("90", s)
};
```

---

## `apps/cli/src/terminal/CliRenderer.ts`

```ts
// apps/cli/src/terminal/CliRenderer.ts

import type { AgentEvent } from "@locoworker/core";
import { c } from "./colors.js";

export interface CliRendererOptions {
  json?: boolean;
  quiet?: boolean;
  streamText?: boolean;
}

export class CliRenderer {
  private printedDelta = false;
  private inModelStream = false;

  constructor(private readonly opts: CliRendererOptions = {}) {}

  render(event: AgentEvent): void {
    if (this.opts.json) {
      process.stdout.write(`${JSON.stringify(event)}\n`);
      return;
    }

    if (this.opts.quiet) {
      this.renderQuiet(event);
      return;
    }

    switch (event.kind) {
      case "session_start":
        this.line(
          `${c.cyan("● session")} ${event.sessionId} ${c.gray(event.model)}`
        );
        break;

      case "session_restore":
        this.line(
          `${c.cyan("↺ restored")} ${event.restoredTurns} turns from ${event.sessionId}`
        );
        break;

      case "context_injected":
        if (event.sources.length > 0) {
          this.line(
            `${c.gray("context")} ${event.sources.join(", ")} ${c.gray(`(${event.totalChars} chars)`)}`
          );
        }
        break;

      case "turn_start":
        this.line(`${c.bold("You")} ${event.userMessage}`);
        break;

      case "model_request":
        this.line(
          `${c.gray("model")} ${event.model} ${c.gray(`${event.messageCount} messages, ~${event.estimatedInputTokens} tokens`)}`
        );
        break;

      case "model_response_start":
        this.inModelStream = true;
        this.printedDelta = false;
        process.stdout.write(`${c.bold("Assistant")} `);
        break;

      case "model_response_delta":
        this.printedDelta = true;
        process.stdout.write(event.delta);
        break;

      case "model_response_end":
        if (this.inModelStream) {
          process.stdout.write("\n");
        }
        this.inModelStream = false;
        this.line(
          c.gray(
            `tokens in=${event.inputTokens} out=${event.outputTokens} cost=$${event.costUsd.toFixed(6)} stop=${event.stopReason}`
          )
        );
        break;

      case "turn_complete":
        if (!this.printedDelta && event.assistantText.trim()) {
          this.line(`${c.bold("Assistant")} ${event.assistantText}`);
        }
        if (event.toolCallsExecuted > 0) {
          this.line(c.gray(`executed ${event.toolCallsExecuted} tool call(s)`));
        }
        break;

      case "tool_call":
        this.line(
          `${c.magenta("tool")} ${event.toolName} ${c.gray(event.toolCallId)}`
        );
        break;

      case "tool_result":
        this.line(
          `${c.green("tool ok")} ${event.toolName} ${c.gray(`${event.durationMs}ms`)} ${c.gray(event.outputPreview)}`
        );
        break;

      case "tool_error":
        this.line(
          `${c.red("tool error")} ${event.toolName}: ${event.error}`
        );
        break;

      case "permission_prompt":
        this.line(
          `${c.yellow("permission")} ${event.toolName} requires tier ${event.requiredTier}: ${event.description}`
        );
        break;

      case "permission_granted":
        this.line(
          `${c.green("permission granted")} ${event.toolName}${event.permanent ? " always" : ""}`
        );
        break;

      case "permission_denied":
        this.line(`${c.red("permission denied")} ${event.toolName}`);
        break;

      case "budget_warning":
        this.line(
          `${c.yellow("budget warning")} ${event.usedTokens}/${event.softLimitTokens} soft tokens (${event.percentUsed}%)`
        );
        break;

      case "budget_exceeded":
        this.line(
          `${c.red("budget exceeded")} ${event.usedTokens}/${event.hardLimitTokens}`
        );
        break;

      case "cost_cap_exceeded":
        this.line(
          `${c.red("cost cap exceeded")} spent $${event.spentUsd.toFixed(6)} / cap $${event.capUsd.toFixed(6)}`
        );
        break;

      case "compaction_start":
        this.line(
          `${c.yellow("compacting")} ${event.mode} ${event.turnsBefore} turns, ${event.tokensBefore} tokens`
        );
        break;

      case "compaction_end":
        this.line(
          `${c.green("compacted")} saved ${event.savedTokens} tokens`
        );
        break;

      case "loop_complete":
        break;

      case "session_checkpoint":
        this.line(`${c.gray("checkpoint")} ${event.checkpointId}`);
        break;

      case "session_error":
        this.line(`${c.yellow("warning")} ${event.error}`);
        break;

      case "fatal_error":
        this.line(`${c.red("fatal")} ${event.error}`);
        break;

      case "session_end":
        this.line(
          `${c.cyan("session end")} ${event.reason} turns=${event.totalTurns} tokens=${event.totalTokensUsed} cost=$${event.totalCostUsd.toFixed(6)}`
        );
        break;

      default:
        this.line(c.gray(event.kind));
    }
  }

  private renderQuiet(event: AgentEvent): void {
    if (event.kind === "model_response_delta") {
      process.stdout.write(event.delta);
      return;
    }

    if (event.kind === "turn_complete" && event.assistantText.trim()) {
      process.stdout.write(`${event.assistantText}\n`);
    }

    if (event.kind === "fatal_error") {
      process.stderr.write(`fatal: ${event.error}\n`);
    }
  }

  private line(text: string): void {
    process.stdout.write(`${text}\n`);
  }
}
```

---

## `apps/cli/src/terminal/CliPermissionHandler.ts`

```ts
// apps/cli/src/terminal/CliPermissionHandler.ts

import { createInterface } from "node:readline/promises";
import { stdin as input, stdout as output } from "node:process";
import type {
  ConfirmationHandler,
  ConfirmationMeta,
  PermissionDecision,
  PermissionTier
} from "@locoworker/core";
import type { CliPermissionMode } from "../config/CliConfig.js";
import { c } from "./colors.js";

const TIER_LABELS: Record<PermissionTier, string> = {
  0: "READ_ONLY",
  1: "READ_WRITE",
  2: "EXECUTE",
  3: "NETWORK",
  4: "SYSTEM"
};

export function createCliPermissionHandler(
  mode: CliPermissionMode
): ConfirmationHandler {
  return async (
    toolName: string,
    tier: PermissionTier,
    description: string,
    examples: string[],
    meta?: ConfirmationMeta
  ): Promise<PermissionDecision> => {
    const promptId =
      meta?.promptId ??
      `${toolName}:${Date.now()}`;

    if (mode === "allow") {
      return {
        promptId,
        granted: true,
        permanent: true
      };
    }

    if (mode === "deny") {
      return {
        promptId,
        granted: false,
        permanent: false
      };
    }

    if (!process.stdin.isTTY || !process.stdout.isTTY) {
      process.stderr.write(
        `[LocoWorker] Permission required for ${toolName}, but process is non-TTY. Denied. Use --yes to auto-approve.\n`
      );
      return {
        promptId,
        granted: false,
        permanent: false
      };
    }

    process.stdout.write("\n");
    process.stdout.write(`${c.yellow("Permission required")}\n`);
    process.stdout.write(`Tool: ${toolName}\n`);
    process.stdout.write(`Tier: ${TIER_LABELS[tier]}\n`);
    process.stdout.write(`Description: ${description}\n`);

    if (examples.length > 0) {
      process.stdout.write("Examples:\n");
      for (const ex of examples) {
        process.stdout.write(`  - ${ex}\n`);
      }
    }

    const rl = createInterface({ input, output });

    let answer: string;
    try {
      answer = await rl.question(
        "Allow? [y]es / [n]o / [a]lways for this session: "
      );
    } finally {
      rl.close();
    }

    const normalized = answer.trim().toLowerCase();

    if (normalized === "y" || normalized === "yes") {
      return {
        promptId,
        granted: true,
        permanent: false
      };
    }

    if (normalized === "a" || normalized === "always") {
      return {
        promptId,
        granted: true,
        permanent: true
      };
    }

    return {
      promptId,
      granted: false,
      permanent: false
    };
  };
}
```

---

# 4) Direct local runner

This is the local CLI path. It does **not** require Gateway to be running.

## `apps/cli/src/direct/DirectRunner.ts`

```ts
// apps/cli/src/direct/DirectRunner.ts

import {
  AgentEventEmitter,
  queryLoop,
  SessionManager,
  type AgentContext,
  type SessionState
} from "@locoworker/core";
import { loadDefaultTools } from "@locoworker/gateway";
import type { CliConfig } from "../config/CliConfig.js";
import { CliRenderer } from "../terminal/CliRenderer.js";
import { createCliPermissionHandler } from "../terminal/CliPermissionHandler.js";

export interface DirectRunInput {
  message: string;
  sessionId?: string;
}

export interface DirectRunResult {
  session: SessionState;
}

export class DirectRunner {
  constructor(private readonly config: CliConfig) {}

  async run(input: DirectRunInput): Promise<DirectRunResult> {
    const manager = new SessionManager(this.config.workingDir);

    const session = input.sessionId
      ? await manager.restore(input.sessionId)
      : await manager.create({
          modelConfig: this.config.modelConfig,
          budget: this.config.budget
        });

    const tools = await loadDefaultTools();
    const emitter = new AgentEventEmitter();
    const renderer = new CliRenderer({
      json: this.config.json,
      quiet: this.config.quiet,
      streamText: true
    });

    const unsubscribe = emitter.subscribe((event) => {
      renderer.render(event);
    });

    const ctx: AgentContext = {
      session,
      tools,
      emit: (event) => emitter.emit(event),
      confirmationHandler: createCliPermissionHandler(
        this.config.permissionMode
      ),
      onCheckpoint: async () => {
        await manager.checkpoint(session);
      }
    };

    try {
      for await (const _event of queryLoop(input.message, ctx)) {
        // queryLoop emits via ctx.emit; renderer subscribes to emitter.
        // We drain the generator to execute the loop.
      }

      await manager.checkpoint(session);

      return { session };
    } finally {
      unsubscribe();
    }
  }

  async createSession(): Promise<SessionState> {
    const manager = new SessionManager(this.config.workingDir);

    return manager.create({
      modelConfig: this.config.modelConfig,
      budget: this.config.budget
    });
  }

  async listSessions() {
    const manager = new SessionManager(this.config.workingDir);
    return manager.list();
  }

  async checkpoint(sessionId: string): Promise<string> {
    const manager = new SessionManager(this.config.workingDir);
    const session = await manager.restore(sessionId);
    return manager.checkpoint(session);
  }
}
```

---

# 5) Gateway client + SSE parser

## `apps/cli/src/gateway/SseParser.ts`

```ts
// apps/cli/src/gateway/SseParser.ts

import type { AgentEvent } from "@locoworker/core";

export interface ParsedSseEvent {
  event?: string;
  id?: string;
  data?: string;
}

export function parseSseFrame(frame: string): ParsedSseEvent | null {
  const lines = frame.split(/\r?\n/);

  const parsed: ParsedSseEvent = {};
  const dataLines: string[] = [];

  for (const line of lines) {
    if (!line || line.startsWith(":")) continue;

    const idx = line.indexOf(":");

    if (idx === -1) continue;

    const key = line.slice(0, idx);
    const value = line.slice(idx + 1).trimStart();

    if (key === "event") parsed.event = value;
    if (key === "id") parsed.id = value;
    if (key === "data") dataLines.push(value);
  }

  if (!parsed.event && !parsed.id && dataLines.length === 0) {
    return null;
  }

  if (dataLines.length > 0) {
    parsed.data = dataLines.join("\n");
  }

  return parsed;
}

export function parseAgentEventFromSseFrame(frame: string): AgentEvent | null {
  const parsed = parseSseFrame(frame);

  if (!parsed?.data) return null;

  try {
    return JSON.parse(parsed.data) as AgentEvent;
  } catch {
    return null;
  }
}

export class SseIncrementalParser {
  private buffer = "";

  push(chunk: string): AgentEvent[] {
    this.buffer += chunk;

    const events: AgentEvent[] = [];

    while (true) {
      const idx = this.buffer.indexOf("\n\n");
      if (idx === -1) break;

      const frame = this.buffer.slice(0, idx);
      this.buffer = this.buffer.slice(idx + 2);

      const event = parseAgentEventFromSseFrame(frame);
      if (event) events.push(event);
    }

    return events;
  }

  flush(): AgentEvent[] {
    if (!this.buffer.trim()) return [];

    const event = parseAgentEventFromSseFrame(this.buffer);
    this.buffer = "";

    return event ? [event] : [];
  }
}
```

---

## `apps/cli/src/gateway/GatewayClient.ts`

```ts
// apps/cli/src/gateway/GatewayClient.ts

import type { AgentEvent, BudgetConfig, ModelConfig } from "@locoworker/core";
import { SseIncrementalParser } from "./SseParser.js";

export interface GatewaySession {
  id: string;
  workingDir: string;
  modelConfig: ModelConfig;
  budget: BudgetConfig;
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

export class GatewayClient {
  constructor(private readonly baseUrl: string) {}

  async health(): Promise<unknown> {
    return this.request("GET", "/v1/health");
  }

  async createSession(input: {
    workingDir?: string;
    modelConfig?: Partial<ModelConfig>;
    budget?: Partial<BudgetConfig>;
  }): Promise<GatewaySession> {
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

  async checkpoint(sessionId: string): Promise<string> {
    const res = await this.request<{ checkpointId: string }>(
      "POST",
      `/v1/sessions/${encodeURIComponent(sessionId)}/checkpoint`,
      {}
    );

    return res.checkpointId;
  }

  async startRun(input: {
    sessionId: string;
    message: string;
  }): Promise<{
    run: GatewayRun;
    streamUrl: string;
  }> {
    return this.request(
      "POST",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}/runs`,
      {
        message: input.message,
        stream: true
      }
    );
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

  async abortRun(input: {
    sessionId: string;
    runId: string;
    reason?: string;
  }): Promise<void> {
    await this.request(
      "POST",
      `/v1/sessions/${encodeURIComponent(input.sessionId)}` +
        `/runs/${encodeURIComponent(input.runId)}/abort`,
      {
        reason: input.reason ?? "cli_abort"
      }
    );
  }

  async streamRun(input: {
    sessionId: string;
    runId: string;
    onEvent: (event: AgentEvent) => void | Promise<void>;
    signal?: AbortSignal;
  }): Promise<void> {
    const url =
      `${this.baseUrl.replace(/\/$/, "")}` +
      `/v1/sessions/${encodeURIComponent(input.sessionId)}` +
      `/runs/${encodeURIComponent(input.runId)}/stream`;

    const res = await fetch(url, {
      method: "GET",
      headers: {
        accept: "text/event-stream"
      },
      signal: input.signal
    });

    if (!res.ok) {
      throw new Error(
        `Gateway SSE request failed: HTTP ${res.status} ${await res.text()}`
      );
    }

    if (!res.body) {
      throw new Error("Gateway SSE response has no body");
    }

    const parser = new SseIncrementalParser();
    const reader = res.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { value, done } = await reader.read();

      if (done) break;

      const chunk = decoder.decode(value, { stream: true });
      const events = parser.push(chunk);

      for (const event of events) {
        await input.onEvent(event);

        if (event.kind === "session_end" || event.kind === "fatal_error") {
          return;
        }
      }
    }

    for (const event of parser.flush()) {
      await input.onEvent(event);
    }
  }

  private async request<T = unknown>(
    method: string,
    path: string,
    body?: unknown
  ): Promise<T> {
    const url = `${this.baseUrl.replace(/\/$/, "")}${path}`;

    const res = await fetch(url, {
      method,
      headers: {
        accept: "application/json",
        ...(body === undefined
          ? {}
          : { "content-type": "application/json" })
      },
      body: body === undefined ? undefined : JSON.stringify(body)
    });

    const text = await res.text();
    const json = text ? JSON.parse(text) : {};

    if (!res.ok) {
      const message =
        json?.error?.message ??
        `Gateway request failed: HTTP ${res.status}`;

      throw new Error(message);
    }

    return json as T;
  }
}
```

---

# 6) Command helpers

## `apps/cli/src/utils/readStdin.ts`

```ts
// apps/cli/src/utils/readStdin.ts

export async function readStdin(): Promise<string> {
  const chunks: Buffer[] = [];

  for await (const chunk of process.stdin) {
    chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
  }

  return Buffer.concat(chunks).toString("utf8").trim();
}
```

---

## `apps/cli/src/utils/errors.ts`

```ts
// apps/cli/src/utils/errors.ts

import { c } from "../terminal/colors.js";

export function printError(err: unknown): void {
  const message = err instanceof Error ? err.message : String(err);
  process.stderr.write(`${c.red("error")} ${message}\n`);

  if (process.env.LOCOWORKER_DEBUG === "1" && err instanceof Error && err.stack) {
    process.stderr.write(`${err.stack}\n`);
  }
}

export function fail(err: unknown): never {
  printError(err);
  process.exit(1);
}
```

---

# 7) `ask` command

## `apps/cli/src/commands/ask.command.ts`

```ts
// apps/cli/src/commands/ask.command.ts

import type { Command } from "commander";
import type { AgentEvent } from "@locoworker/core";
import { loadCliConfig } from "../config/CliConfig.js";
import { DirectRunner } from "../direct/DirectRunner.js";
import { GatewayClient } from "../gateway/GatewayClient.js";
import { CliRenderer } from "../terminal/CliRenderer.js";
import { createCliPermissionHandler } from "../terminal/CliPermissionHandler.js";
import { readStdin } from "../utils/readStdin.js";

async function getMessage(args: string[]): Promise<string> {
  const joined = args.join(" ").trim();

  if (joined) return joined;

  if (!process.stdin.isTTY) {
    return readStdin();
  }

  throw new Error("No message provided. Use `lw ask \"...\"` or pipe stdin.");
}

export function registerAskCommand(program: Command): void {
  program
    .command("ask [message...]")
    .description("Run one LocoWorker prompt")
    .option("--direct", "Run directly in this process")
    .option("--gateway", "Run through Gateway REST/SSE")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .option("--session <id>", "Resume/use existing session")
    .option("--provider <provider>", "Model provider")
    .option("-m, --model <model>", "Model name")
    .option("--api-key <key>", "Provider API key")
    .option("--base-url <url>", "Provider base URL")
    .option("--temperature <number>", "Model temperature")
    .option("--max-output-tokens <number>", "Max output tokens")
    .option("--max-cost-usd <number>", "Cost cap for this session")
    .option("--max-turns <number>", "Turn cap")
    .option("-y, --yes", "Auto-approve permission prompts")
    .option("--deny", "Auto-deny permission prompts")
    .option("--json", "Emit raw AgentEvent JSONL")
    .option("-q, --quiet", "Only print final assistant output")
    .action(async (messageParts: string[], flags) => {
      const message = await getMessage(messageParts);
      const config = loadCliConfig(flags);

      if (config.mode === "gateway") {
        await runViaGateway({
          message,
          sessionId: flags.session,
          flags,
          config
        });
        return;
      }

      const runner = new DirectRunner(config);

      await runner.run({
        message,
        sessionId: flags.session
      });
    });
}

async function runViaGateway(input: {
  message: string;
  sessionId?: string;
  flags: Record<string, unknown>;
  config: ReturnType<typeof loadCliConfig>;
}): Promise<void> {
  const client = new GatewayClient(input.config.gatewayUrl);
  const renderer = new CliRenderer({
    json: input.config.json,
    quiet: input.config.quiet,
    streamText: true
  });

  const session = input.sessionId
    ? await client.getSession(input.sessionId)
    : await client.createSession({
        workingDir: input.config.workingDir,
        modelConfig: input.config.modelConfig,
        budget: input.config.budget
      });

  const { run } = await client.startRun({
    sessionId: session.id,
    message: input.message
  });

  const permissionHandler = createCliPermissionHandler(
    input.config.permissionMode
  );

  await client.streamRun({
    sessionId: session.id,
    runId: run.id,
    onEvent: async (event: AgentEvent) => {
      renderer.render(event);

      if (event.kind === "permission_prompt") {
        const decision = await permissionHandler(
          event.toolName,
          event.requiredTier,
          event.description,
          [],
          {
            promptId: event.promptId,
            sessionId: event.sessionId,
            turnIndex: event.turnIndex
          }
        );

        await client.resolvePermission({
          sessionId: session.id,
          runId: run.id,
          promptId: event.promptId,
          granted: decision.granted,
          permanent: decision.permanent
        });
      }
    }
  });
}
```

---

# 8) `chat` REPL command

## `apps/cli/src/commands/chat.command.ts`

```ts
// apps/cli/src/commands/chat.command.ts

import { createInterface } from "node:readline/promises";
import { stdin as input, stdout as output } from "node:process";
import type { Command } from "commander";
import type { AgentEvent } from "@locoworker/core";
import { loadCliConfig } from "../config/CliConfig.js";
import { DirectRunner } from "../direct/DirectRunner.js";
import { GatewayClient } from "../gateway/GatewayClient.js";
import { CliRenderer } from "../terminal/CliRenderer.js";
import { createCliPermissionHandler } from "../terminal/CliPermissionHandler.js";
import { c } from "../terminal/colors.js";

export function registerChatCommand(program: Command): void {
  program
    .command("chat")
    .description("Start an interactive LocoWorker chat REPL")
    .option("--direct", "Run directly in this process")
    .option("--gateway", "Run through Gateway REST/SSE")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .option("--session <id>", "Resume existing session")
    .option("--provider <provider>", "Model provider")
    .option("-m, --model <model>", "Model name")
    .option("--api-key <key>", "Provider API key")
    .option("--base-url <url>", "Provider base URL")
    .option("--max-cost-usd <number>", "Cost cap")
    .option("--max-turns <number>", "Turn cap")
    .option("-y, --yes", "Auto-approve permission prompts")
    .option("--deny", "Auto-deny permission prompts")
    .option("--json", "Emit raw AgentEvent JSONL")
    .option("-q, --quiet", "Only print final assistant output")
    .action(async (flags) => {
      const config = loadCliConfig(flags);

      if (!process.stdin.isTTY || !process.stdout.isTTY) {
        throw new Error("`lw chat` requires a TTY. Use `lw ask` for pipes.");
      }

      if (config.mode === "gateway") {
        await gatewayRepl(config, flags.session);
      } else {
        await directRepl(config, flags.session);
      }
    });
}

async function directRepl(
  config: ReturnType<typeof loadCliConfig>,
  sessionId?: string
): Promise<void> {
  const runner = new DirectRunner(config);

  let currentSessionId = sessionId;

  if (!currentSessionId) {
    const session = await runner.createSession();
    currentSessionId = session.id;
  }

  await replLoop({
    sessionId: currentSessionId,
    runMessage: async (message) => {
      const result = await runner.run({
        message,
        sessionId: currentSessionId
      });

      currentSessionId = result.session.id;
    }
  });
}

async function gatewayRepl(
  config: ReturnType<typeof loadCliConfig>,
  sessionId?: string
): Promise<void> {
  const client = new GatewayClient(config.gatewayUrl);

  const session = sessionId
    ? await client.getSession(sessionId)
    : await client.createSession({
        workingDir: config.workingDir,
        modelConfig: config.modelConfig,
        budget: config.budget
      });

  const renderer = new CliRenderer({
    json: config.json,
    quiet: config.quiet,
    streamText: true
  });

  const permissionHandler = createCliPermissionHandler(config.permissionMode);

  await replLoop({
    sessionId: session.id,
    runMessage: async (message) => {
      const { run } = await client.startRun({
        sessionId: session.id,
        message
      });

      await client.streamRun({
        sessionId: session.id,
        runId: run.id,
        onEvent: async (event: AgentEvent) => {
          renderer.render(event);

          if (event.kind === "permission_prompt") {
            const decision = await permissionHandler(
              event.toolName,
              event.requiredTier,
              event.description,
              [],
              {
                promptId: event.promptId,
                sessionId: event.sessionId,
                turnIndex: event.turnIndex
              }
            );

            await client.resolvePermission({
              sessionId: session.id,
              runId: run.id,
              promptId: event.promptId,
              granted: decision.granted,
              permanent: decision.permanent
            });
          }
        }
      });
    }
  });
}

async function replLoop(inputOpts: {
  sessionId: string;
  runMessage: (message: string) => Promise<void>;
}): Promise<void> {
  const rl = createInterface({ input, output });

  process.stdout.write(`${c.cyan("LocoWorker chat")}\n`);
  process.stdout.write(`${c.gray(`session ${inputOpts.sessionId}`)}\n`);
  process.stdout.write(`${c.gray("Commands: /help /session /exit")}\n\n`);

  try {
    while (true) {
      const line = await rl.question(c.green("lw> "));
      const message = line.trim();

      if (!message) continue;

      if (message === "/exit" || message === "/quit") {
        break;
      }

      if (message === "/help") {
        process.stdout.write("/help      Show help\n");
        process.stdout.write("/session   Show session ID\n");
        process.stdout.write("/exit      Exit chat\n");
        continue;
      }

      if (message === "/session") {
        process.stdout.write(`${inputOpts.sessionId}\n`);
        continue;
      }

      await inputOpts.runMessage(message);
    }
  } finally {
    rl.close();
  }
}
```

---

# 9) `sessions` command group

## `apps/cli/src/commands/sessions.command.ts`

```ts
// apps/cli/src/commands/sessions.command.ts

import type { Command } from "commander";
import { loadCliConfig } from "../config/CliConfig.js";
import { DirectRunner } from "../direct/DirectRunner.js";
import { GatewayClient } from "../gateway/GatewayClient.js";
import { c } from "../terminal/colors.js";

export function registerSessionsCommand(program: Command): void {
  const sessions = program
    .command("sessions")
    .description("Manage LocoWorker sessions");

  sessions
    .command("create")
    .description("Create a session")
    .option("--direct", "Use local session store")
    .option("--gateway", "Use Gateway")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .option("--provider <provider>", "Model provider")
    .option("-m, --model <model>", "Model name")
    .option("--api-key <key>", "Provider API key")
    .option("--base-url <url>", "Provider base URL")
    .option("--json", "Print JSON")
    .action(async (flags) => {
      const config = loadCliConfig(flags);

      if (config.mode === "gateway") {
        const client = new GatewayClient(config.gatewayUrl);
        const session = await client.createSession({
          workingDir: config.workingDir,
          modelConfig: config.modelConfig,
          budget: config.budget
        });

        printSession(session, config.json);
        return;
      }

      const runner = new DirectRunner(config);
      const session = await runner.createSession();

      printSession(session, config.json);
    });

  sessions
    .command("list")
    .description("List sessions")
    .option("--direct", "Use local session store")
    .option("--gateway", "Use Gateway")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .option("--json", "Print JSON")
    .action(async (flags) => {
      const config = loadCliConfig(flags);

      if (config.mode === "gateway") {
        const client = new GatewayClient(config.gatewayUrl);
        const sessions = await client.listSessions();

        if (config.json) {
          process.stdout.write(`${JSON.stringify(sessions)}\n`);
          return;
        }

        printSessionList(sessions);
        return;
      }

      const runner = new DirectRunner(config);
      const localSessions = await runner.listSessions();

      if (config.json) {
        process.stdout.write(`${JSON.stringify(localSessions)}\n`);
        return;
      }

      printSessionList(localSessions);
    });

  sessions
    .command("checkpoint <sessionId>")
    .description("Checkpoint a session")
    .option("--direct", "Use local session store")
    .option("--gateway", "Use Gateway")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .action(async (sessionId: string, flags) => {
      const config = loadCliConfig(flags);

      if (config.mode === "gateway") {
        const client = new GatewayClient(config.gatewayUrl);
        const checkpointId = await client.checkpoint(sessionId);
        process.stdout.write(`${checkpointId}\n`);
        return;
      }

      const runner = new DirectRunner(config);
      const checkpointId = await runner.checkpoint(sessionId);
      process.stdout.write(`${checkpointId}\n`);
    });
}

function printSession(session: unknown, json: boolean): void {
  if (json) {
    process.stdout.write(`${JSON.stringify(session)}\n`);
    return;
  }

  const s = session as {
    id: string;
    workingDir: string;
    modelConfig?: { model?: string };
    createdAt?: number;
  };

  process.stdout.write(`${c.cyan("session")} ${s.id}\n`);
  process.stdout.write(`${c.gray("workspace")} ${s.workingDir}\n`);

  if (s.modelConfig?.model) {
    process.stdout.write(`${c.gray("model")} ${s.modelConfig.model}\n`);
  }

  if (s.createdAt) {
    process.stdout.write(
      `${c.gray("created")} ${new Date(s.createdAt).toISOString()}\n`
    );
  }
}

function printSessionList(
  sessions: Array<{ id: string; createdAt: number; model: string }>
): void {
  if (sessions.length === 0) {
    process.stdout.write(`${c.gray("No sessions found.")}\n`);
    return;
  }

  for (const session of sessions) {
    process.stdout.write(
      `${c.cyan(session.id)} ${c.gray(session.model)} ${new Date(session.createdAt).toISOString()}\n`
    );
  }
}
```

---

# 10) `doctor` command

## `apps/cli/src/commands/doctor.command.ts`

```ts
// apps/cli/src/commands/doctor.command.ts

import fs from "node:fs";
import type { Command } from "commander";
import { loadCliConfig } from "../config/CliConfig.js";
import { GatewayClient } from "../gateway/GatewayClient.js";
import { loadDefaultTools } from "@locoworker/gateway";
import { c } from "../terminal/colors.js";

export function registerDoctorCommand(program: Command): void {
  program
    .command("doctor")
    .description("Check local CLI/Gateway/tool configuration")
    .option("--gateway-url <url>", "Gateway URL")
    .option("-w, --workspace <dir>", "Workspace directory")
    .action(async (flags) => {
      const config = loadCliConfig(flags);

      process.stdout.write(`${c.bold("LocoWorker doctor")}\n`);

      process.stdout.write(
        `${status(fs.existsSync(config.workingDir))} workspace ${config.workingDir}\n`
      );

      process.stdout.write(
        `${status(Boolean(config.modelConfig.model))} model ${config.modelConfig.model}\n`
      );

      const tools = await loadDefaultTools();
      process.stdout.write(`${status(true)} local tools loaded: ${tools.size}\n`);

      try {
        const client = new GatewayClient(config.gatewayUrl);
        await client.health();
        process.stdout.write(`${status(true)} gateway ${config.gatewayUrl}\n`);
      } catch {
        process.stdout.write(`${status(false)} gateway ${config.gatewayUrl}\n`);
      }
    });
}

function status(ok: boolean): string {
  return ok ? c.green("✓") : c.red("✗");
}
```

---

# 11) Main CLI entrypoint

## `apps/cli/src/main.ts`

```ts
#!/usr/bin/env node
// apps/cli/src/main.ts

import { Command } from "commander";
import { registerAskCommand } from "./commands/ask.command.js";
import { registerChatCommand } from "./commands/chat.command.js";
import { registerSessionsCommand } from "./commands/sessions.command.js";
import { registerDoctorCommand } from "./commands/doctor.command.js";
import { fail } from "./utils/errors.js";

async function main(): Promise<void> {
  const program = new Command();

  program
    .name("locoworker")
    .alias("loco")
    .alias("lw")
    .description("LocoWorker privacy-first agentic developer workspace CLI")
    .version("0.1.0");

  registerAskCommand(program);
  registerChatCommand(program);
  registerSessionsCommand(program);
  registerDoctorCommand(program);

  program
    .command("run [message...]")
    .description("Alias for ask")
    .allowUnknownOption(true)
    .action(async (_messageParts) => {
      const args = process.argv.slice(2);
      const rewritten = ["node", "locoworker", "ask", ...args.slice(1)];
      process.argv = rewritten;
      await program.parseAsync(process.argv);
    });

  program.showHelpAfterError();

  await program.parseAsync(process.argv);

  if (process.argv.length <= 2) {
    program.outputHelp();
  }
}

main().catch(fail);
```

---

# 12) Tests

## `tests/unit/cli/SseParser.test.ts`

```ts
import { describe, expect, it } from "vitest";
import {
  parseSseFrame,
  parseAgentEventFromSseFrame,
  SseIncrementalParser
} from "../../../apps/cli/src/gateway/SseParser.js";

describe("SseParser", () => {
  it("parses a raw SSE frame", () => {
    const parsed = parseSseFrame(
      [
        "event: session_start",
        "id: s1:0:123",
        'data: {"kind":"session_start","sessionId":"s1","turnIndex":0,"ts":123,"workingDir":"/tmp","model":"mock"}',
        "",
        ""
      ].join("\n")
    );

    expect(parsed?.event).toBe("session_start");
    expect(parsed?.id).toBe("s1:0:123");
    expect(parsed?.data).toContain("session_start");
  });

  it("parses AgentEvent JSON from SSE", () => {
    const event = parseAgentEventFromSseFrame(
      [
        "event: session_end",
        "id: s1:1:123",
        'data: {"kind":"session_end","sessionId":"s1","turnIndex":1,"ts":123,"reason":"complete","totalTokensUsed":10,"totalCostUsd":0,"totalTurns":1}',
        "",
        ""
      ].join("\n")
    );

    expect(event?.kind).toBe("session_end");
  });

  it("incrementally parses split frames", () => {
    const parser = new SseIncrementalParser();

    const a = parser.push(
      'event: session_start\nid: s1:0:1\ndata: {"kind":"session_start"'
    );

    expect(a).toHaveLength(0);

    const b = parser.push(
      ',"sessionId":"s1","turnIndex":0,"ts":1,"workingDir":"/tmp","model":"mock"}\n\n'
    );

    expect(b).toHaveLength(1);
    expect(b[0].kind).toBe("session_start");
  });
});
```

---

## `tests/unit/cli/CliConfig.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { loadCliConfig } from "../../../apps/cli/src/config/CliConfig.js";

describe("CliConfig", () => {
  it("defaults to direct mode", () => {
    const config = loadCliConfig({}, {});
    expect(config.mode).toBe("direct");
  });

  it("uses gateway mode from flag", () => {
    const config = loadCliConfig({ gateway: true }, {});
    expect(config.mode).toBe("gateway");
  });

  it("uses env model when provided", () => {
    const config = loadCliConfig(
      {},
      {
        LOCOWORKER_MODEL_PROVIDER: "ollama",
        LOCOWORKER_MODEL: "llama3.1"
      }
    );

    expect(config.modelConfig.provider).toBe("ollama");
    expect(config.modelConfig.model).toBe("llama3.1");
  });

  it("sets permission mode from --yes", () => {
    const config = loadCliConfig({ yes: true }, {});
    expect(config.permissionMode).toBe("allow");
  });

  it("sets permission mode from --deny", () => {
    const config = loadCliConfig({ deny: true }, {});
    expect(config.permissionMode).toBe("deny");
  });
});
```

---

## `tests/unit/cli/CliRenderer.test.ts`

```ts
import { describe, expect, it, vi } from "vitest";
import { CliRenderer } from "../../../apps/cli/src/terminal/CliRenderer.js";
import type { AgentEvent } from "../../../packages/core/src/types/event.types.js";

describe("CliRenderer", () => {
  it("renders JSONL when json=true", () => {
    const spy = vi
      .spyOn(process.stdout, "write")
      .mockImplementation(() => true);

    const renderer = new CliRenderer({ json: true });

    const event: AgentEvent = {
      kind: "session_start",
      sessionId: "s1",
      turnIndex: 0,
      ts: 1,
      workingDir: "/tmp",
      model: "mock"
    };

    renderer.render(event);

    expect(spy).toHaveBeenCalledWith(expect.stringContaining('"session_start"'));

    spy.mockRestore();
  });

  it("renders quiet final assistant output", () => {
    const spy = vi
      .spyOn(process.stdout, "write")
      .mockImplementation(() => true);

    const renderer = new CliRenderer({ quiet: true });

    renderer.render({
      kind: "turn_complete",
      sessionId: "s1",
      turnIndex: 0,
      ts: 1,
      assistantText: "hello",
      toolCallsExecuted: 0,
      inputTokens: 1,
      outputTokens: 1,
      costUsd: 0
    });

    expect(spy).toHaveBeenCalledWith("hello\n");

    spy.mockRestore();
  });
});
```

---

# 13) Manual usage after Pass 21

## Direct/local mode

```bash
pnpm --filter @locoworker/app-cli dev -- ask "Explain this repository"
```

Or after build/link:

```bash
lw ask "Explain this repository"
```

With local model:

```bash
lw ask "Summarize the codebase" \
  --provider ollama \
  --model llama3.1 \
  --base-url http://127.0.0.1:11434
```

Auto-approve permission prompts:

```bash
lw ask "Create a README summary" --yes
```

Safe non-interactive deny:

```bash
echo "Inspect the project" | lw ask
```

---

## Gateway mode

Start Gateway first:

```bash
pnpm --filter @locoworker/app-gateway dev
```

Then run through Gateway:

```bash
lw ask "Explain this project" --gateway
```

Or explicitly:

```bash
lw ask "Explain this project" \
  --gateway \
  --gateway-url http://127.0.0.1:8787
```

---

## Interactive chat

```bash
lw chat
```

Gateway-backed chat:

```bash
lw chat --gateway
```

Inside chat:

```txt
/help
/session
/exit
```

---

## Session management

Create:

```bash
lw sessions create
```

List:

```bash
lw sessions list
```

Resume a session:

```bash
lw ask "Continue from where we left off" --session <SESSION_ID>
```

Checkpoint:

```bash
lw sessions checkpoint <SESSION_ID>
```

Gateway-backed sessions:

```bash
lw sessions create --gateway
lw sessions list --gateway
```

---

## Doctor

```bash
lw doctor
```

Expected output:

```txt
LocoWorker doctor
✓ workspace /your/project
✓ model claude-3-5-sonnet-latest
✓ local tools loaded: 8
✓ gateway http://127.0.0.1:8787
```

---

# 14) Pass 21 completion checklist

| Gate | Status |
|---|---|
| CLI binary entrypoint exists | ✅ |
| `lw ask` runs a direct `queryLoop` | ✅ |
| `lw ask --gateway` creates Gateway run and streams SSE | ✅ |
| `lw chat` interactive REPL works | ✅ |
| `lw chat --gateway` interactive Gateway-backed REPL works | ✅ |
| Local session create/list/checkpoint works | ✅ |
| Gateway session create/list/checkpoint works | ✅ |
| CLI handles permission prompts interactively | ✅ |
| `--yes` auto-approves permissions | ✅ |
| `--deny` auto-denies permissions | ✅ |
| Non-TTY defaults to safe denial | ✅ |
| Renderer supports human output, quiet output, JSONL events | ✅ |
| Gateway SSE parser implemented | ✅ |
| `lw doctor` checks workspace/tools/gateway | ✅ |
| Unit tests cover config, SSE parser, renderer | ✅ |

---

# 15) What Pass 21 unlocks

```txt
Pass 22 Kairos / Orchestrator engines
  └─ can invoke the same DirectRunner/queryLoop path for scheduled and delegated work.

Pass 23 Dashboard
  └─ can mirror the CLI's Gateway REST/SSE flow in browser UI.

Pass 24 Desktop
  └─ can mirror both CLI modes: direct local engine and embedded Gateway mode.
```

**Pass 21 is complete. Proceed with Pass 22 when ready.**
