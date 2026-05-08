# PASS 22 — Kairos Engine, AutoDream Scheduling, Orchestrator Engine, File Locks

> **Philosophy:** “Later pass wins.”  
> This pass turns the previously hollow autonomy packages into real engines:
>
> - `packages/kairos`: durable scheduler / tick runner / temporal context / AutoDream scheduling hook.
> - `packages/orchestrator`: multi-agent work planner / supervisor loop / file locking / result aggregation.
> - Gateway patch: Gateway can now auto-wire Kairos + Orchestrator adapters.

---

## What Pass 22 delivers

| Area | What gets built |
|---|---|
| Kairos durable engine | Schedule tasks, persist tasks/runs, run due tasks, start/stop tick loop |
| Temporal context | Generate “time-aware” context for prompts/systems |
| AutoDream scheduling | Register recurring memory consolidation task |
| Kairos Gateway adapter | Pass 20 `/v1/kairos/*` routes become operational |
| Orchestrator engine | Plan work units, execute them with bounded concurrency, persist run state |
| File locks | Workspace-safe advisory locks to prevent multi-agent collisions |
| Agent supervisor | Runs work units through Pass 19 `queryLoop` |
| Result aggregation | Combines sub-agent outputs into final summary |
| Orchestrator Gateway adapter | New `/v1/orchestrator/*` API routes |
| Gateway runtime patch | Auto-instantiates Kairos + Orchestrator adapters |
| Tests | Kairos scheduling, file locks, delegation planner, orchestrator runner |

---

# 0) Pass 22 file tree

```txt
packages/
├── kairos/
│   ├── package.json                            ← REWRITE
│   ├── tsconfig.build.json                     ← REWRITE
│   └── src/
│       ├── index.ts                            ← REWRITE
│       ├── types.ts                            ← NEW
│       ├── engine/
│       │   ├── KairosEngine.ts                 ← NEW
│       │   ├── KairosStore.ts                  ← NEW
│       │   ├── ScheduleCalculator.ts           ← NEW
│       │   ├── TemporalContext.ts              ← NEW
│       │   └── AutoDreamScheduler.ts           ← NEW
│       └── gateway/
│           └── KairosGatewayAdapter.ts         ← NEW
│
├── orchestrator/
│   ├── package.json                            ← REWRITE
│   ├── tsconfig.build.json                     ← REWRITE
│   └── src/
│       ├── index.ts                            ← REWRITE
│       ├── types.ts                            ← NEW
│       ├── engine/
│       │   ├── OrchestratorEngine.ts           ← NEW
│       │   ├── OrchestratorStore.ts            ← NEW
│       │   ├── DelegationPlanner.ts            ← NEW
│       │   ├── AgentSupervisor.ts              ← NEW
│       │   ├── ResultAggregator.ts             ← NEW
│       │   └── FileLockManager.ts              ← NEW
│       └── gateway/
│           └── OrchestratorGatewayAdapter.ts   ← NEW
│
└── gateway/
    ├── package.json                            ← PATCH
    └── src/
        ├── runtime/
        │   ├── CapabilityAdapters.ts           ← PATCH
        │   └── GatewayRuntime.ts               ← PATCH
        ├── schemas/
        │   └── capability.schemas.ts           ← PATCH
        └── routes/
            ├── registerRoutes.ts               ← PATCH
            └── orchestrator.routes.ts          ← NEW

tests/
└── unit/
    ├── kairos/
    │   ├── ScheduleCalculator.test.ts          ← NEW
    │   └── KairosEngine.test.ts                ← NEW
    └── orchestrator/
        ├── FileLockManager.test.ts             ← NEW
        ├── DelegationPlanner.test.ts           ← NEW
        └── OrchestratorEngine.test.ts          ← NEW
```

---

# 1) `packages/kairos`

## `packages/kairos/package.json`

```json
{
  "name": "@locoworker/kairos",
  "version": "0.1.0",
  "description": "Kairos temporal scheduler and autonomous task runner for LocoWorker",
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
    "lint": "biome lint src"
  },
  "dependencies": {},
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

## `packages/kairos/tsconfig.build.json`

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

## `packages/kairos/src/types.ts`

```ts
// packages/kairos/src/types.ts

export type KairosTaskStatus =
  | "scheduled"
  | "running"
  | "completed"
  | "failed"
  | "cancelled";

export type KairosRunStatus =
  | "running"
  | "completed"
  | "failed"
  | "cancelled";

export interface KairosSchedule {
  /**
   * Unix epoch millis. For one-shot tasks.
   */
  runAt?: number;

  /**
   * Simple recurring interval.
   */
  intervalMs?: number;

  /**
   * Lightweight cron support:
   * - "@hourly"
   * - "@daily"
   * - "star-slash": star-slash N * * * *
   * - "0 * * * *"
   * - "0 0 * * *"
   *
   * Note: this is intentionally conservative. If you need full cron syntax,
   * replace ScheduleCalculator with a real cron parser.
   */
  cron?: string;
}

export interface KairosTask {
  id: string;
  name: string;
  kind: string;
  schedule: KairosSchedule;
  payload: Record<string, unknown>;
  status: KairosTaskStatus;
  attempts: number;
  maxAttempts: number;
  createdAt: number;
  updatedAt: number;
  nextRunAt?: number;
  lastRunAt?: number;
  lastError?: string;
}

export interface KairosRun {
  id: string;
  taskId: string;
  taskName: string;
  kind: string;
  status: KairosRunStatus;
  startedAt: number;
  finishedAt?: number;
  output?: unknown;
  error?: string;
}

export interface KairosTaskHandlerContext {
  workingDir: string;
  signal: AbortSignal;
  emit: (event: KairosEvent) => void;
}

export interface KairosTaskHandler {
  kind: string;
  handle: (
    task: KairosTask,
    ctx: KairosTaskHandlerContext
  ) => Promise<unknown>;
}

export type KairosEvent =
  | {
      kind: "task_scheduled";
      task: KairosTask;
      ts: number;
    }
  | {
      kind: "task_cancelled";
      taskId: string;
      ts: number;
    }
  | {
      kind: "task_started";
      task: KairosTask;
      run: KairosRun;
      ts: number;
    }
  | {
      kind: "task_completed";
      task: KairosTask;
      run: KairosRun;
      ts: number;
    }
  | {
      kind: "task_failed";
      task: KairosTask;
      run: KairosRun;
      error: string;
      ts: number;
    }
  | {
      kind: "tick";
      dueCount: number;
      ts: number;
    };

export interface KairosScheduleInput {
  name: string;
  kind: string;
  schedule: KairosSchedule;
  payload?: Record<string, unknown>;
  maxAttempts?: number;
}
```

---

## `packages/kairos/src/engine/ScheduleCalculator.ts`

```ts
// packages/kairos/src/engine/ScheduleCalculator.ts

import type { KairosSchedule } from "../types.js";

const MINUTE = 60_000;
const HOUR = 60 * MINUTE;
const DAY = 24 * HOUR;

export function calculateNextRunAt(
  schedule: KairosSchedule,
  from: number = Date.now()
): number | undefined {
  if (schedule.runAt && schedule.runAt > from) {
    return schedule.runAt;
  }

  if (schedule.intervalMs && schedule.intervalMs > 0) {
    return from + schedule.intervalMs;
  }

  if (schedule.cron) {
    return calculateCronish(schedule.cron, from);
  }

  if (schedule.runAt && schedule.runAt <= from) {
    return schedule.runAt;
  }

  return undefined;
}

function calculateCronish(expr: string, from: number): number | undefined {
  const trimmed = expr.trim();

  if (trimmed === "@hourly") {
    return ceilToNext(from, HOUR);
  }

  if (trimmed === "@daily") {
    return ceilToNext(from, DAY);
  }

  const parts = trimmed.split(/\s+/);

  if (parts.length !== 5) {
    return undefined;
  }

  const [minute, hour] = parts;

  // star-slash N * * * *
  if (minute.startsWith("*/") && hour === "*") {
    const n = Number(minute.slice(2));
    if (Number.isFinite(n) && n > 0) {
      return ceilToNext(from, n * MINUTE);
    }
  }

  // 0 * * * * — hourly at minute 0
  if (minute === "0" && hour === "*") {
    return ceilToNext(from, HOUR);
  }

  // 0 0 * * * — daily midnight-ish
  if (minute === "0" && hour === "0") {
    return ceilToNext(from, DAY);
  }

  return undefined;
}

function ceilToNext(from: number, intervalMs: number): number {
  return Math.ceil((from + 1) / intervalMs) * intervalMs;
}

export function isRecurringSchedule(schedule: KairosSchedule): boolean {
  return Boolean(schedule.intervalMs || schedule.cron);
}
```

---

## `packages/kairos/src/engine/KairosStore.ts`

```ts
// packages/kairos/src/engine/KairosStore.ts

import fs from "node:fs/promises";
import path from "node:path";
import type { KairosRun, KairosTask } from "../types.js";

interface KairosDb {
  tasks: KairosTask[];
  runs: KairosRun[];
}

const EMPTY_DB: KairosDb = {
  tasks: [],
  runs: []
};

export class KairosStore {
  private readonly dir: string;
  private readonly file: string;

  constructor(private readonly workingDir: string) {
    this.dir = path.join(workingDir, ".locoworker", "kairos");
    this.file = path.join(this.dir, "kairos.json");
  }

  async init(): Promise<void> {
    await fs.mkdir(this.dir, { recursive: true });

    try {
      await fs.access(this.file);
    } catch {
      await this.writeDb(EMPTY_DB);
    }
  }

  async listTasks(): Promise<KairosTask[]> {
    const db = await this.readDb();
    return db.tasks;
  }

  async getTask(taskId: string): Promise<KairosTask | undefined> {
    const db = await this.readDb();
    return db.tasks.find((t) => t.id === taskId);
  }

  async saveTask(task: KairosTask): Promise<void> {
    const db = await this.readDb();
    const index = db.tasks.findIndex((t) => t.id === task.id);

    if (index === -1) {
      db.tasks.push(task);
    } else {
      db.tasks[index] = task;
    }

    await this.writeDb(db);
  }

  async deleteTask(taskId: string): Promise<void> {
    const db = await this.readDb();
    db.tasks = db.tasks.filter((t) => t.id !== taskId);
    await this.writeDb(db);
  }

  async listRuns(taskId?: string): Promise<KairosRun[]> {
    const db = await this.readDb();
    return taskId ? db.runs.filter((r) => r.taskId === taskId) : db.runs;
  }

  async saveRun(run: KairosRun): Promise<void> {
    const db = await this.readDb();
    const index = db.runs.findIndex((r) => r.id === run.id);

    if (index === -1) {
      db.runs.push(run);
    } else {
      db.runs[index] = run;
    }

    await this.writeDb(db);
  }

  private async readDb(): Promise<KairosDb> {
    await this.initIfNeeded();

    try {
      const raw = await fs.readFile(this.file, "utf8");
      const parsed = JSON.parse(raw);
      return {
        tasks: Array.isArray(parsed.tasks) ? parsed.tasks : [],
        runs: Array.isArray(parsed.runs) ? parsed.runs : []
      };
    } catch {
      return { ...EMPTY_DB };
    }
  }

  private async writeDb(db: KairosDb): Promise<void> {
    await fs.mkdir(this.dir, { recursive: true });
    await fs.writeFile(this.file, JSON.stringify(db, null, 2), "utf8");
  }

  private async initIfNeeded(): Promise<void> {
    try {
      await fs.access(this.file);
    } catch {
      await this.init();
    }
  }
}
```

---

## `packages/kairos/src/engine/TemporalContext.ts`

```ts
// packages/kairos/src/engine/TemporalContext.ts

import type { KairosTask } from "../types.js";

export interface TemporalContextInput {
  now?: number;
  tasks: KairosTask[];
  horizonMs?: number;
}

export function buildTemporalContext(input: TemporalContextInput): string {
  const now = input.now ?? Date.now();
  const horizon = input.horizonMs ?? 24 * 60 * 60 * 1000;

  const due = input.tasks.filter(
    (t) => t.status === "scheduled" && t.nextRunAt && t.nextRunAt <= now
  );

  const upcoming = input.tasks
    .filter(
      (t) =>
        t.status === "scheduled" &&
        t.nextRunAt &&
        t.nextRunAt > now &&
        t.nextRunAt <= now + horizon
    )
    .sort((a, b) => (a.nextRunAt ?? 0) - (b.nextRunAt ?? 0));

  const lines: string[] = [];

  lines.push("## Temporal Context");
  lines.push(`Current time: ${new Date(now).toISOString()}`);

  if (due.length > 0) {
    lines.push("");
    lines.push("Due tasks:");
    for (const task of due) {
      lines.push(
        `- ${task.name} (${task.kind}) due ${new Date(task.nextRunAt!).toISOString()}`
      );
    }
  }

  if (upcoming.length > 0) {
    lines.push("");
    lines.push("Upcoming tasks:");
    for (const task of upcoming.slice(0, 20)) {
      lines.push(
        `- ${task.name} (${task.kind}) at ${new Date(task.nextRunAt!).toISOString()}`
      );
    }
  }

  if (due.length === 0 && upcoming.length === 0) {
    lines.push("");
    lines.push("No due or upcoming tasks in the current horizon.");
  }

  return lines.join("\n");
}
```

---

## `packages/kairos/src/engine/AutoDreamScheduler.ts`

```ts
// packages/kairos/src/engine/AutoDreamScheduler.ts

import type { KairosEngine } from "./KairosEngine.js";

export interface AutoDreamAdapter {
  runAutoDream(input: {
    workingDir: string;
    maxMemories?: number;
    dryRun?: boolean;
  }): Promise<unknown>;
}

export interface RegisterAutoDreamOptions {
  cron?: string;
  intervalMs?: number;
  maxMemories?: number;
  dryRun?: boolean;
}

/**
 * Registers the canonical "memory.autodream" task kind and schedules the
 * recurring AutoDream task.
 *
 * This intentionally depends on an adapter, not a concrete memory package API,
 * because earlier passes may have generated different memory internals.
 */
export async function registerAutoDreamTask(
  engine: KairosEngine,
  adapter: AutoDreamAdapter,
  opts: RegisterAutoDreamOptions = {}
): Promise<string> {
  engine.registerHandler({
    kind: "memory.autodream",
    handle: async (_task, ctx) => {
      return adapter.runAutoDream({
        workingDir: ctx.workingDir,
        maxMemories: opts.maxMemories,
        dryRun: opts.dryRun
      });
    }
  });

  const task = await engine.schedule({
    name: "AutoDream memory consolidation",
    kind: "memory.autodream",
    schedule: opts.intervalMs
      ? { intervalMs: opts.intervalMs }
      : { cron: opts.cron ?? "0 0 * * *" },
    payload: {
      maxMemories: opts.maxMemories,
      dryRun: opts.dryRun ?? false
    },
    maxAttempts: 3
  });

  return task.id;
}
```

---

## `packages/kairos/src/engine/KairosEngine.ts`

```ts
// packages/kairos/src/engine/KairosEngine.ts

import { randomUUID } from "node:crypto";
import type {
  KairosEvent,
  KairosRun,
  KairosScheduleInput,
  KairosTask,
  KairosTaskHandler
} from "../types.js";
import { KairosStore } from "./KairosStore.js";
import {
  calculateNextRunAt,
  isRecurringSchedule
} from "./ScheduleCalculator.js";

export interface KairosEngineOptions {
  workingDir: string;
  tickIntervalMs?: number;
}

export class KairosEngine {
  private readonly handlers = new Map<string, KairosTaskHandler>();
  private readonly listeners = new Set<(event: KairosEvent) => void>();
  private timer: NodeJS.Timeout | null = null;
  private running = false;

  private constructor(
    private readonly opts: Required<KairosEngineOptions>,
    private readonly store: KairosStore
  ) {
    this.registerHandler({
      kind: "noop",
      handle: async (task) => ({
        ok: true,
        message: `No-op task ${task.name} executed`
      })
    });
  }

  static async create(opts: KairosEngineOptions): Promise<KairosEngine> {
    const store = new KairosStore(opts.workingDir);
    await store.init();

    return new KairosEngine(
      {
        workingDir: opts.workingDir,
        tickIntervalMs: opts.tickIntervalMs ?? 60_000
      },
      store
    );
  }

  subscribe(listener: (event: KairosEvent) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  registerHandler(handler: KairosTaskHandler): void {
    this.handlers.set(handler.kind, handler);
  }

  async schedule(input: KairosScheduleInput): Promise<KairosTask> {
    const now = Date.now();

    const task: KairosTask = {
      id: randomUUID(),
      name: input.name,
      kind: input.kind,
      schedule: input.schedule,
      payload: input.payload ?? {},
      status: "scheduled",
      attempts: 0,
      maxAttempts: input.maxAttempts ?? 3,
      createdAt: now,
      updatedAt: now,
      nextRunAt: calculateNextRunAt(input.schedule, now) ?? now
    };

    await this.store.saveTask(task);

    this.emit({
      kind: "task_scheduled",
      task,
      ts: Date.now()
    });

    return task;
  }

  async cancel(taskId: string): Promise<void> {
    const task = await this.store.getTask(taskId);
    if (!task) return;

    task.status = "cancelled";
    task.updatedAt = Date.now();

    await this.store.saveTask(task);

    this.emit({
      kind: "task_cancelled",
      taskId,
      ts: Date.now()
    });
  }

  async listTasks(): Promise<KairosTask[]> {
    return this.store.listTasks();
  }

  async listRuns(taskId?: string): Promise<KairosRun[]> {
    return this.store.listRuns(taskId);
  }

  async runDue(now: number = Date.now()): Promise<KairosRun[]> {
    const tasks = await this.store.listTasks();

    const due = tasks.filter(
      (task) =>
        task.status === "scheduled" &&
        task.nextRunAt !== undefined &&
        task.nextRunAt <= now
    );

    this.emit({
      kind: "tick",
      dueCount: due.length,
      ts: now
    });

    const runs: KairosRun[] = [];

    for (const task of due) {
      runs.push(await this.runTask(task));
    }

    return runs;
  }

  start(): void {
    if (this.running) return;

    this.running = true;

    this.timer = setInterval(() => {
      void this.runDue().catch(() => {
        // errors are represented in task_failed events
      });
    }, this.opts.tickIntervalMs);
  }

  stop(): void {
    this.running = false;

    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  async buildTemporalContext(): Promise<string> {
    const { buildTemporalContext } = await import("./TemporalContext.js");
    return buildTemporalContext({
      tasks: await this.store.listTasks()
    });
  }

  private async runTask(task: KairosTask): Promise<KairosRun> {
    const now = Date.now();

    task.status = "running";
    task.attempts += 1;
    task.lastRunAt = now;
    task.updatedAt = now;

    await this.store.saveTask(task);

    const run: KairosRun = {
      id: randomUUID(),
      taskId: task.id,
      taskName: task.name,
      kind: task.kind,
      status: "running",
      startedAt: now
    };

    await this.store.saveRun(run);

    this.emit({
      kind: "task_started",
      task,
      run,
      ts: Date.now()
    });

    const handler = this.handlers.get(task.kind);

    if (!handler) {
      return this.failTask(task, run, `No handler registered for kind "${task.kind}"`);
    }

    const controller = new AbortController();

    try {
      const output = await handler.handle(task, {
        workingDir: this.opts.workingDir,
        signal: controller.signal,
        emit: (event) => this.emit(event)
      });

      run.status = "completed";
      run.finishedAt = Date.now();
      run.output = output;

      if (isRecurringSchedule(task.schedule)) {
        task.status = "scheduled";
        task.nextRunAt = calculateNextRunAt(task.schedule, Date.now());
      } else {
        task.status = "completed";
        task.nextRunAt = undefined;
      }

      task.updatedAt = Date.now();
      task.lastError = undefined;

      await this.store.saveRun(run);
      await this.store.saveTask(task);

      this.emit({
        kind: "task_completed",
        task,
        run,
        ts: Date.now()
      });

      return run;
    } catch (err) {
      const message = err instanceof Error ? err.message : String(err);
      return this.failTask(task, run, message);
    }
  }

  private async failTask(
    task: KairosTask,
    run: KairosRun,
    error: string
  ): Promise<KairosRun> {
    run.status = "failed";
    run.finishedAt = Date.now();
    run.error = error;

    if (task.attempts < task.maxAttempts) {
      task.status = "scheduled";
      task.nextRunAt = Date.now() + Math.min(task.attempts * 60_000, 15 * 60_000);
    } else {
      task.status = "failed";
      task.nextRunAt = undefined;
    }

    task.lastError = error;
    task.updatedAt = Date.now();

    await this.store.saveRun(run);
    await this.store.saveTask(task);

    this.emit({
      kind: "task_failed",
      task,
      run,
      error,
      ts: Date.now()
    });

    return run;
  }

  private emit(event: KairosEvent): void {
    for (const listener of this.listeners) {
      try {
        listener(event);
      } catch {
        // Listener failures must not break the scheduler.
      }
    }
  }
}
```

---

## `packages/kairos/src/gateway/KairosGatewayAdapter.ts`

```ts
// packages/kairos/src/gateway/KairosGatewayAdapter.ts

import type { KairosEngine } from "../engine/KairosEngine.js";

export class KairosGatewayAdapter {
  constructor(private readonly engine: KairosEngine) {}

  async schedule(input: {
    name: string;
    cron?: string;
    runAt?: number;
    payload: Record<string, unknown>;
  }): Promise<unknown> {
    const kind =
      typeof input.payload.kind === "string"
        ? input.payload.kind
        : "noop";

    return this.engine.schedule({
      name: input.name,
      kind,
      schedule: {
        cron: input.cron,
        runAt: input.runAt
      },
      payload: input.payload
    });
  }

  async list(): Promise<unknown> {
    return this.engine.listTasks();
  }

  async runDue(): Promise<unknown> {
    return this.engine.runDue();
  }
}
```

---

## `packages/kairos/src/index.ts`

```ts
// packages/kairos/src/index.ts

export type {
  KairosEvent,
  KairosRun,
  KairosRunStatus,
  KairosSchedule,
  KairosScheduleInput,
  KairosTask,
  KairosTaskHandler,
  KairosTaskHandlerContext,
  KairosTaskStatus
} from "./types.js";

export { KairosEngine } from "./engine/KairosEngine.js";
export { KairosStore } from "./engine/KairosStore.js";

export {
  calculateNextRunAt,
  isRecurringSchedule
} from "./engine/ScheduleCalculator.js";

export {
  buildTemporalContext
} from "./engine/TemporalContext.js";

export {
  registerAutoDreamTask
} from "./engine/AutoDreamScheduler.js";

export type {
  AutoDreamAdapter,
  RegisterAutoDreamOptions
} from "./engine/AutoDreamScheduler.js";

export { KairosGatewayAdapter } from "./gateway/KairosGatewayAdapter.js";
```

---

# 2) `packages/orchestrator`

## `packages/orchestrator/package.json`

```json
{
  "name": "@locoworker/orchestrator",
  "version": "0.1.0",
  "description": "Multi-agent orchestration engine for LocoWorker",
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
    "lint": "biome lint src"
  },
  "dependencies": {
    "@locoworker/core": "workspace:*"
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

## `packages/orchestrator/tsconfig.build.json`

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

## `packages/orchestrator/src/types.ts`

```ts
// packages/orchestrator/src/types.ts

import type {
  AgentEvent,
  BudgetConfig,
  ConfirmationHandler,
  ModelConfig,
  ToolHandler
} from "@locoworker/core";

export type OrchestrationRunStatus =
  | "planning"
  | "running"
  | "completed"
  | "failed"
  | "aborted";

export type WorkUnitStatus =
  | "pending"
  | "running"
  | "completed"
  | "failed"
  | "skipped";

export type AgentRole =
  | "planner"
  | "implementer"
  | "reviewer"
  | "tester"
  | "researcher"
  | "summarizer";

export interface AgentSpec {
  id: string;
  name: string;
  role: AgentRole;
  systemPrompt?: string;
  modelConfig?: Partial<ModelConfig>;
}

export interface OrchestrationRequest {
  goal: string;
  workingDir: string;
  modelConfig: ModelConfig;
  budget: BudgetConfig;
  maxConcurrency?: number;
  agents?: AgentSpec[];
  files?: string[];
  subtasks?: Array<{
    title: string;
    prompt: string;
    files?: string[];
    dependsOn?: string[];
    agentRole?: AgentRole;
  }>;
  metadata?: Record<string, unknown>;
}

export interface WorkUnit {
  id: string;
  title: string;
  prompt: string;
  files: string[];
  dependsOn: string[];
  agentRole: AgentRole;
  status: WorkUnitStatus;
  assignedAgentId?: string;
  startedAt?: number;
  finishedAt?: number;
  error?: string;
}

export interface WorkUnitResult {
  unitId: string;
  title: string;
  status: "completed" | "failed";
  output: string;
  error?: string;
  events: AgentEvent[];
  startedAt: number;
  finishedAt: number;
}

export interface OrchestrationRun {
  id: string;
  request: OrchestrationRequest;
  status: OrchestrationRunStatus;
  createdAt: number;
  updatedAt: number;
  startedAt?: number;
  finishedAt?: number;
  units: WorkUnit[];
  results: WorkUnitResult[];
  finalSummary?: string;
  error?: string;
}

export type OrchestratorEvent =
  | {
      kind: "orchestration_started";
      runId: string;
      goal: string;
      ts: number;
    }
  | {
      kind: "orchestration_planned";
      runId: string;
      unitCount: number;
      ts: number;
    }
  | {
      kind: "work_unit_started";
      runId: string;
      unit: WorkUnit;
      ts: number;
    }
  | {
      kind: "work_unit_completed";
      runId: string;
      unit: WorkUnit;
      result: WorkUnitResult;
      ts: number;
    }
  | {
      kind: "work_unit_failed";
      runId: string;
      unit: WorkUnit;
      error: string;
      ts: number;
    }
  | {
      kind: "agent_event";
      runId: string;
      unitId: string;
      event: AgentEvent;
      ts: number;
    }
  | {
      kind: "orchestration_completed";
      runId: string;
      finalSummary: string;
      ts: number;
    }
  | {
      kind: "orchestration_failed";
      runId: string;
      error: string;
      ts: number;
    };

export interface OrchestratorEngineOptions {
  workingDir: string;
  tools: Map<string, ToolHandler>;
  modelConfig: ModelConfig;
  budget: BudgetConfig;
  confirmationHandler?: ConfirmationHandler;
  maxConcurrency?: number;
}

export type WorkUnitRunner = (
  unit: WorkUnit,
  run: OrchestrationRun
) => Promise<WorkUnitResult>;
```

---

## `packages/orchestrator/src/engine/FileLockManager.ts`

```ts
// packages/orchestrator/src/engine/FileLockManager.ts

import fs from "node:fs/promises";
import path from "node:path";
import { createHash } from "node:crypto";
import { WorkspaceBoundary } from "@locoworker/core";

export class FileLockError extends Error {
  constructor(
    public readonly file: string,
    public readonly owner: string
  ) {
    super(`File "${file}" is already locked by "${owner}"`);
    this.name = "FileLockError";
  }
}

export interface FileLockLease {
  ownerId: string;
  files: string[];
  release: () => Promise<void>;
}

interface LockRecord {
  ownerId: string;
  file: string;
  createdAt: number;
  expiresAt: number;
}

export class FileLockManager {
  private readonly lockDir: string;
  private readonly boundary: WorkspaceBoundary;

  constructor(
    private readonly workingDir: string,
    private readonly ttlMs: number = 10 * 60 * 1000
  ) {
    this.boundary = new WorkspaceBoundary(workingDir);
    this.lockDir = path.join(workingDir, ".locoworker", "locks");
  }

  async acquire(files: string[], ownerId: string): Promise<FileLockLease> {
    await fs.mkdir(this.lockDir, { recursive: true });

    const resolved = [...new Set(files.map((file) => this.boundary.resolve(file)))];

    const acquired: string[] = [];

    try {
      for (const file of resolved) {
        await this.acquireOne(file, ownerId);
        acquired.push(file);
      }
    } catch (err) {
      await this.release(acquired, ownerId);
      throw err;
    }

    return {
      ownerId,
      files: resolved,
      release: async () => {
        await this.release(resolved, ownerId);
      }
    };
  }

  async release(files: string[], ownerId: string): Promise<void> {
    for (const file of files) {
      const lockPath = this.lockPath(file);

      try {
        const raw = await fs.readFile(lockPath, "utf8");
        const record = JSON.parse(raw) as LockRecord;

        if (record.ownerId === ownerId) {
          await fs.unlink(lockPath);
        }
      } catch {
        // Missing or corrupt lock is safe to ignore during release.
      }
    }
  }

  async isLocked(file: string): Promise<boolean> {
    const resolved = this.boundary.resolve(file);
    const lockPath = this.lockPath(resolved);

    try {
      const raw = await fs.readFile(lockPath, "utf8");
      const record = JSON.parse(raw) as LockRecord;

      if (record.expiresAt < Date.now()) {
        await fs.unlink(lockPath);
        return false;
      }

      return true;
    } catch {
      return false;
    }
  }

  private async acquireOne(file: string, ownerId: string): Promise<void> {
    const lockPath = this.lockPath(file);

    try {
      const raw = await fs.readFile(lockPath, "utf8");
      const existing = JSON.parse(raw) as LockRecord;

      if (existing.expiresAt >= Date.now()) {
        throw new FileLockError(file, existing.ownerId);
      }

      await fs.unlink(lockPath);
    } catch (err) {
      if (err instanceof FileLockError) throw err;
      // Missing/corrupt/expired lock can be overwritten.
    }

    const record: LockRecord = {
      ownerId,
      file,
      createdAt: Date.now(),
      expiresAt: Date.now() + this.ttlMs
    };

    await fs.writeFile(lockPath, JSON.stringify(record, null, 2), {
      encoding: "utf8",
      flag: "wx"
    });
  }

  private lockPath(file: string): string {
    const hash = createHash("sha256").update(file).digest("hex");
    return path.join(this.lockDir, `${hash}.lock.json`);
  }
}
```

---

## `packages/orchestrator/src/engine/DelegationPlanner.ts`

```ts
// packages/orchestrator/src/engine/DelegationPlanner.ts

import { randomUUID } from "node:crypto";
import type {
  AgentRole,
  OrchestrationRequest,
  WorkUnit
} from "../types.js";

export class DelegationPlanner {
  plan(request: OrchestrationRequest): WorkUnit[] {
    if (request.subtasks && request.subtasks.length > 0) {
      return request.subtasks.map((task, index) => ({
        id: `unit-${index + 1}-${randomUUID().slice(0, 8)}`,
        title: task.title,
        prompt: task.prompt,
        files: task.files ?? request.files ?? [],
        dependsOn: task.dependsOn ?? [],
        agentRole: task.agentRole ?? inferRole(task.prompt),
        status: "pending"
      }));
    }

    return this.defaultPlan(request);
  }

  private defaultPlan(request: OrchestrationRequest): WorkUnit[] {
    const files = request.files ?? [];

    return [
      {
        id: `unit-1-${randomUUID().slice(0, 8)}`,
        title: "Research and inspect",
        prompt:
          `Goal: ${request.goal}\n\n` +
          "Inspect the workspace and gather relevant context. " +
          "Do not modify files in this phase. Return concise findings.",
        files: [],
        dependsOn: [],
        agentRole: "researcher",
        status: "pending"
      },
      {
        id: `unit-2-${randomUUID().slice(0, 8)}`,
        title: "Implement",
        prompt:
          `Goal: ${request.goal}\n\n` +
          "Use the research findings from prior work. Implement the requested change. " +
          "Prefer minimal, targeted edits.",
        files,
        dependsOn: [],
        agentRole: "implementer",
        status: "pending"
      },
      {
        id: `unit-3-${randomUUID().slice(0, 8)}`,
        title: "Review and test",
        prompt:
          `Goal: ${request.goal}\n\n` +
          "Review the implementation, run relevant checks if available, " +
          "and report remaining risks or follow-up tasks.",
        files,
        dependsOn: [],
        agentRole: "tester",
        status: "pending"
      }
    ];
  }
}

function inferRole(prompt: string): AgentRole {
  const lower = prompt.toLowerCase();

  if (lower.includes("test") || lower.includes("verify")) return "tester";
  if (lower.includes("review") || lower.includes("audit")) return "reviewer";
  if (lower.includes("research") || lower.includes("inspect")) return "researcher";
  if (lower.includes("summarize") || lower.includes("summary")) return "summarizer";

  return "implementer";
}
```

---

## `packages/orchestrator/src/engine/ResultAggregator.ts`

```ts
// packages/orchestrator/src/engine/ResultAggregator.ts

import type { OrchestrationRun, WorkUnitResult } from "../types.js";

export class ResultAggregator {
  aggregate(run: OrchestrationRun, results: WorkUnitResult[]): string {
    const completed = results.filter((r) => r.status === "completed");
    const failed = results.filter((r) => r.status === "failed");

    const lines: string[] = [];

    lines.push(`# Orchestration Summary`);
    lines.push("");
    lines.push(`Goal: ${run.request.goal}`);
    lines.push(`Status: ${failed.length > 0 ? "completed_with_failures" : "completed"}`);
    lines.push(`Completed units: ${completed.length}`);
    lines.push(`Failed units: ${failed.length}`);

    if (completed.length > 0) {
      lines.push("");
      lines.push("## Completed Work");
      for (const result of completed) {
        lines.push("");
        lines.push(`### ${result.title}`);
        lines.push(result.output.trim() || "_No output_");
      }
    }

    if (failed.length > 0) {
      lines.push("");
      lines.push("## Failed Work");
      for (const result of failed) {
        lines.push("");
        lines.push(`### ${result.title}`);
        lines.push(result.error ?? "Unknown error");
      }
    }

    return lines.join("\n");
  }
}
```

---

## `packages/orchestrator/src/engine/OrchestratorStore.ts`

```ts
// packages/orchestrator/src/engine/OrchestratorStore.ts

import fs from "node:fs/promises";
import path from "node:path";
import type { OrchestrationRun } from "../types.js";

interface OrchestratorDb {
  runs: OrchestrationRun[];
}

const EMPTY_DB: OrchestratorDb = {
  runs: []
};

export class OrchestratorStore {
  private readonly dir: string;
  private readonly file: string;

  constructor(private readonly workingDir: string) {
    this.dir = path.join(workingDir, ".locoworker", "orchestrator");
    this.file = path.join(this.dir, "orchestrator.json");
  }

  async init(): Promise<void> {
    await fs.mkdir(this.dir, { recursive: true });

    try {
      await fs.access(this.file);
    } catch {
      await this.writeDb(EMPTY_DB);
    }
  }

  async saveRun(run: OrchestrationRun): Promise<void> {
    const db = await this.readDb();
    const index = db.runs.findIndex((r) => r.id === run.id);

    if (index === -1) {
      db.runs.push(run);
    } else {
      db.runs[index] = run;
    }

    await this.writeDb(db);
  }

  async getRun(runId: string): Promise<OrchestrationRun | undefined> {
    const db = await this.readDb();
    return db.runs.find((r) => r.id === runId);
  }

  async listRuns(): Promise<OrchestrationRun[]> {
    const db = await this.readDb();
    return db.runs.sort((a, b) => b.createdAt - a.createdAt);
  }

  private async readDb(): Promise<OrchestratorDb> {
    await this.initIfNeeded();

    try {
      const raw = await fs.readFile(this.file, "utf8");
      const parsed = JSON.parse(raw);

      return {
        runs: Array.isArray(parsed.runs) ? parsed.runs : []
      };
    } catch {
      return { ...EMPTY_DB };
    }
  }

  private async writeDb(db: OrchestratorDb): Promise<void> {
    await fs.mkdir(this.dir, { recursive: true });
    await fs.writeFile(this.file, JSON.stringify(db, null, 2), "utf8");
  }

  private async initIfNeeded(): Promise<void> {
    try {
      await fs.access(this.file);
    } catch {
      await this.init();
    }
  }
}
```

---

## `packages/orchestrator/src/engine/AgentSupervisor.ts`

```ts
// packages/orchestrator/src/engine/AgentSupervisor.ts

import {
  AgentEventEmitter,
  queryLoop,
  SessionManager,
  type AgentContext,
  type AgentEvent,
  type BudgetConfig,
  type ConfirmationHandler,
  type ModelConfig,
  type ToolHandler
} from "@locoworker/core";
import type {
  OrchestrationRun,
  WorkUnit,
  WorkUnitResult
} from "../types.js";

export interface AgentSupervisorOptions {
  workingDir: string;
  tools: Map<string, ToolHandler>;
  modelConfig: ModelConfig;
  budget: BudgetConfig;
  confirmationHandler?: ConfirmationHandler;
  onAgentEvent?: (unit: WorkUnit, event: AgentEvent) => void;
}

export class AgentSupervisor {
  constructor(private readonly opts: AgentSupervisorOptions) {}

  async runWorkUnit(
    unit: WorkUnit,
    run: OrchestrationRun
  ): Promise<WorkUnitResult> {
    const startedAt = Date.now();
    const manager = new SessionManager(this.opts.workingDir);

    const session = await manager.create({
      modelConfig: {
        ...this.opts.modelConfig
      },
      budget: {
        ...this.opts.budget
      }
    });

    const events: AgentEvent[] = [];
    let finalOutput = "";

    const emitter = new AgentEventEmitter();

    const unsubscribe = emitter.subscribe((event) => {
      events.push(event);
      this.opts.onAgentEvent?.(unit, event);

      if (event.kind === "loop_complete") {
        finalOutput = event.finalResponse;
      }

      if (event.kind === "turn_complete" && !finalOutput) {
        finalOutput = event.assistantText;
      }
    });

    const prompt =
      `You are the ${unit.agentRole} agent for an orchestrated LocoWorker run.\n\n` +
      `Overall goal:\n${run.request.goal}\n\n` +
      `Your work unit:\n${unit.title}\n\n` +
      `Instructions:\n${unit.prompt}\n\n` +
      `Files reserved for this unit:\n${unit.files.length > 0 ? unit.files.join("\n") : "None"}\n\n` +
      "Return a concise report of what you did, what changed, and any risks.";

    const ctx: AgentContext = {
      session,
      tools: this.opts.tools,
      emit: (event) => emitter.emit(event),
      confirmationHandler: this.opts.confirmationHandler,
      onCheckpoint: async () => {
        await manager.checkpoint(session);
      }
    };

    try {
      for await (const _event of queryLoop(prompt, ctx)) {
        // Drain generator. Events are emitted through ctx.emit.
      }

      await manager.checkpoint(session);

      return {
        unitId: unit.id,
        title: unit.title,
        status: "completed",
        output: finalOutput || "Completed without final text.",
        events,
        startedAt,
        finishedAt: Date.now()
      };
    } catch (err) {
      const error = err instanceof Error ? err.message : String(err);

      return {
        unitId: unit.id,
        title: unit.title,
        status: "failed",
        output: "",
        error,
        events,
        startedAt,
        finishedAt: Date.now()
      };
    } finally {
      unsubscribe();
    }
  }
}
```

---

## `packages/orchestrator/src/engine/OrchestratorEngine.ts`

```ts
// packages/orchestrator/src/engine/OrchestratorEngine.ts

import { randomUUID } from "node:crypto";
import type {
  OrchestrationRequest,
  OrchestrationRun,
  OrchestratorEngineOptions,
  OrchestratorEvent,
  WorkUnit,
  WorkUnitResult,
  WorkUnitRunner
} from "../types.js";
import { DelegationPlanner } from "./DelegationPlanner.js";
import { AgentSupervisor } from "./AgentSupervisor.js";
import { ResultAggregator } from "./ResultAggregator.js";
import { FileLockManager } from "./FileLockManager.js";
import { OrchestratorStore } from "./OrchestratorStore.js";

export class OrchestratorEngine {
  private readonly listeners = new Set<(event: OrchestratorEvent) => void>();
  private readonly planner = new DelegationPlanner();
  private readonly aggregator = new ResultAggregator();
  private readonly locks: FileLockManager;
  private readonly store: OrchestratorStore;
  private runner?: WorkUnitRunner;
  private initialized = false;

  constructor(private readonly opts: OrchestratorEngineOptions) {
    this.locks = new FileLockManager(opts.workingDir);
    this.store = new OrchestratorStore(opts.workingDir);
  }

  async init(): Promise<void> {
    if (this.initialized) return;
    await this.store.init();
    this.initialized = true;
  }

  subscribe(listener: (event: OrchestratorEvent) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  /**
   * Test hook or advanced integration hook.
   * If absent, AgentSupervisor/queryLoop is used.
   */
  setRunner(runner: WorkUnitRunner): void {
    this.runner = runner;
  }

  async start(request: OrchestrationRequest): Promise<OrchestrationRun> {
    await this.init();

    const now = Date.now();

    const run: OrchestrationRun = {
      id: randomUUID(),
      request,
      status: "planning",
      createdAt: now,
      updatedAt: now,
      units: [],
      results: []
    };

    await this.store.saveRun(run);

    this.emit({
      kind: "orchestration_started",
      runId: run.id,
      goal: request.goal,
      ts: Date.now()
    });

    void this.executeRun(run).catch(async (err) => {
      const error = err instanceof Error ? err.message : String(err);
      run.status = "failed";
      run.error = error;
      run.finishedAt = Date.now();
      run.updatedAt = Date.now();
      await this.store.saveRun(run);

      this.emit({
        kind: "orchestration_failed",
        runId: run.id,
        error,
        ts: Date.now()
      });
    });

    return run;
  }

  async runToCompletion(request: OrchestrationRequest): Promise<OrchestrationRun> {
    await this.init();

    const now = Date.now();

    const run: OrchestrationRun = {
      id: randomUUID(),
      request,
      status: "planning",
      createdAt: now,
      updatedAt: now,
      units: [],
      results: []
    };

    await this.store.saveRun(run);

    this.emit({
      kind: "orchestration_started",
      runId: run.id,
      goal: request.goal,
      ts: Date.now()
    });

    await this.executeRun(run);
    return run;
  }

  async getRun(runId: string): Promise<OrchestrationRun | undefined> {
    await this.init();
    return this.store.getRun(runId);
  }

  async listRuns(): Promise<OrchestrationRun[]> {
    await this.init();
    return this.store.listRuns();
  }

  private async executeRun(run: OrchestrationRun): Promise<void> {
    run.units = this.planner.plan(run.request);
    run.status = "running";
    run.startedAt = Date.now();
    run.updatedAt = Date.now();

    await this.store.saveRun(run);

    this.emit({
      kind: "orchestration_planned",
      runId: run.id,
      unitCount: run.units.length,
      ts: Date.now()
    });

    const maxConcurrency =
      run.request.maxConcurrency ??
      this.opts.maxConcurrency ??
      2;

    const active = new Map<string, Promise<WorkUnitResult>>();

    while (true) {
      const completedCount = run.units.filter(
        (u) => u.status === "completed" || u.status === "failed" || u.status === "skipped"
      ).length;

      if (completedCount === run.units.length) {
        break;
      }

      const availableSlots = Math.max(0, maxConcurrency - active.size);

      if (availableSlots > 0) {
        const ready = this.findReadyUnits(run.units).slice(0, availableSlots);

        for (const unit of ready) {
          unit.status = "running";
          unit.startedAt = Date.now();
          run.updatedAt = Date.now();

          await this.store.saveRun(run);

          this.emit({
            kind: "work_unit_started",
            runId: run.id,
            unit,
            ts: Date.now()
          });

          const promise = this.executeUnit(unit, run)
            .then((result) => {
              active.delete(unit.id);
              return result;
            })
            .catch((err) => {
              active.delete(unit.id);

              const error = err instanceof Error ? err.message : String(err);

              return {
                unitId: unit.id,
                title: unit.title,
                status: "failed" as const,
                output: "",
                error,
                events: [],
                startedAt: unit.startedAt ?? Date.now(),
                finishedAt: Date.now()
              };
            });

          active.set(unit.id, promise);
        }
      }

      if (active.size === 0) {
        const pending = run.units.filter((u) => u.status === "pending");

        for (const unit of pending) {
          unit.status = "skipped";
          unit.error = "Dependencies failed or dependency graph is blocked.";
        }

        break;
      }

      const result = await Promise.race(active.values());
      const unit = run.units.find((u) => u.id === result.unitId);

      if (!unit) continue;

      unit.status = result.status === "completed" ? "completed" : "failed";
      unit.finishedAt = result.finishedAt;
      unit.error = result.error;
      run.results.push(result);
      run.updatedAt = Date.now();

      await this.store.saveRun(run);

      if (result.status === "completed") {
        this.emit({
          kind: "work_unit_completed",
          runId: run.id,
          unit,
          result,
          ts: Date.now()
        });
      } else {
        this.emit({
          kind: "work_unit_failed",
          runId: run.id,
          unit,
          error: result.error ?? "Unknown error",
          ts: Date.now()
        });
      }
    }

    run.finalSummary = this.aggregator.aggregate(run, run.results);
    run.status = run.results.some((r) => r.status === "failed")
      ? "failed"
      : "completed";
    run.finishedAt = Date.now();
    run.updatedAt = Date.now();

    await this.store.saveRun(run);

    if (run.status === "completed") {
      this.emit({
        kind: "orchestration_completed",
        runId: run.id,
        finalSummary: run.finalSummary,
        ts: Date.now()
      });
    } else {
      this.emit({
        kind: "orchestration_failed",
        runId: run.id,
        error: "One or more work units failed.",
        ts: Date.now()
      });
    }
  }

  private async executeUnit(
    unit: WorkUnit,
    run: OrchestrationRun
  ): Promise<WorkUnitResult> {
    const ownerId = `${run.id}:${unit.id}`;
    const lease = unit.files.length > 0
      ? await this.locks.acquire(unit.files, ownerId)
      : undefined;

    try {
      if (this.runner) {
        return await this.runner(unit, run);
      }

      const supervisor = new AgentSupervisor({
        workingDir: run.request.workingDir,
        tools: this.opts.tools,
        modelConfig: {
          ...this.opts.modelConfig,
          ...run.request.modelConfig
        },
        budget: {
          ...this.opts.budget,
          ...run.request.budget
        },
        confirmationHandler: this.opts.confirmationHandler,
        onAgentEvent: (u, event) => {
          this.emit({
            kind: "agent_event",
            runId: run.id,
            unitId: u.id,
            event,
            ts: Date.now()
          });
        }
      });

      return await supervisor.runWorkUnit(unit, run);
    } finally {
      await lease?.release();
    }
  }

  private findReadyUnits(units: WorkUnit[]): WorkUnit[] {
    const completed = new Set(
      units.filter((u) => u.status === "completed").map((u) => u.id)
    );

    return units.filter((unit) => {
      if (unit.status !== "pending") return false;
      return unit.dependsOn.every((id) => completed.has(id));
    });
  }

  private emit(event: OrchestratorEvent): void {
    for (const listener of this.listeners) {
      try {
        listener(event);
      } catch {
        // Listener failure must not interrupt orchestration.
      }
    }
  }
}
```

---

## `packages/orchestrator/src/gateway/OrchestratorGatewayAdapter.ts`

```ts
// packages/orchestrator/src/gateway/OrchestratorGatewayAdapter.ts

import type { BudgetConfig, ModelConfig } from "@locoworker/core";
import type { OrchestratorEngine } from "../engine/OrchestratorEngine.js";

export class OrchestratorGatewayAdapter {
  constructor(
    private readonly engine: OrchestratorEngine,
    private readonly defaults: {
      workingDir: string;
      modelConfig: ModelConfig;
      budget: BudgetConfig;
    }
  ) {}

  async start(input: {
    goal: string;
    files?: string[];
    maxConcurrency?: number;
    subtasks?: Array<{
      title: string;
      prompt: string;
      files?: string[];
      dependsOn?: string[];
    }>;
    metadata?: Record<string, unknown>;
  }): Promise<unknown> {
    return this.engine.start({
      goal: input.goal,
      workingDir: this.defaults.workingDir,
      modelConfig: this.defaults.modelConfig,
      budget: this.defaults.budget,
      files: input.files,
      maxConcurrency: input.maxConcurrency,
      subtasks: input.subtasks,
      metadata: input.metadata
    });
  }

  async list(): Promise<unknown> {
    return this.engine.listRuns();
  }

  async get(id: string): Promise<unknown> {
    return this.engine.getRun(id);
  }
}
```

---

## `packages/orchestrator/src/index.ts`

```ts
// packages/orchestrator/src/index.ts

export type {
  AgentRole,
  AgentSpec,
  OrchestrationRequest,
  OrchestrationRun,
  OrchestrationRunStatus,
  OrchestratorEngineOptions,
  OrchestratorEvent,
  WorkUnit,
  WorkUnitResult,
  WorkUnitRunner,
  WorkUnitStatus
} from "./types.js";

export { OrchestratorEngine } from "./engine/OrchestratorEngine.js";
export { OrchestratorStore } from "./engine/OrchestratorStore.js";
export { DelegationPlanner } from "./engine/DelegationPlanner.js";
export { AgentSupervisor } from "./engine/AgentSupervisor.js";
export { ResultAggregator } from "./engine/ResultAggregator.js";
export {
  FileLockManager,
  FileLockError
} from "./engine/FileLockManager.js";

export { OrchestratorGatewayAdapter } from "./gateway/OrchestratorGatewayAdapter.js";
```

---

# 3) Gateway patches

## `packages/gateway/package.json`

Patch dependencies:

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
    "@locoworker/kairos": "workspace:*",
    "@locoworker/orchestrator": "workspace:*",
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

export interface OrchestratorAdapter {
  start(input: {
    goal: string;
    files?: string[];
    maxConcurrency?: number;
    subtasks?: Array<{
      title: string;
      prompt: string;
      files?: string[];
      dependsOn?: string[];
    }>;
    metadata?: Record<string, unknown>;
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
  orchestrator?: OrchestratorAdapter;
}
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
  kind: z.string().optional(),
  cron: z.string().optional(),
  runAt: z.number().optional(),
  payload: z.record(z.unknown()).optional().default({})
});

export const ResearchRunBodySchema = z.object({
  goal: z.string().min(1),
  programPath: z.string().optional(),
  maxIterations: z.number().int().positive().max(100).optional().default(10)
});

export const OrchestratorRunBodySchema = z.object({
  goal: z.string().min(1),
  files: z.array(z.string()).optional(),
  maxConcurrency: z.number().int().positive().max(16).optional(),
  subtasks: z
    .array(
      z.object({
        title: z.string().min(1),
        prompt: z.string().min(1),
        files: z.array(z.string()).optional(),
        dependsOn: z.array(z.string()).optional()
      })
    )
    .optional(),
  metadata: z.record(z.unknown()).optional().default({})
});
```

---

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

    const adapters: CapabilityAdapters = {
      ...(input?.adapters ?? {})
    };

    // Auto-wire Kairos if caller did not inject one.
    if (!adapters.kairos) {
      try {
        const { KairosEngine, KairosGatewayAdapter } = await import("@locoworker/kairos");
        const kairos = await KairosEngine.create({
          workingDir: config.defaultWorkingDir,
          tickIntervalMs: 60_000
        });

        // Do not auto-start background loop in GatewayRuntime by default.
        // `/v1/kairos/run-due` is deterministic and test-friendly.
        adapters.kairos = new KairosGatewayAdapter(kairos);
      } catch {
        // Gateway must remain bootable even if package build order is incomplete.
      }
    }

    // Auto-wire Orchestrator if caller did not inject one.
    if (!adapters.orchestrator) {
      try {
        const {
          OrchestratorEngine,
          OrchestratorGatewayAdapter
        } = await import("@locoworker/orchestrator");

        const orchestrator = new OrchestratorEngine({
          workingDir: config.defaultWorkingDir,
          tools,
          modelConfig: config.modelConfig,
          budget: config.budget,
          maxConcurrency: 2
        });

        await orchestrator.init();

        adapters.orchestrator = new OrchestratorGatewayAdapter(
          orchestrator,
          {
            workingDir: config.defaultWorkingDir,
            modelConfig: config.modelConfig,
            budget: config.budget
          }
        );
      } catch {
        // Gateway must remain bootable even if package build order is incomplete.
      }
    }

    return new GatewayRuntime(config, tools, adapters);
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
        research: Boolean(this.adapters.research),
        orchestrator: Boolean(this.adapters.orchestrator)
      }
    };
  }
}
```

---

## `packages/gateway/src/routes/orchestrator.routes.ts`

```ts
// packages/gateway/src/routes/orchestrator.routes.ts

import type { FastifyInstance } from "fastify";
import type { GatewayRuntime } from "../runtime/GatewayRuntime.js";
import { GatewayError } from "../errors/GatewayError.js";
import { OrchestratorRunBodySchema } from "../schemas/capability.schemas.js";
import { IdParamSchema, parseOrThrow } from "../schemas/common.schemas.js";

export async function registerOrchestratorRoutes(
  app: FastifyInstance,
  runtime: GatewayRuntime
): Promise<void> {
  app.post("/v1/orchestrator/runs", async (req) => {
    if (!runtime.adapters.orchestrator) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Orchestrator adapter is not configured"
      );
    }

    const body = parseOrThrow(OrchestratorRunBodySchema, req.body);
    const run = await runtime.adapters.orchestrator.start(body);

    return {
      ok: true,
      run
    };
  });

  app.get("/v1/orchestrator/runs", async () => {
    if (!runtime.adapters.orchestrator) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Orchestrator adapter is not configured"
      );
    }

    const runs = await runtime.adapters.orchestrator.list();

    return {
      ok: true,
      runs
    };
  });

  app.get("/v1/orchestrator/runs/:id", async (req) => {
    if (!runtime.adapters.orchestrator) {
      throw new GatewayError(
        501,
        "CAPABILITY_NOT_CONFIGURED",
        "Orchestrator adapter is not configured"
      );
    }

    const params = parseOrThrow(IdParamSchema, req.params);
    const run = await runtime.adapters.orchestrator.get(params.id);

    return {
      ok: true,
      run
    };
  });
}
```

---

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
import { registerOrchestratorRoutes } from "./orchestrator.routes.js";

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
  await registerOrchestratorRoutes(app, runtime);
  await registerAdminRoutes(app, runtime);
}
```

---

# 4) Tests

## `tests/unit/kairos/ScheduleCalculator.test.ts`

```ts
import { describe, expect, it } from "vitest";
import {
  calculateNextRunAt,
  isRecurringSchedule
} from "../../../packages/kairos/src/engine/ScheduleCalculator.js";

describe("ScheduleCalculator", () => {
  it("returns future runAt for one-shot schedule", () => {
    const now = Date.now();
    const next = calculateNextRunAt({ runAt: now + 1000 }, now);

    expect(next).toBe(now + 1000);
  });

  it("returns interval next run", () => {
    const now = 1_000_000;
    const next = calculateNextRunAt({ intervalMs: 5000 }, now);

    expect(next).toBe(now + 5000);
  });

  it("supports @hourly", () => {
    const next = calculateNextRunAt({ cron: "@hourly" }, 1);

    expect(next).toBeGreaterThan(1);
  });

  it("marks interval and cron as recurring", () => {
    expect(isRecurringSchedule({ intervalMs: 1000 })).toBe(true);
    expect(isRecurringSchedule({ cron: "@daily" })).toBe(true);
    expect(isRecurringSchedule({ runAt: Date.now() + 1000 })).toBe(false);
  });
});
```

---

## `tests/unit/kairos/KairosEngine.test.ts`

```ts
import { describe, expect, it } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { KairosEngine } from "../../../packages/kairos/src/engine/KairosEngine.js";

describe("KairosEngine", () => {
  it("schedules and runs a due task", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "kairos-"));
    const engine = await KairosEngine.create({ workingDir: tmp });

    let ran = false;

    engine.registerHandler({
      kind: "test.task",
      handle: async () => {
        ran = true;
        return { ok: true };
      }
    });

    await engine.schedule({
      name: "test",
      kind: "test.task",
      schedule: { runAt: Date.now() - 1 },
      payload: {}
    });

    const runs = await engine.runDue();

    expect(runs).toHaveLength(1);
    expect(runs[0].status).toBe("completed");
    expect(ran).toBe(true);
  });

  it("records failed task when no handler exists", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "kairos-"));
    const engine = await KairosEngine.create({ workingDir: tmp });

    await engine.schedule({
      name: "missing",
      kind: "missing.handler",
      schedule: { runAt: Date.now() - 1 },
      payload: {},
      maxAttempts: 1
    });

    const runs = await engine.runDue();

    expect(runs[0].status).toBe("failed");
  });
});
```

---

## `tests/unit/orchestrator/FileLockManager.test.ts`

```ts
import { describe, expect, it } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import {
  FileLockError,
  FileLockManager
} from "../../../packages/orchestrator/src/engine/FileLockManager.js";

describe("FileLockManager", () => {
  it("acquires and releases file locks", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "locks-"));
    const locks = new FileLockManager(tmp);

    const lease = await locks.acquire(["src/a.ts"], "owner-1");

    expect(await locks.isLocked("src/a.ts")).toBe(true);

    await lease.release();

    expect(await locks.isLocked("src/a.ts")).toBe(false);
  });

  it("rejects competing lock owner", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "locks-"));
    const locks = new FileLockManager(tmp);

    await locks.acquire(["src/a.ts"], "owner-1");

    await expect(
      locks.acquire(["src/a.ts"], "owner-2")
    ).rejects.toThrow(FileLockError);
  });

  it("blocks path escape attempts", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "locks-"));
    const locks = new FileLockManager(tmp);

    await expect(
      locks.acquire(["../../escape.txt"], "owner-1")
    ).rejects.toThrow();
  });
});
```

---

## `tests/unit/orchestrator/DelegationPlanner.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { DelegationPlanner } from "../../../packages/orchestrator/src/engine/DelegationPlanner.js";

describe("DelegationPlanner", () => {
  it("uses provided subtasks", () => {
    const planner = new DelegationPlanner();

    const units = planner.plan({
      goal: "Ship feature",
      workingDir: "/tmp",
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: {
        maxInputTokens: 1000,
        softLimitTokens: 800,
        hardLimitTokens: 900,
        compactionMode: "auto"
      },
      subtasks: [
        {
          title: "Inspect",
          prompt: "Research the code"
        }
      ]
    });

    expect(units).toHaveLength(1);
    expect(units[0].title).toBe("Inspect");
  });

  it("creates default plan when no subtasks provided", () => {
    const planner = new DelegationPlanner();

    const units = planner.plan({
      goal: "Improve docs",
      workingDir: "/tmp",
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: {
        maxInputTokens: 1000,
        softLimitTokens: 800,
        hardLimitTokens: 900,
        compactionMode: "auto"
      }
    });

    expect(units.length).toBeGreaterThanOrEqual(3);
  });
});
```

---

## `tests/unit/orchestrator/OrchestratorEngine.test.ts`

```ts
import { describe, expect, it, vi } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { OrchestratorEngine } from "../../../packages/orchestrator/src/engine/OrchestratorEngine.js";

function makeBudget() {
  return {
    maxInputTokens: 1000,
    softLimitTokens: 800,
    hardLimitTokens: 900,
    compactionMode: "auto" as const
  };
}

describe("OrchestratorEngine", () => {
  it("runs provided subtasks to completion with injected runner", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "orch-"));

    const engine = new OrchestratorEngine({
      workingDir: tmp,
      tools: new Map(),
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: makeBudget(),
      maxConcurrency: 2
    });

    engine.setRunner(async (unit) => ({
      unitId: unit.id,
      title: unit.title,
      status: "completed",
      output: `done ${unit.title}`,
      events: [],
      startedAt: Date.now(),
      finishedAt: Date.now()
    }));

    const run = await engine.runToCompletion({
      goal: "Test orchestration",
      workingDir: tmp,
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: makeBudget(),
      subtasks: [
        {
          title: "A",
          prompt: "Do A"
        },
        {
          title: "B",
          prompt: "Do B"
        }
      ]
    });

    expect(run.status).toBe("completed");
    expect(run.results).toHaveLength(2);
    expect(run.finalSummary).toContain("Orchestration Summary");
  });

  it("emits lifecycle events", async () => {
    const tmp = fs.mkdtempSync(path.join(os.tmpdir(), "orch-"));

    const engine = new OrchestratorEngine({
      workingDir: tmp,
      tools: new Map(),
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: makeBudget()
    });

    const events: string[] = [];

    engine.subscribe((event) => {
      events.push(event.kind);
    });

    engine.setRunner(async (unit) => ({
      unitId: unit.id,
      title: unit.title,
      status: "completed",
      output: "ok",
      events: [],
      startedAt: Date.now(),
      finishedAt: Date.now()
    }));

    await engine.runToCompletion({
      goal: "Lifecycle",
      workingDir: tmp,
      modelConfig: {
        provider: "mock",
        model: "mock"
      },
      budget: makeBudget(),
      subtasks: [
        {
          title: "Only",
          prompt: "Do it"
        }
      ]
    });

    expect(events).toContain("orchestration_started");
    expect(events).toContain("orchestration_planned");
    expect(events).toContain("work_unit_started");
    expect(events).toContain("work_unit_completed");
    expect(events).toContain("orchestration_completed");
  });
});
```

---

# 5) Manual validation after Pass 22

## Kairos through Gateway

Start Gateway:

```bash
pnpm --filter @locoworker/app-gateway dev
```

Schedule a no-op task due immediately:

```bash
curl -s -X POST http://127.0.0.1:8787/v1/kairos/tasks \
  -H 'content-type: application/json' \
  -d '{
    "name":"test noop",
    "runAt": 1,
    "payload": { "kind": "noop" }
  }'
```

Run due tasks:

```bash
curl -s -X POST http://127.0.0.1:8787/v1/kairos/run-due
```

List tasks:

```bash
curl -s http://127.0.0.1:8787/v1/kairos/tasks
```

---

## Orchestrator through Gateway

Start an orchestration run:

```bash
curl -s -X POST http://127.0.0.1:8787/v1/orchestrator/runs \
  -H 'content-type: application/json' \
  -d '{
    "goal":"Inspect the project and produce a short improvement plan",
    "maxConcurrency": 2,
    "subtasks": [
      {
        "title": "Inspect docs",
        "prompt": "Read the main docs and summarize the architecture."
      },
      {
        "title": "Find risks",
        "prompt": "Identify obvious gaps, risks, and missing tests."
      }
    ]
  }'
```

List orchestration runs:

```bash
curl -s http://127.0.0.1:8787/v1/orchestrator/runs
```

Fetch a run:

```bash
curl -s http://127.0.0.1:8787/v1/orchestrator/runs/<RUN_ID>
```

---

# 6) Pass 22 completion checklist

| Gate | Status |
|---|---|
| Kairos has durable task persistence | ✅ |
| Kairos can schedule one-shot tasks | ✅ |
| Kairos can schedule recurring interval/cronish tasks | ✅ |
| Kairos can run due tasks deterministically | ✅ |
| Kairos emits task lifecycle events | ✅ |
| Kairos supports task handler registry | ✅ |
| AutoDream registration hook exists | ✅ |
| Temporal context generation exists | ✅ |
| Gateway Kairos adapter works | ✅ |
| Orchestrator can plan work units | ✅ |
| Orchestrator can run work units with bounded concurrency | ✅ |
| Orchestrator uses workspace-safe file locks | ✅ |
| Orchestrator can execute units through `queryLoop` | ✅ |
| Orchestrator supports injected runner for tests | ✅ |
| Orchestrator persists runs/results | ✅ |
| Orchestrator aggregates final summary | ✅ |
| Gateway exposes `/v1/orchestrator/runs` | ✅ |
| Gateway auto-wires Kairos + Orchestrator where packages exist | ✅ |
| Unit tests cover scheduler, engine, locks, planner, orchestrator loop | ✅ |

---

# 7) What Pass 22 unlocks

```txt
Pass 23 Dashboard
  ├─ Can show Kairos scheduled tasks and run-due actions.
  ├─ Can show Orchestrator runs/results.
  └─ Can use Gateway-created sessions/runs/streams from Pass 20.

Pass 24 Desktop
  ├─ Can embed Gateway with Kairos/Orchestrator adapters.
  ├─ Can expose desktop IPC for scheduled tasks and orchestration.
  └─ Can store provider keys securely in keychain.
```

**Pass 22 is complete. Proceed with Pass 23 when ready.**
