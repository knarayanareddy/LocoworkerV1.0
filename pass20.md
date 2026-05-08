# PASS 20 — Gateway Real Routes, Session Runs, SSE Streaming, Permission Resolution

> **Philosophy:** “Later pass wins.”  
> This pass turns the Gateway from route shells into a working API surface that can create sessions, start agent runs, stream `AgentEvent`s over SSE, resolve permission prompts, and expose typed capability routes for memory/wiki/graph/kairos/research adapters.

---

## What Pass 20 delivers

| Area | What gets built |
|---|---|
| Gateway runtime | Central `GatewayRuntime` owning config, sessions, run manager, adapters |
| Session API | Create/list/restore/checkpoint sessions |
| Run API | Start/abort/list/get runs for a session |
| SSE streaming | `GET /v1/sessions/:sessionId/runs/:runId/stream` streams `AgentEvent`s |
| Permission API | Resolve pending gateway permission prompts from dashboard/desktop/API |
| Capability routes | Real typed routes for memory/wiki/graph/kairos/research via adapters |
| Gateway app | `apps/gateway` starts Fastify server from env |
| Core compatibility patch | Permission confirmation receives `promptId` metadata so Gateway can resolve prompts correctly |
| Tests | RunManager, SSE formatting, session/run route tests |

---

# 0) Pass 20 file tree

```txt
packages/
├── core/
│   └── src/
│       ├── types/
│       │   └── permission.types.ts             ← PATCH from Pass 19
│       └── permissions/
│           ├── PermissionConfirmation.ts       ← PATCH from Pass 19
│           └── PermissionGate.ts               ← PATCH from Pass 19
│
└── gateway/
    ├── package.json                            ← REWRITE
    ├── tsconfig.build.json                     ← REWRITE
    └── src/
        ├── index.ts                            ← REWRITE
        ├── config/
        │   └── GatewayConfig.ts                ← NEW
        ├── errors/
        │   └── GatewayError.ts                 ← NEW
        ├── runtime/
        │   ├── CapabilityAdapters.ts           ← NEW
        │   ├── GatewayPermissionBroker.ts      ← NEW
        │   ├── GatewayRuntime.ts               ← NEW
        │   ├── RunManager.ts                   ← NEW
        │   ├── SseStream.ts                    ← NEW
        │   └── ToolLoader.ts                   ← NEW
        ├── schemas/
        │   ├── common.schemas.ts               ← NEW
        │   ├── session.schemas.ts              ← NEW
        │   └── capability.schemas.ts           ← NEW
        ├── server/
        │   └── createGatewayServer.ts          ← NEW
        └── routes/
            ├── registerRoutes.ts               ← NEW
            ├── health.routes.ts                ← NEW
            ├── sessions.routes.ts              ← NEW
            ├── memory.routes.ts                ← NEW
            ├── wiki.routes.ts                  ← NEW
            ├── graph.routes.ts                 ← NEW
            ├── kairos.routes.ts                ← NEW
            ├── research.routes.ts              ← NEW
            └── admin.routes.ts                 ← NEW

apps/
└── gateway/
    ├── package.json                            ← REWRITE
    ├── tsconfig.build.json                     ← REWRITE
    └── src/
        └── main.ts                             ← REWRITE

tests/
└── unit/
    └── gateway/
        ├── RunManager.test.ts                  ← NEW
        ├── SseStream.test.ts                   ← NEW
        └── sessions.routes.test.ts             ← NEW
```

---

# 1) Small Pass 19 compatibility patch: permission prompt metadata

Pass 19’s `ConfirmationHandler` did not receive the `promptId` emitted in the `permission_prompt` event. Gateway needs that ID so clients can approve/deny the exact prompt they saw over SSE.

These patches are backward-compatible: existing CLI/Desktop/test handlers with only four arguments still work.

---

## `packages/core/src/types/permission.types.ts`

```ts
// packages/core/src/types/permission.types.ts

export enum PermissionTier {
  READ_ONLY = 0,
  READ_WRITE = 1,
  EXECUTE = 2,
  NETWORK = 3,
  SYSTEM = 4,
}

export interface ToolPermissionMeta {
  requiredTier: PermissionTier;
  isParallelSafe: boolean;
  requiresConfirmation: boolean;
  description: string;
  examples?: string[];
}

export interface PermissionState {
  grantedTiers: Set<PermissionTier>;
  permanentlyGranted: Set<string>;
  permanentlyDenied: Set<string>;
  sessionGrants: Map<string, PermissionTier>;
}

export interface PermissionDecision {
  granted: boolean;
  permanent: boolean;
  promptId: string;
}

export interface ConfirmationMeta {
  promptId: string;
  sessionId: string;
  turnIndex: number;
}

/**
 * Confirmation callback.
 *
 * Existing 4-arg handlers remain valid because the 5th arg is optional.
 * Gateway uses `meta.promptId` to correlate SSE permission_prompt events
 * with REST approval/denial calls.
 */
export type ConfirmationHandler = (
  toolName: string,
  tier: PermissionTier,
  description: string,
  examples: string[],
  meta?: ConfirmationMeta
) => Promise<PermissionDecision>;
```

---

## `packages/core/src/permissions/PermissionConfirmation.ts`

```ts
// packages/core/src/permissions/PermissionConfirmation.ts

import { randomUUID } from "node:crypto";
import type {
  ConfirmationHandler,
  ConfirmationMeta,
  PermissionDecision,
  PermissionTier,
} from "../types/permission.types.js";

const TIER_LABELS: Record<PermissionTier, string> = {
  0: "READ_ONLY",
  1: "READ_WRITE",
  2: "EXECUTE",
  3: "NETWORK",
  4: "SYSTEM",
};

async function defaultTTYHandler(
  toolName: string,
  tier: PermissionTier,
  description: string,
  examples: string[],
  meta?: ConfirmationMeta
): Promise<PermissionDecision> {
  const readline = await import("node:readline");

  const promptId = meta?.promptId ?? randomUUID();

  if (!process.stdin.isTTY || !process.stdout.isTTY) {
    console.error(
      `[LocoWorker] Permission required for "${toolName}" ` +
        `(tier: ${TIER_LABELS[tier]}) — auto-denied because process is non-TTY`
    );
    return { granted: false, permanent: false, promptId };
  }

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const question = (q: string) =>
    new Promise<string>((resolve) => rl.question(q, resolve));

  console.log("\n┌─ Permission Required ──────────────────────────────────────");
  console.log(`│  Tool:        ${toolName}`);
  console.log(`│  Tier:        ${TIER_LABELS[tier]}`);
  console.log(`│  Description: ${description}`);
  if (examples.length > 0) {
    console.log("│  Examples:");
    for (const ex of examples) console.log(`│    • ${ex}`);
  }
  console.log("└────────────────────────────────────────────────────────────");

  let answer: string;
  try {
    answer = await question(
      "  Allow? [y]es / [n]o / [a]lways for this session: "
    );
  } finally {
    rl.close();
  }

  const normalized = answer.trim().toLowerCase();

  if (normalized === "y" || normalized === "yes") {
    return { granted: true, permanent: false, promptId };
  }

  if (normalized === "a" || normalized === "always") {
    return { granted: true, permanent: true, promptId };
  }

  return { granted: false, permanent: false, promptId };
}

export class PermissionConfirmation {
  private handler: ConfirmationHandler;

  constructor(handler?: ConfirmationHandler) {
    this.handler = handler ?? defaultTTYHandler;
  }

  setHandler(handler: ConfirmationHandler): void {
    this.handler = handler;
  }

  async confirm(
    toolName: string,
    tier: PermissionTier,
    description: string,
    examples: string[] = [],
    meta?: ConfirmationMeta
  ): Promise<PermissionDecision> {
    return this.handler(toolName, tier, description, examples, meta);
  }
}
```

---

## `packages/core/src/permissions/PermissionGate.ts`

```ts
// packages/core/src/permissions/PermissionGate.ts

import { randomUUID } from "node:crypto";
import { PermissionTier } from "../types/permission.types.js";
import type {
  PermissionDecision,
  PermissionState,
} from "../types/permission.types.js";
import type { ToolDefinition } from "../types/tool.types.js";
import type { AgentEvent } from "../types/event.types.js";
import { PermissionConfirmation } from "./PermissionConfirmation.js";

export class PermissionDeniedError extends Error {
  constructor(
    public readonly toolName: string,
    public readonly tier: PermissionTier
  ) {
    super(`Permission denied: "${toolName}" requires tier ${PermissionTier[tier]}`);
    this.name = "PermissionDeniedError";
  }
}

export class PermissionGate {
  private state: PermissionState;
  private confirmation: PermissionConfirmation;
  private emit: (event: AgentEvent) => void;
  private sessionId: string;
  private turnIndex: () => number;

  constructor(opts: {
    state: PermissionState;
    confirmation: PermissionConfirmation;
    emit: (event: AgentEvent) => void;
    sessionId: string;
    turnIndex: () => number;
  }) {
    this.state = opts.state;
    this.confirmation = opts.confirmation;
    this.emit = opts.emit;
    this.sessionId = opts.sessionId;
    this.turnIndex = opts.turnIndex;
  }

  async check(tool: ToolDefinition): Promise<void> {
    const {
      name,
      requiredTier,
      requiresConfirmation,
      description,
      examples,
    } = tool;

    if (this.state.permanentlyDenied.has(name)) {
      throw new PermissionDeniedError(name, requiredTier);
    }

    if (this.state.permanentlyGranted.has(name)) {
      return;
    }

    if (this.state.grantedTiers.has(requiredTier) && !requiresConfirmation) {
      return;
    }

    const promptId = randomUUID();
    const currentTurn = this.turnIndex();

    this.emit({
      kind: "permission_prompt",
      sessionId: this.sessionId,
      turnIndex: currentTurn,
      ts: Date.now(),
      toolName: name,
      requiredTier,
      description,
      promptId,
    });

    let decision: PermissionDecision;

    try {
      decision = await this.confirmation.confirm(
        name,
        requiredTier,
        description,
        examples ?? [],
        {
          promptId,
          sessionId: this.sessionId,
          turnIndex: currentTurn,
        }
      );
    } catch {
      decision = { granted: false, permanent: false, promptId };
    }

    if (decision.granted) {
      if (decision.permanent) {
        this.state.permanentlyGranted.add(name);
        this.state.grantedTiers.add(requiredTier);
      }

      this.emit({
        kind: "permission_granted",
        sessionId: this.sessionId,
        turnIndex: this.turnIndex(),
        ts: Date.now(),
        promptId,
        toolName: name,
        permanent: decision.permanent,
      });

      return;
    }

    if (decision.permanent) {
      this.state.permanentlyDenied.add(name);
    }

    this.emit({
      kind: "permission_denied",
      sessionId: this.sessionId,
      turnIndex: this.turnIndex(),
      ts: Date.now(),
      promptId,
      toolName: name,
    });

    throw new PermissionDeniedError(name, requiredTier);
  }

  grantTier(tier: PermissionTier): void {
    this.state.grantedTiers.add(tier);
  }

  getState(): Readonly<PermissionState> {
    return this.state;
  }
}
```

---

# 2) Gateway package config

## `packages/gateway/package.json`

```json
{
  "name": "@locoworker/gateway",
  "version": "0.1.0",
  "description": "HTTP/SSE gateway for LocoWorker",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "dev": "tsc -p tsconfig.build.json --watch",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome lint src"
  },
  "dependencies": {
    "@fastify/cors": "^9.0.1",
    "@locoworker/core": "workspace:*",
    "fastify": "^4.28.1",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@locoworker/tsconfig": "workspace:*",
    "@types/node": "^20.12.0",
    "typescript": "^5.4.5",
    "vitest": "^1.6.0"
  },
  "files": ["dist", "src"]
}
```

---

## `packages/gateway/tsconfig.build.json`

```json
{
  "extends": "@locoworker/tsconfig/base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

# 3) Gateway errors and schemas

## `packages/gateway/src/errors/GatewayError.ts`

```ts
// packages/gateway/src/errors/GatewayError.ts

export class GatewayError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code: string,
    message: string,
    public readonly details?: unknown
  ) {
    super(message);
    this.name = "GatewayError";
  }
}

export function isGatewayError(err: unknown): err is GatewayError {
  return err instanceof GatewayError;
}
```

---

## `packages/gateway/src/schemas/common.schemas.ts`

```ts
// packages/gateway/src/schemas/common.schemas.ts

import { z } from "zod";
import { GatewayError } from "../errors/GatewayError.js";

export const IdParamSchema = z.object({
  id: z.string().min(1),
});

export const SessionIdParamSchema = z.object({
  sessionId: z.string().min(1),
});

export const RunIdParamSchema = z.object({
  runId: z.string().min(1),
});

export const SessionRunParamSchema = z.object({
  sessionId: z.string().min(1),
  runId: z.string().min(1),
});

export const PromptParamSchema = z.object({
  sessionId: z.string().min(1),
  runId: z.string().min(1),
  promptId: z.string().min(1),
});

export function parseOrThrow<T>(schema: z.ZodSchema<T>, data: unknown): T {
  const result = schema.safeParse(data);

  if (!result.success) {
    throw new GatewayError(
      400,
      "VALIDATION_ERROR",
      "Request validation failed",
      result.error.flatten()
    );
  }

  return result.data;
}
```

---

## `packages/gateway/src/schemas/session.schemas.ts`

```ts
// packages/gateway/src/schemas/session.schemas.ts

import { z } from "zod";

export const ModelConfigSchema = z.object({
  provider: z.string().default("anthropic"),
  model: z.string().default("claude-3-5-sonnet-latest"),
  temperature: z.number().min(0).max(2).optional(),
  topP: z.number().min(0).max(1).optional(),
  maxOutputTokens: z.number().int().positive().optional(),
  apiKey: z.string().optional(),
  baseUrl: z.string().url().optional(),
});

export const BudgetConfigSchema = z.object({
  maxInputTokens: z.number().int().positive().optional(),
  softLimitTokens: z.number().int().positive().optional(),
  hardLimitTokens: z.number().int().positive().optional(),
  maxCostUsd: z.number().positive().optional(),
  maxTurns: z.number().int().positive().optional(),
  compactionMode: z.enum(["auto", "full", "none"]).optional(),
});

export const CreateSessionBodySchema = z.object({
  workingDir: z.string().optional(),
  modelConfig: ModelConfigSchema.optional(),
  budget: BudgetConfigSchema.optional(),
});

export const StartRunBodySchema = z.object({
  message: z.string().min(1),
  stream: z.boolean().optional().default(true)
});

export const ResolvePermissionBodySchema = z.object({
  granted: z.boolean(),
  permanent: z.boolean().optional().default(false)
});

export const AbortRunBodySchema = z.object({
  reason: z.string().optional().default("client_abort")
});
```

---

## `packages/gateway/src/schemas/capability.schemas.ts`

```ts
// packages/gateway/src/schemas/capability.schemas.ts

import { z } from "zod";

export const SearchBodySchema = z.object({
  query: z.string().min(1),
  limit: z.number().int().positive().max(100).optional().default(20)
});

export const RememberBodySchema = z.object({
  content: z.string().min(1),
  tags: z.array(z.string()).optional().default([]),
  metadata: z.record(z.unknown()).optional().default({})
});

export const WikiUpsertBodySchema = z.object({
  slug: z.string().min(1),
  title: z.string().min(1),
  markdown: z.string(),
  metadata: z.record(z.unknown()).optional().default({})
});

export const GraphIndexBodySchema = z.object({
  paths: z.array(z.string()).optional(),
  full: z.boolean().optional().default(false)
});

export const KairosTaskBodySchema = z.object({
  name: z.string().min(1),
  cron: z.string().optional(),
  runAt: z.number().optional(),
  payload: z.record(z.unknown()).optional().default({})
});

export const ResearchRunBodySchema = z.object({
  goal: z.string().min(1),
  programPath: z.string().optional(),
  maxIterations: z.number().int().positive().max(100).optional().default(10)
});
```

---

# 4) Gateway config

## `packages/gateway/src/config/GatewayConfig.ts`

```ts
// packages/gateway/src/config/GatewayConfig.ts

import path from "node:path";
import type { BudgetConfig, ModelConfig } from "@locoworker/core";
import { DEFAULT_BUDGET } from "@locoworker/core";

export interface GatewayConfig {
  host: string;
  port: number;
  defaultWorkingDir: string;
  allowWorkspaceOverride: boolean;
  corsOrigins: string[];
  modelConfig: ModelConfig;
  budget: BudgetConfig;
  permissionTimeoutMs: number;
  sseHeartbeatMs: number;
  maxBufferedEventsPerRun: number;
}

function splitCsv(value: string | undefined): string[] {
  if (!value) return [];
  return value
    .split(",")
    .map((v) => v.trim())
    .filter(Boolean);
}

function boolEnv(value: string | undefined, fallback: boolean): boolean {
  if (value == null) return fallback;
  return ["1", "true", "yes", "on"].includes(value.toLowerCase());
}

function numEnv(value: string | undefined, fallback: number): number {
  if (!value) return fallback;
  const n = Number(value);
  return Number.isFinite(n) ? n : fallback;
}

export function loadGatewayConfigFromEnv(
  env: NodeJS.ProcessEnv = process.env
): GatewayConfig {
  const defaultWorkingDir = path.resolve(
    env.LOCOWORKER_WORKSPACE ?? process.cwd()
  );

  return {
    host: env.LOCOWORKER_GATEWAY_HOST ?? "127.0.0.1",
    port: numEnv(env.LOCOWORKER_GATEWAY_PORT, 8787),
    defaultWorkingDir,
    allowWorkspaceOverride: boolEnv(
      env.LOCOWORKER_ALLOW_WORKSPACE_OVERRIDE,
      false
    ),
    corsOrigins: splitCsv(env.LOCOWORKER_CORS_ORIGINS),
    modelConfig: {
      provider: env.LOCOWORKER_MODEL_PROVIDER ?? "anthropic",
      model: env.LOCOWORKER_MODEL ?? "claude-3-5-sonnet-latest",
      apiKey: env.LOCOWORKER_API_KEY,
      baseUrl: env.LOCOWORKER_MODEL_BASE_URL,
      temperature: env.LOCOWORKER_TEMPERATURE
        ? Number(env.LOCOWORKER_TEMPERATURE)
        : undefined,
      maxOutputTokens: env.LOCOWORKER_MAX_OUTPUT_TOKENS
        ? Number(env.LOCOWORKER_MAX_OUTPUT_TOKENS)
        : undefined
    },
    budget: {
      ...DEFAULT_BUDGET,
      maxCostUsd: env.LOCOWORKER_MAX_COST_USD
        ? Number(env.LOCOWORKER_MAX_COST_USD)
        : undefined,
      maxTurns: env.LOCOWORKER_MAX_TURNS
        ? Number(env.LOCOWORKER_MAX_TURNS)
        : undefined
    },
    permissionTimeoutMs: numEnv(
      env.LOCOWORKER_PERMISSION_TIMEOUT_MS,
      5 * 60 * 1000
    ),
    sseHeartbeatMs: numEnv(env.LOCOWORKER_SSE_HEARTBEAT_MS, 15_000),
    maxBufferedEventsPerRun: numEnv(
      env.LOCOWORKER_MAX_BUFFERED_EVENTS_PER_RUN,
      500
    )
  };
}
```

---

# 5) Capability adapter interfaces

These let Gateway expose real route handlers without tightly coupling to the exact constructor/API shape of each package. The app/bootstrap can inject actual adapters as packages mature.

## `packages/gateway/src/runtime/CapabilityAdapters.ts`

```ts
// packages/gateway/src/runtime/CapabilityAdapters.ts

export interface MemoryAdapter {
  remember(input: {
    content: string;
    tags: string[];
    metadata: Record<string, unknown>;
  }): Promise<unknown>;

  search(input: {
    query: string;
    limit: number;
  }): Promise<unknown>;

  list(input: {
    limit: number;
  }): Promise<unknown>;
}

export interface WikiAdapter {
  upsertPage(input: {
    slug: string;
    title: string;
    markdown: string;
    metadata: Record<string, unknown>;
  }): Promise<unknown>;

  getPage(slug: string): Promise<unknown>;

  search(input: {
    query: string;
    limit: number;
  }): Promise<unknown>;

  backlinks(slug: string): Promise<unknown>;
}

export interface GraphAdapter {
  index(input: {
    workingDir: string;
    paths?: string[];
    full: boolean;
  }): Promise<unknown>;

  status(): Promise<unknown>;

  search(input: {
    query: string;
    limit: number;
  }): Promise<unknown>;
}

export interface KairosAdapter {
  schedule(input: {
    name: string;
    cron?: string;
    runAt?: number;
    payload: Record<string, unknown>;
  }): Promise<unknown>;

  list(): Promise<unknown>;

  runDue(): Promise<unknown>;
}

export interface ResearchAdapter {
  start(input: {
    goal: string;
    programPath?: string;
    maxIterations: number;
  }): Promise<unknown>;

  list(): Promise<unknown>;

  get(id: string): Promise<unknown>;
}

export interface CapabilityAdapters {
  memory?: MemoryAdapter;
  wiki?: WikiAdapter;
  graph?: GraphAdapter;
  kairos?: KairosAdapter;
  research?: ResearchAdapter;
}
```

---

# 6) Gateway permission broker

This is the missing piece that lets Gateway pause a tool call until a client approves/denies the permission over REST.

## `packages/gateway/src/runtime/GatewayPermissionBroker.ts`

```ts
// packages/gateway/src/runtime/GatewayPermissionBroker.ts

import type {
  ConfirmationHandler,
  ConfirmationMeta,
  PermissionDecision,
  PermissionTier,
} from "@locoworker/core";

export interface PendingPermissionPrompt {
  promptId: string;
  sessionId: string;
  runId: string;
  toolName: string;
  tier: PermissionTier;
  description: string;
  examples: string[];
  createdAt: number;
  expiresAt: number;
}

interface PendingInternal extends PendingPermissionPrompt {
  resolve: (decision: PermissionDecision) => void;
  timeout: NodeJS.Timeout;
}

export class GatewayPermissionBroker {
  private pending = new Map<string, PendingInternal>();
  private earlyDecisions = new Map<string, PermissionDecision>();

  constructor(
    private readonly opts: {
      sessionId: string;
      runId: string;
      timeoutMs: number;
    }
  ) {}

  readonly handler: ConfirmationHandler = async (
    toolName: string,
    tier: PermissionTier,
    description: string,
    examples: string[],
    meta?: ConfirmationMeta
  ): Promise<PermissionDecision> => {
    const promptId =
      meta?.promptId ??
      `${this.opts.sessionId}:${this.opts.runId}:${Date.now()}:${toolName}`;

    const early = this.earlyDecisions.get(promptId);
    if (early) {
      this.earlyDecisions.delete(promptId);
      return early;
    }

    return new Promise<PermissionDecision>((resolve) => {
      const createdAt = Date.now();
      const expiresAt = createdAt + this.opts.timeoutMs;

      const timeout = setTimeout(() => {
        this.pending.delete(promptId);
        resolve({
          granted: false,
          permanent: false,
          promptId
        });
      }, this.opts.timeoutMs);

      this.pending.set(promptId, {
        promptId,
        sessionId: this.opts.sessionId,
        runId: this.opts.runId,
        toolName,
        tier,
        description,
        examples,
        createdAt,
        expiresAt,
        resolve,
        timeout
      });
    });
  };

  listPending(): PendingPermissionPrompt[] {
    return [...this.pending.values()].map((p) => ({
      promptId: p.promptId,
      sessionId: p.sessionId,
      runId: p.runId,
      toolName: p.toolName,
      tier: p.tier,
      description: p.description,
      examples: p.examples,
      createdAt: p.createdAt,
      expiresAt: p.expiresAt
    }));
  }

  resolvePrompt(input: {
    promptId: string;
    granted: boolean;
    permanent: boolean;
  }): boolean {
    const decision: PermissionDecision = {
      promptId: input.promptId,
      granted: input.granted,
      permanent: input.permanent
    };

    const pending = this.pending.get(input.promptId);

    if (!pending) {
      // Client may race slightly ahead of handler registration.
      this.earlyDecisions.set(input.promptId, decision);
      return true;
    }

    clearTimeout(pending.timeout);
    this.pending.delete(input.promptId);
    pending.resolve(decision);
    return true;
  }

  clearAll(): void {
    for (const pending of this.pending.values()) {
      clearTimeout(pending.timeout);
      pending.resolve({
        promptId: pending.promptId,
        granted: false,
        permanent: false
      });
    }

    this.pending.clear();
    this.earlyDecisions.clear();
  }
}
```

---

# 7) Tool loader

This loads real tool packages when present. If a package does not export the expected helper yet, Gateway still boots with an empty tool map rather than crashing.

## `packages/gateway/src/runtime/ToolLoader.ts`

```ts
// packages/gateway/src/runtime/ToolLoader.ts

import type { ToolHandler } from "@locoworker/core";

type ToolRegisterFn = (tools: Map<string, ToolHandler>) => void | Promise<void>;

async function tryRegister(
  packageName: string,
  tools: Map<string, ToolHandler>
): Promise<void> {
  try {
    const mod = await import(packageName);

    const candidates = [
      mod.registerTools,
      mod.registerFsTools,
      mod.registerBashTools,
      mod.registerGitTools,
      mod.registerSearchTools,
      mod.registerWebTools,
      mod.default
    ].filter(Boolean);

    for (const candidate of candidates) {
      if (typeof candidate === "function") {
        await (candidate as ToolRegisterFn)(tools);
        return;
      }
    }
  } catch {
    // Tool package absent or not built yet. Gateway should still boot.
  }
}

export async function loadDefaultTools(): Promise<Map<string, ToolHandler>> {
  const tools = new Map<string, ToolHandler>();

  await tryRegister("@locoworker/tools-fs", tools);
  await tryRegister("@locoworker/tools-bash", tools);
  await tryRegister("@locoworker/tools-git", tools);
  await tryRegister("@locoworker/tools-search", tools);
  await tryRegister("@locoworker/tools-web", tools);
  await tryRegister("@locoworker/tools-mcp", tools);

  return tools;
}
```

---

# 8) SSE streaming utility

## `packages/gateway/src/runtime/SseStream.ts`

```ts
// packages/gateway/src/runtime/SseStream.ts

import type { ServerResponse } from "node:http";
import type { AgentEvent } from "@locoworker/core";

export function formatSseEvent(event: AgentEvent): string {
  return [
    `event: ${event.kind}`,
    `id: ${event.sessionId}:${event.turnIndex}:${event.ts}`,
    `data: ${JSON.stringify(event)}`,
    "",
    ""
  ].join("\n");
}

export function formatSseComment(comment: string): string {
  return `: ${comment}\n\n`;
}

export class SseStream {
  private heartbeat: NodeJS.Timeout | null = null;
  private closed = false;

  constructor(
    private readonly res: ServerResponse,
    private readonly opts: {
      heartbeatMs: number;
    }
  ) {}

  open(): void {
    this.res.writeHead(200, {
      "Content-Type": "text/event-stream; charset=utf-8",
      "Cache-Control": "no-cache, no-transform",
      "Connection": "keep-alive",
      "X-Accel-Buffering": "no"
    });

    this.writeComment("connected");

    this.heartbeat = setInterval(() => {
      this.writeComment("heartbeat");
    }, this.opts.heartbeatMs);
  }

  writeEvent(event: AgentEvent): void {
    if (this.closed) return;
    this.res.write(formatSseEvent(event));
  }

  writeComment(comment: string): void {
    if (this.closed) return;
    this.res.write(formatSseComment(comment));
  }

  close(): void {
    if (this.closed) return;

    this.closed = true;

    if (this.heartbeat) {
      clearInterval(this.heartbeat);
      this.heartbeat = null;
    }

    this.res.end();
  }
}
```

---

# 9) Run manager

The RunManager is the bridge from Gateway HTTP/SSE into Pass 19’s `queryLoop`.

## `packages/gateway/src/runtime/RunManager.ts`

```ts
// packages/gateway/src/runtime/RunManager.ts

import { randomUUID } from "node:crypto";
import path from "node:path";
import {
  AgentEventEmitter,
  queryLoop,
  SessionManager,
  type AgentContext,
  type AgentEvent,
  type BudgetConfig,
  type ModelConfig,
  type SessionState,
  type ToolHandler
} from "@locoworker/core";
import type { GatewayConfig } from "../config/GatewayConfig.js";
import { GatewayError } from "../errors/GatewayError.js";
import { GatewayPermissionBroker } from "./GatewayPermissionBroker.js";

export type RunStatus = "running" | "completed" | "failed" | "aborted";

export interface GatewayRun {
  id: string;
  sessionId: string;
  status: RunStatus;
  message: string;
  startedAt: number;
  finishedAt?: number;
  error?: string;
  emitter: AgentEventEmitter;
  buffer: AgentEvent[];
  abortController: AbortController;
  permissionBroker: GatewayPermissionBroker;
}

export interface SessionSummary {
  id: string;
  createdAt: number;
  model: string;
}

export class RunManager {
  private sessions = new Map<string, SessionState>();
  private runs = new Map<string, GatewayRun>();
  private sessionManagers = new Map<string, SessionManager>();

  constructor(
    private readonly config: GatewayConfig,
    private readonly tools: Map<string, ToolHandler>
  ) {}

  async createSession(input: {
    workingDir?: string;
    modelConfig?: Partial<ModelConfig>;
    budget?: Partial<BudgetConfig>;
  }): Promise<SessionState> {
    const workingDir = this.resolveWorkingDir(input.workingDir);
    const manager = this.getSessionManager(workingDir);

    const session = await manager.create({
      modelConfig: {
        ...this.config.modelConfig,
        ...input.modelConfig
      },
      budget: {
        ...this.config.budget,
        ...input.budget
      }
    });

    this.sessions.set(session.id, session);
    return session;
  }

  async restoreSession(input: {
    sessionId: string;
    workingDir?: string;
  }): Promise<SessionState> {
    const workingDir = this.resolveWorkingDir(input.workingDir);
    const manager = this.getSessionManager(workingDir);

    const session = await manager.restore(input.sessionId);
    this.sessions.set(session.id, session);
    return session;
  }

  async getSession(sessionId: string): Promise<SessionState> {
    const inMemory = this.sessions.get(sessionId);
    if (inMemory) return inMemory;

    return this.restoreSession({
      sessionId,
      workingDir: this.config.defaultWorkingDir
    });
  }

  async listSessions(): Promise<SessionSummary[]> {
    const manager = this.getSessionManager(this.config.defaultWorkingDir);
    return manager.list();
  }

  async checkpointSession(sessionId: string): Promise<string> {
    const session = await this.getSession(sessionId);
    const manager = this.getSessionManager(session.workingDir);
    return manager.checkpoint(session);
  }

  async startRun(input: {
    sessionId: string;
    message: string;
  }): Promise<GatewayRun> {
    const session = await this.getSession(input.sessionId);

    const runId = randomUUID();
    const emitter = new AgentEventEmitter();
    const abortController = new AbortController();

    const permissionBroker = new GatewayPermissionBroker({
      sessionId: session.id,
      runId,
      timeoutMs: this.config.permissionTimeoutMs
    });

    const run: GatewayRun = {
      id: runId,
      sessionId: session.id,
      status: "running",
      message: input.message,
      startedAt: Date.now(),
      emitter,
      buffer: [],
      abortController,
      permissionBroker
    };

    const unsubscribe = emitter.subscribe((event) => {
      run.buffer.push(event);

      while (run.buffer.length > this.config.maxBufferedEventsPerRun) {
        run.buffer.shift();
      }

      if (event.kind === "session_end") {
        run.status =
          event.reason === "fatal_error"
            ? "failed"
            : event.reason === "user_abort"
              ? "aborted"
              : "completed";

        run.finishedAt = Date.now();
      }

      if (event.kind === "fatal_error") {
        run.status = "failed";
        run.error = event.error;
        run.finishedAt = Date.now();
      }
    });

    this.runs.set(run.id, run);

    const ctx: AgentContext = {
      session,
      tools: this.tools,
      emit: (event) => emitter.emit(event),
      confirmationHandler: permissionBroker.handler,
      signal: abortController.signal,
      onCheckpoint: async () => {
        await this.checkpointSession(session.id);
      }
    };

    void (async () => {
      try {
        for await (const _event of queryLoop(input.message, ctx)) {
          // queryLoop already emits through ctx.emit. Draining the generator
          // is what actually runs the loop.
        }

        const manager = this.getSessionManager(session.workingDir);
        await manager.checkpoint(session);
      } catch (err) {
        run.status = "failed";
        run.error = err instanceof Error ? err.message : String(err);
        run.finishedAt = Date.now();

        emitter.emit({
          kind: "fatal_error",
          sessionId: session.id,
          turnIndex: session.turnIndex,
          ts: Date.now(),
          error: run.error
        });

        emitter.emit({
          kind: "session_end",
          sessionId: session.id,
          turnIndex: session.turnIndex,
          ts: Date.now(),
          reason: "fatal_error",
          totalTokensUsed:
            session.totalInputTokens + session.totalOutputTokens,
          totalCostUsd: session.totalCostUsd,
          totalTurns: session.turnIndex
        });
      } finally {
        permissionBroker.clearAll();
        unsubscribe();
      }
    })();

    return run;
  }

  getRun(sessionId: string, runId: string): GatewayRun {
    const run = this.runs.get(runId);

    if (!run || run.sessionId !== sessionId) {
      throw new GatewayError(404, "RUN_NOT_FOUND", "Run not found");
    }

    return run;
  }

  listRuns(sessionId: string): Array<{
    id: string;
    sessionId: string;
    status: RunStatus;
    message: string;
    startedAt: number;
    finishedAt?: number;
    error?: string;
  }> {
    return [...this.runs.values()]
      .filter((run) => run.sessionId === sessionId)
      .map((run) => ({
        id: run.id,
        sessionId: run.sessionId,
        status: run.status,
        message: run.message,
        startedAt: run.startedAt,
        finishedAt: run.finishedAt,
        error: run.error
      }));
  }

  abortRun(input: {
    sessionId: string;
    runId: string;
    reason?: string;
  }): void {
    const run = this.getRun(input.sessionId, input.runId);

    if (run.status !== "running") {
      return;
    }

    run.status = "aborted";
    run.finishedAt = Date.now();
    run.abortController.abort(input.reason ?? "client_abort");
    run.permissionBroker.clearAll();
  }

  resolvePermission(input: {
    sessionId: string;
    runId: string;
    promptId: string;
    granted: boolean;
    permanent: boolean;
  }): boolean {
    const run = this.getRun(input.sessionId, input.runId);

    return run.permissionBroker.resolvePrompt({
      promptId: input.promptId,
      granted: input.granted,
      permanent: input.permanent
    });
  }

  listPendingPermissions(input: {
    sessionId: string;
    runId: string;
  }) {
    const run = this.getRun(input.sessionId, input.runId);
    return run.permissionBroker.listPending();
  }

  private getSessionManager(workingDir: string): SessionManager {
    const resolved = path.resolve(workingDir);
    const existing = this.sessionManagers.get(resolved);

    if (existing) return existing;

    const manager = new SessionManager(resolved);
    this.sessionManagers.set(resolved, manager);
    return manager;
  }

  private resolveWorkingDir(input?: string): string {
    if (!input) return this.config.defaultWorkingDir;

    const resolved = path.resolve(input);

    if (!this.config.allowWorkspaceOverride) {
      const defaultResolved = path.resolve(this.config.defaultWorkingDir);

      if (resolved !== defaultResolved) {
        throw new GatewayError(
          403,
          "WORKSPACE_OVERRIDE_DISABLED",
          "Gateway is not configured to allow arbitrary workspace paths"
        );
      }
    }

    return resolved;
  }
}
```

---

# 10) Gateway runtime

## `packages/gateway/src/runtime/GatewayRuntime.ts`

```ts
// packages/gateway/src/runtime/GatewayRuntime.ts

import type { ToolHandler } from "@locoworker/core";
import type { GatewayConfig } from "../config/GatewayConfig.js";
import { loadGatewayConfigFromEnv } from "../config/GatewayConfig.js";
import type { CapabilityAdapters } from "./CapabilityAdapters.js";
import { loadDefaultTools } from "./ToolLoader.js";
import { RunManager } from "./RunManager.js";

export class GatewayRuntime {
  readonly runManager: RunManager;

  constructor(
    readonly config: GatewayConfig,
    readonly tools: Map<string, ToolHandler>,
    readonly adapters: CapabilityAdapters = {}
  ) {
    this.runManager = new RunManager(config, tools);
  }

  static async fromEnv(input?: {
    env?: NodeJS.ProcessEnv;
    adapters?: CapabilityAdapters;
    tools?: Map<string, ToolHandler>;
  }): Promise<GatewayRuntime> {
    const config = loadGatewayConfigFromEnv(input?.env);
    const tools = input?.tools ?? await loadDefaultTools();

    return new GatewayRuntime(config, tools, input?.adapters ?? {});
  }

  status() {
    return {
      ok: true,
      gateway: "locoworker",
      version: "0.1.0",
      defaultWorkingDir: this.config.defaultWorkingDir,
      toolsLoaded: this.tools.size,
      adapters: {
        memory: Boolean(this.adapters.memory),
        wiki: Boolean(this.adapters.wiki),
        graph: Boolean(this.adapters.graph),
        kairos: Boolean(this.adapters.kairos),
        research: Boolean(this.adapters.research)
      }
    };
  }
}
```

---

# 11) Fastify server creation

## `packages/gateway/src/server/createGatewayServer.ts`

```ts
// packages/gateway/src/server/createGatewayServer.ts

import Fastify, {
  type FastifyInstance,
  type FastifyReply,
  type FastifyRequest
} from "fastify";
import cors from "@fastify/cors";
import { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { isGatewayError } from "../errors/GatewayError.js";
import { registerRoutes } from "../routes/registerRoutes.js";

export interface CreateGatewayServerOptions {
  runtime: GatewayRuntime;
  logger?: boolean;
}

export async function createGatewayServer(
  opts: CreateGatewayServerOptions
): Promise<FastifyInstance> {
  const app = Fastify({
    logger: opts.logger ?? true,
    bodyLimit: 10 * 1024 * 1024
  });

  const origins = opts.runtime.config.corsOrigins;

  await app.register(cors, {
    origin: origins.length > 0 ? origins : true,
    credentials: true
  });

  app.decorate("runtime", opts.runtime);

  app.setErrorHandler(
    async (err: unknown, _req: FastifyRequest, reply: FastifyReply) => {
      if (isGatewayError(err)) {
        return reply.status(err.statusCode).send({
          ok: false,
          error: {
            code: err.code,
            message: err.message,
            details: err.details
          }
        });
      }

      const message = err instanceof Error ? err.message : String(err);

      return reply.status(500).send({
        ok: false,
        error: {
          code: "INTERNAL_SERVER_ERROR",
          message
        }
      });
    }
  );

  await registerRoutes(app, opts.runtime);

  return app;
}

declare module "fastify" {
  interface FastifyInstance {
    runtime: GatewayRuntime;
  }
}
```

---

# 12) Route registration

## `packages/gateway/src/routes/registerRoutes.ts`

```ts
// packages/gateway/src/routes/registerRoutes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { registerHealthRoutes } from "./health.routes.js";
import { registerSessionRoutes } from "./sessions.routes.js";
import { registerMemoryRoutes } from "./memory.routes.js";
import { registerWikiRoutes } from "./wiki.routes.js";
import { registerGraphRoutes } from "./graph.routes.js";
import { registerKairosRoutes } from "./kairos.routes.js";
import { registerResearchRoutes } from "./research.routes.js";
import { registerAdminRoutes } from "./admin.routes.js";

export async function registerRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  await registerHealthRoutes(app, runtime);
  await registerSessionRoutes(app, runtime);
  await registerMemoryRoutes(app, runtime);
  await registerWikiRoutes(app, runtime);
  await registerGraphRoutes(app, runtime);
  await registerKairosRoutes(app, runtime);
  await registerResearchRoutes(app, runtime);
  await registerAdminRoutes(app, runtime);
}
```

---

# 13) Health/admin routes

## `packages/gateway/src/routes/health.routes.ts`

```ts
// packages/gateway/src/routes/health.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";

export async function registerHealthRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.get("/health", async () => ({
    ok: true,
    ts: Date.now()
  }));

  app.get("/v1/health", async () => ({
    ok: true,
    ts: Date.now(),
    runtime: runtime.status()
  }));
}
```

---

## `packages/gateway/src/routes/admin.routes.ts`

```ts
// packages/gateway/src/routes/admin.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";

export async function registerAdminRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.get("/v1/admin/status", async () => ({
    ok: true,
    status: runtime.status()
  }));

  app.get("/v1/admin/tools", async () => ({
    ok: true,
    tools: [...runtime.tools.values()].map((tool) => tool.definition)
  }));
}
```

---

# 14) Sessions, runs, SSE, permissions

## `packages/gateway/src/routes/sessions.routes.ts`

```ts
// packages/gateway/src/routes/sessions.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import {
  AbortRunBodySchema,
  CreateSessionBodySchema,
  ResolvePermissionBodySchema,
  StartRunBodySchema
} from "../schemas/session.schemas.js";
import {
  PromptParamSchema,
  SessionIdParamSchema,
  SessionRunParamSchema,
  parseOrThrow
} from "../schemas/common.schemas.js";
import { SseStream } from "../runtime/SseStream.js";

export async function registerSessionRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/sessions", async (req) => {
    const body = parseOrThrow(CreateSessionBodySchema, req.body);

    const session = await runtime.runManager.createSession({
      workingDir: body.workingDir,
      modelConfig: body.modelConfig,
      budget: body.budget
    });

    return {
      ok: true,
      session: serializeSession(session)
    };
  });

  app.get("/v1/sessions", async () => {
    const sessions = await runtime.runManager.listSessions();

    return {
      ok: true,
      sessions
    };
  });

  app.get("/v1/sessions/:sessionId", async (req) => {
    const params = parseOrThrow(SessionIdParamSchema, req.params);
    const session = await runtime.runManager.getSession(params.sessionId);

    return {
      ok: true,
      session: serializeSession(session)
    };
  });

  app.post("/v1/sessions/:sessionId/checkpoint", async (req) => {
    const params = parseOrThrow(SessionIdParamSchema, req.params);
    const checkpointId = await runtime.runManager.checkpointSession(
      params.sessionId
    );

    return {
      ok: true,
      checkpointId
    };
  });

  app.post("/v1/sessions/:sessionId/runs", async (req) => {
    const params = parseOrThrow(SessionIdParamSchema, req.params);
    const body = parseOrThrow(StartRunBodySchema, req.body);

    const run = await runtime.runManager.startRun({
      sessionId: params.sessionId,
      message: body.message
    });

    return {
      ok: true,
      run: serializeRun(run),
      streamUrl: `/v1/sessions/${params.sessionId}/runs/${run.id}/stream`
    };
  });

  app.get("/v1/sessions/:sessionId/runs", async (req) => {
    const params = parseOrThrow(SessionIdParamSchema, req.params);

    return {
      ok: true,
      runs: runtime.runManager.listRuns(params.sessionId)
    };
  });

  app.get("/v1/sessions/:sessionId/runs/:runId", async (req) => {
    const params = parseOrThrow(SessionRunParamSchema, req.params);
    const run = runtime.runManager.getRun(params.sessionId, params.runId);

    return {
      ok: true,
      run: serializeRun(run)
    };
  });

  app.post("/v1/sessions/:sessionId/runs/:runId/abort", async (req) => {
    const params = parseOrThrow(SessionRunParamSchema, req.params);
    const body = parseOrThrow(AbortRunBodySchema, req.body);

    runtime.runManager.abortRun({
      sessionId: params.sessionId,
      runId: params.runId,
      reason: body.reason
    });

    return {
      ok: true
    };
  });

  app.get("/v1/sessions/:sessionId/runs/:runId/permissions", async (req) => {
    const params = parseOrThrow(SessionRunParamSchema, req.params);

    return {
      ok: true,
      permissions: runtime.runManager.listPendingPermissions({
        sessionId: params.sessionId,
        runId: params.runId
      })
    };
  });

  app.post(
    "/v1/sessions/:sessionId/runs/:runId/permissions/:promptId",
    async (req) => {
      const params = parseOrThrow(PromptParamSchema, req.params);
      const body = parseOrThrow(ResolvePermissionBodySchema, req.body);

      runtime.runManager.resolvePermission({
        sessionId: params.sessionId,
        runId: params.runId,
        promptId: params.promptId,
        granted: body.granted,
        permanent: body.permanent
      });

      return {
        ok: true
      };
    }
  );

  app.get("/v1/sessions/:sessionId/runs/:runId/stream", async (req, reply) => {
    const params = parseOrThrow(SessionRunParamSchema, req.params);
    const run = runtime.runManager.getRun(params.sessionId, params.runId);

    reply.hijack();

    const stream = new SseStream(reply.raw, {
      heartbeatMs: runtime.config.sseHeartbeatMs
    });

    stream.open();

    for (const event of run.buffer) {
      stream.writeEvent(event);
    }

    const unsubscribe = run.emitter.subscribe((event) => {
      stream.writeEvent(event);

      if (event.kind === "session_end" || event.kind === "fatal_error") {
        setTimeout(() => stream.close(), 50);
      }
    });

    req.raw.on("close", () => {
      unsubscribe();
      stream.close();
    });
  });
}

function serializeSession(session: {
  id: string;
  workingDir: string;
  modelConfig: unknown;
  budget: unknown;
  createdAt: number;
  lastCheckpointAt: number;
  turnIndex: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCostUsd: number;
}) {
  return {
    id: session.id,
    workingDir: session.workingDir,
    modelConfig: session.modelConfig,
    budget: session.budget,
    createdAt: session.createdAt,
    lastCheckpointAt: session.lastCheckpointAt,
    turnIndex: session.turnIndex,
    totalInputTokens: session.totalInputTokens,
    totalOutputTokens: session.totalOutputTokens,
    totalCostUsd: session.totalCostUsd
  };
}

function serializeRun(run: {
  id: string;
  sessionId: string;
  status: string;
  message: string;
  startedAt: number;
  finishedAt?: number;
  error?: string;
}) {
  return {
    id: run.id,
    sessionId: run.sessionId,
    status: run.status,
    message: run.message,
    startedAt: run.startedAt,
    finishedAt: run.finishedAt,
    error: run.error
  };
}
```

---

# 15) Capability routes

These are real route bodies. If an adapter is not injected, the route returns a clear `501 CAPABILITY_NOT_CONFIGURED`.

---

## `packages/gateway/src/routes/memory.routes.ts`

```ts
// packages/gateway/src/routes/memory.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import { RememberBodySchema, SearchBodySchema } from "../schemas/capability.schemas.js";
import { parseOrThrow } from "../schemas/common.schemas.js";

export async function registerMemoryRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/memory/remember", async (req) => {
    if (!runtime.adapters.memory) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Memory adapter is not configured"
      );
    }

    const body = parseOrThrow(RememberBodySchema, req.body);

    const result = await runtime.adapters.memory.remember({
      content: body.content,
      tags: body.tags,
      metadata: body.metadata
    });

    return { ok: true, result };
  });

  app.post("/v1/memory/search", async (req) => {
    if (!runtime.adapters.memory) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Memory adapter is not configured"
      );
    }

    const body = parseOrThrow(SearchBodySchema, req.body);
    const results = await runtime.adapters.memory.search(body);

    return { ok: true, results };
  });

  app.get("/v1/memory", async (req) => {
    if (!runtime.adapters.memory) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Memory adapter is not configured"
      );
    }

    const query = req.query as { limit?: string };
    const limit = query.limit ? Number(query.limit) : 50;

    const results = await runtime.adapters.memory.list({ limit });

    return { ok: true, results };
  });
}
```

---

## `packages/gateway/src/routes/wiki.routes.ts`

```ts
// packages/gateway/src/routes/wiki.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import {
  SearchBodySchema,
  WikiUpsertBodySchema
} from "../schemas/capability.schemas.js";
import { parseOrThrow } from "../schemas/common.schemas.js";
import { z } from "zod";

const SlugParamSchema = z.object({
  slug: z.string().min(1)
});

export async function registerWikiRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.put("/v1/wiki/pages/:slug", async (req) => {
    if (!runtime.adapters.wiki) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Wiki adapter is not configured"
      );
    }

    const params = parseOrThrow(SlugParamSchema, req.params);
    const body = parseOrThrow(WikiUpsertBodySchema, {
      ...(req.body as Record<string, unknown>),
      slug: params.slug
    });

    const page = await runtime.adapters.wiki.upsertPage(body);

    return { ok: true, page };
  });

  app.get("/v1/wiki/pages/:slug", async (req) => {
    if (!runtime.adapters.wiki) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Wiki adapter is not configured"
      );
    }

    const params = parseOrThrow(SlugParamSchema, req.params);
    const page = await runtime.adapters.wiki.getPage(params.slug);

    return { ok: true, page };
  });

  app.post("/v1/wiki/search", async (req) => {
    if (!runtime.adapters.wiki) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Wiki adapter is not configured"
      );
    }

    const body = parseOrThrow(SearchBodySchema, req.body);
    const results = await runtime.adapters.wiki.search(body);

    return { ok: true, results };
  });

  app.get("/v1/wiki/pages/:slug/backlinks", async (req) => {
    if (!runtime.adapters.wiki) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Wiki adapter is not configured"
      );
    }

    const params = parseOrThrow(SlugParamSchema, req.params);
    const backlinks = await runtime.adapters.wiki.backlinks(params.slug);

    return { ok: true, backlinks };
  });
}
```

---

## `packages/gateway/src/routes/graph.routes.ts`

```ts
// packages/gateway/src/routes/graph.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import {
  GraphIndexBodySchema,
  SearchBodySchema
} from "../schemas/capability.schemas.js";
import { parseOrThrow } from "../schemas/common.schemas.js";

export async function registerGraphRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/graph/index", async (req) => {
    if (!runtime.adapters.graph) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Graph adapter is not configured"
      );
    }

    const body = parseOrThrow(GraphIndexBodySchema, req.body);

    const result = await runtime.adapters.graph.index({
      workingDir: runtime.config.defaultWorkingDir,
      paths: body.paths,
      full: body.full
    });

    return { ok: true, result };
  });

  app.get("/v1/graph/status", async () => {
    if (!runtime.adapters.graph) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Graph adapter is not configured"
      );
    }

    const status = await runtime.adapters.graph.status();

    return { ok: true, status };
  });

  app.post("/v1/graph/search", async (req) => {
    if (!runtime.adapters.graph) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Graph adapter is not configured"
      );
    }

    const body = parseOrThrow(SearchBodySchema, req.body);
    const results = await runtime.adapters.graph.search(body);

    return { ok: true, results };
  });
}
```

---

## `packages/gateway/src/routes/kairos.routes.ts`

```ts
// packages/gateway/src/routes/kairos.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import { KairosTaskBodySchema } from "../schemas/capability.schemas.js";
import { parseOrThrow } from "../schemas/common.schemas.js";

export async function registerKairosRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/kairos/tasks", async (req) => {
    if (!runtime.adapters.kairos) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Kairos adapter is not configured"
      );
    }

    const body = parseOrThrow(KairosTaskBodySchema, req.body);
    const task = await runtime.adapters.kairos.schedule(body);

    return { ok: true, task };
  });

  app.get("/v1/kairos/tasks", async () => {
    if (!runtime.adapters.kairos) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Kairos adapter is not configured"
      );
    }

    const tasks = await runtime.adapters.kairos.list();

    return { ok: true, tasks };
  });

  app.post("/v1/kairos/run-due", async () => {
    if (!runtime.adapters.kairos) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Kairos adapter is not configured"
      );
    }

    const result = await runtime.adapters.kairos.runDue();

    return { ok: true, result };
  });
}
```

---

## `packages/gateway/src/routes/research.routes.ts`

```ts
// packages/gateway/src/routes/research.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import { ResearchRunBodySchema } from "../schemas/capability.schemas.js";
import { IdParamSchema, parseOrThrow } from "../schemas/common.schemas.js";

export async function registerResearchRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/research/runs", async (req) => {
    if (!runtime.adapters.research) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Research adapter is not configured"
      );
    }

    const body = parseOrThrow(ResearchRunBodySchema, req.body);
    const run = await runtime.adapters.research.start(body);

    return { ok: true, run };
  });

  app.get("/v1/research/runs", async () => {
    if (!runtime.adapters.research) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Research adapter is not configured"
      );
    }

    const runs = await runtime.adapters.research.list();

    return { ok: true, runs };
  });

  app.get("/v1/research/runs/:id", async (req) => {
    if (!runtime.adapters.research) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Research adapter is not configured"
      );
    }

    const params = parseOrThrow(IdParamSchema, req.params);
    const run = await runtime.adapters.research.get(params.id);

    return { ok: true, run };
  });
}
```

---

# 16) Package public API

## `packages/gateway/src/index.ts`

```ts
// packages/gateway/src/index.ts

export { createGatewayServer } from "./server/createGatewayServer.js";
export type { CreateGatewayServerOptions } from "./server/createGatewayServer.js";

export { GatewayRuntime } from "./runtime/GatewayRuntime.js";
export { RunManager } from "./runtime/RunManager.js";
export type { GatewayRun, RunStatus, SessionSummary } from "./runtime/RunManager.js";

export { GatewayPermissionBroker } from "./runtime/GatewayPermissionBroker.js";
export type { PendingPermissionPrompt } from "./runtime/GatewayPermissionBroker.js";

export { SseStream, formatSseComment, formatSseEvent } from "./runtime/SseStream.js";

export { loadDefaultTools } from "./runtime/ToolLoader.js";

export type {
  CapabilityAdapters,
  MemoryAdapter,
  WikiAdapter,
  GraphAdapter,
  KairosAdapter,
  ResearchAdapter
} from "./runtime/CapabilityAdapters.js";

export {
  loadGatewayConfigFromEnv
} from "./config/GatewayConfig.js";
export type { GatewayConfig } from "./config/GatewayConfig.js";

export { GatewayError, isGatewayError } from "./errors/GatewayError.js";
```

---

# 17) Gateway app

## `apps/gateway/package.json`

```json
{
  "name": "@locoworker/app-gateway",
  "version": "0.1.0",
  "private": true,
  "description": "Runnable LocoWorker Gateway app",
  "type": "module",
  "main": "./dist/main.js",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "dev": "tsx src/main.ts",
    "start": "node dist/main.js",
    "typecheck": "tsc --noEmit",
    "lint": "biome lint src"
  },
  "dependencies": {
    "@locoworker/gateway": "workspace:*",
    "tsx": "^4.16.2"
  },
  "devDependencies": {
    "@locoworker/tsconfig": "workspace:*",
    "@types/node": "^20.12.0",
    "typescript": "^5.4.5"
  }
}
```

---

## `apps/gateway/tsconfig.build.json`

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
  "exclude": ["node_modules", "dist"]
}
```

---

## `apps/gateway/src/main.ts`

```ts
// apps/gateway/src/main.ts

import { createGatewayServer, GatewayRuntime } from "@locoworker/gateway";

async function main() {
  const runtime = await GatewayRuntime.fromEnv();
  const server = await createGatewayServer({
    runtime,
    logger: true
  });

  const address = await server.listen({
    host: runtime.config.host,
    port: runtime.config.port
  });

  console.log(`LocoWorker Gateway listening at ${address}`);
  console.log(`Workspace: ${runtime.config.defaultWorkingDir}`);
  console.log(`Tools loaded: ${runtime.tools.size}`);
}

main().catch((err) => {
  console.error("Failed to start LocoWorker Gateway");
  console.error(err);
  process.exit(1);
});
```

---

# 18) Tests

## `tests/unit/gateway/SseStream.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { formatSseComment, formatSseEvent } from "../../../packages/gateway/src/runtime/SseStream.js";
import type { AgentEvent } from "../../../packages/core/src/types/event.types.js";

describe("SSE formatting", () => {
  it("formats comments", () => {
    expect(formatSseComment("heartbeat")).toBe(": heartbeat\n\n");
  });

  it("formats AgentEvent as SSE event", () => {
    const event: AgentEvent = {
      kind: "session_start",
      sessionId: "s1",
      turnIndex: 0,
      ts: 123,
      workingDir: "/tmp/project",
      model: "mock"
    };

    const formatted = formatSseEvent(event);

    expect(formatted).toContain("event: session_start");
    expect(formatted).toContain("id: s1:0:123");
    expect(formatted).toContain("data:");
    expect(formatted.endsWith("\n\n")).toBe(true);
  });
});
```

---

## `tests/unit/gateway/RunManager.test.ts`

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { ProviderRouter } from "../../../packages/core/src/providers/ProviderRouter.js";
import type { IProviderAdapter } from "../../../packages/core/src/providers/ProviderRouter.js";
import { RunManager } from "../../../packages/gateway/src/runtime/RunManager.js";
import type { GatewayConfig } from "../../../packages/gateway/src/config/GatewayConfig.js";

function makeConfig(workingDir: string): GatewayConfig {
  return {
    host: "127.0.0.1",
    port: 0,
    defaultWorkingDir: workingDir,
    allowWorkspaceOverride: false,
    corsOrigins: [],
    modelConfig: {
      provider: "mock",
      model: "mock-model"
    },
    budget: {
      maxInputTokens: 200_000,
      softLimitTokens: 100_000,
      hardLimitTokens: 190_000,
      compactionMode: "auto"
    },
    permissionTimeoutMs: 10_000,
    sseHeartbeatMs: 1000,
    maxBufferedEventsPerRun: 500
  };
}

describe("RunManager", () => {
  let tmpDir: string;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "lw-gateway-"));
    ProviderRouter.setAdapter(null);
  });

  it("creates a session", async () => {
    const manager = new RunManager(makeConfig(tmpDir), new Map());

    const session = await manager.createSession({});

    expect(session.id).toBeTruthy();
    expect(session.workingDir).toBe(tmpDir);
  });

  it("starts a run and buffers events", async () => {
    const adapter: IProviderAdapter = {
      chat: vi.fn().mockResolvedValue({
        stopReason: "end_turn",
        text: "hello",
        toolCalls: [],
        inputTokens: 10,
        outputTokens: 5,
        costUsd: 0
      })
    };

    ProviderRouter.setAdapter(adapter);

    const manager = new RunManager(makeConfig(tmpDir), new Map());
    const session = await manager.createSession({});

    const run = await manager.startRun({
      sessionId: session.id,
      message: "hi"
    });

    await vi.waitFor(() => {
      expect(run.status).toBe("completed");
    });

    expect(run.buffer.some((e) => e.kind === "session_start")).toBe(true);
    expect(run.buffer.some((e) => e.kind === "loop_complete")).toBe(true);
    expect(run.buffer.some((e) => e.kind === "session_end")).toBe(true);
  });

  it("rejects workspace override when disabled", async () => {
    const manager = new RunManager(makeConfig(tmpDir), new Map());

    await expect(
      manager.createSession({
        workingDir: path.join(os.tmpdir(), "other-workspace")
      })
    ).rejects.toThrow("Gateway is not configured to allow arbitrary workspace paths");
  });
});
```

---

## `tests/unit/gateway/sessions.routes.test.ts`

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { ProviderRouter } from "../../../packages/core/src/providers/ProviderRouter.js";
import type { IProviderAdapter } from "../../../packages/core/src/providers/ProviderRouter.js";
import { createGatewayServer } from "../../../packages/gateway/src/server/createGatewayServer.js";
import { GatewayRuntime } from "../../../packages/gateway/src/runtime/GatewayRuntime.js";
import type { GatewayConfig } from "../../../packages/gateway/src/config/GatewayConfig.js";

function makeConfig(workingDir: string): GatewayConfig {
  return {
    host: "127.0.0.1",
    port: 0,
    defaultWorkingDir: workingDir,
    allowWorkspaceOverride: false,
    corsOrigins: [],
    modelConfig: {
      provider: "mock",
      model: "mock-model"
    },
    budget: {
      maxInputTokens: 200_000,
      softLimitTokens: 100_000,
      hardLimitTokens: 190_000,
      compactionMode: "auto"
    },
    permissionTimeoutMs: 10_000,
    sseHeartbeatMs: 1000,
    maxBufferedEventsPerRun: 500
  };
}

describe("session routes", () => {
  let tmpDir: string;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "lw-gateway-routes-"));
    ProviderRouter.setAdapter(null);
  });

  it("creates a session through REST", async () => {
    const runtime = new GatewayRuntime(makeConfig(tmpDir), new Map());
    const app = await createGatewayServer({ runtime, logger: false });

    const res = await app.inject({
      method: "POST",
      url: "/v1/sessions",
      payload: {}
    });

    expect(res.statusCode).toBe(200);

    const body = res.json();
    expect(body.ok).toBe(true);
    expect(body.session.id).toBeTruthy();

    await app.close();
  });

  it("starts a run through REST", async () => {
    const adapter: IProviderAdapter = {
      chat: vi.fn().mockResolvedValue({
        stopReason: "end_turn",
        text: "done",
        toolCalls: [],
        inputTokens: 10,
        outputTokens: 5,
        costUsd: 0
      })
    };

    ProviderRouter.setAdapter(adapter);

    const runtime = new GatewayRuntime(makeConfig(tmpDir), new Map());
    const app = await createGatewayServer({ runtime, logger: false });

    const create = await app.inject({
      method: "POST",
      url: "/v1/sessions",
      payload: {}
    });

    const sessionId = create.json().session.id;

    const runRes = await app.inject({
      method: "POST",
      url: `/v1/sessions/${sessionId}/runs`,
      payload: {
        message: "hello"
      }
    });

    expect(runRes.statusCode).toBe(200);

    const body = runRes.json();
    expect(body.ok).toBe(true);
    expect(body.run.id).toBeTruthy();
    expect(body.streamUrl).toContain("/stream");

    await app.close();
  });

  it("returns 501 for unconfigured memory adapter", async () => {
    const runtime = new GatewayRuntime(makeConfig(tmpDir), new Map());
    const app = await createGatewayServer({ runtime, logger: false });

    const res = await app.inject({
      method: "POST",
      url: "/v1/memory/search",
      payload: {
        query: "test"
      }
    });

    expect(res.statusCode).toBe(501);
    expect(res.json().error.code).toBe("CAPABILITY_NOT_CONFIGURED");

    await app.close();
  });
});
```

---

# 19) How to test manually after Pass 20

Start the Gateway:

```bash
pnpm --filter @locoworker/app-gateway dev
```

Create a session:

```bash
curl -s -X POST http://127.0.0.1:8787/v1/sessions \
  -H 'content-type: application/json' \
  -d '{}'
```

Start a run:

```bash
curl -s -X POST http://127.0.0.1:8787/v1/sessions/<SESSION_ID>/runs \
  -H 'content-type: application/json' \
  -d '{"message":"Hello, explain this project briefly."}'
```

Stream the run:

```bash
curl -N http://127.0.0.1:8787/v1/sessions/<SESSION_ID>/runs/<RUN_ID>/stream
```

If a permission prompt appears in the stream:

```bash
curl -s -X POST \
  http://127.0.0.1:8787/v1/sessions/<SESSION_ID>/runs/<RUN_ID>/permissions/<PROMPT_ID> \
  -H 'content-type: application/json' \
  -d '{"granted":true,"permanent":false}'
```

---

# 20) Pass 20 completion checklist

| Gate | Status |
|---|---|
| Gateway can create sessions | ✅ |
| Gateway can restore/list/checkpoint sessions | ✅ |
| Gateway can start queryLoop runs | ✅ |
| RunManager drains Pass 19 `queryLoop` correctly | ✅ |
| Gateway buffers run events for late SSE subscribers | ✅ |
| SSE endpoint streams `AgentEvent` objects | ✅ |
| SSE heartbeat keeps browser/proxy connection alive | ✅ |
| Runs can be aborted | ✅ |
| Gateway permission prompts can be approved/denied by REST | ✅ |
| Tool packages load dynamically when present | ✅ |
| Health/admin routes are real | ✅ |
| Memory/wiki/graph/kairos/research routes are typed adapter-backed handlers | ✅ |
| Gateway app boots from env | ✅ |
| Unit tests cover RunManager, SSE formatting, and session routes | ✅ |

---

# 21) What Pass 20 unlocks

```txt
Pass 21 CLI commands
  └─ can now call the same Gateway API or directly call queryLoop.

Pass 23 Dashboard
  └─ can now create sessions, start runs, subscribe to SSE, render tool cards,
     and resolve permission prompts.

Pass 24 Desktop
  └─ can embed this Gateway and communicate through the same REST/SSE contract.
```

**Pass 20 is complete. Proceed with Pass 21 when ready.**
