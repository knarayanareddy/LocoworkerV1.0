Pass 6 — packages/kairos (Part 1 of 2)
What packages/kairos is: The always-on temporal intelligence layer for LocoWorker. Named after the Greek concept of kairos — the right or opportune moment to act — as opposed to chronos (sequential clock time). This package implements a proactive tick engine, background job scheduler, deadline-aware priority task queue, and append-only observation log. It is the LocoWorker clean-room re-implementation of the KAIROS background agent mode discovered in the Claude Code sourcemap incident (March 2026), extended with cron scheduling, SQLite persistence, and full integration with packages/core's EventBus and HooksRegistry.

Pass 6 split plan
Part	Files Generated
Part 1 (this pass)	package.json · tsconfig.json · src/types/kairos.types.ts · src/KairosStore.ts · src/TaskQueue.ts · src/CronScheduler.ts · src/TemporalContext.ts
Part 2 (next pass)	src/TickEngine.ts · src/ObservationLog.ts · src/KairosAgent.ts · src/KairosReporter.ts · src/index.ts · All 4 test files fully runnable with bun test
Full package file tree
text

packages/kairos/
├── package.json
├── tsconfig.json
└── src/
    ├── types/
    │   └── kairos.types.ts       ← Full type ABI: tasks, schedules, ticks, observations
    ├── KairosStore.ts             ← SQLite persistence (tasks, schedules, observations, tick log)
    ├── TaskQueue.ts               ← Deadline-aware priority queue (min-heap, urgency scoring)
    ├── CronScheduler.ts           ← Cron-style job scheduling (no external deps, pure TS)
    ├── TemporalContext.ts         ← Injects time-awareness into agent prompts + context budget
    ├── TickEngine.ts              ← Proactive tick loop (always-on, interruptible) — Part 2
    ├── ObservationLog.ts          ← Append-only daily observation log (KAIROS pattern) — Part 2
    ├── KairosAgent.ts             ← Agent integration: surfaces deadlines, runs background tasks — Part 2
    ├── KairosReporter.ts          ← Generates KAIROS_REPORT.md + schedule summary — Part 2
    └── index.ts                   ← Barrel export — Part 2
    tests/
    ├── KairosStore.test.ts        (Part 2)
    ├── TaskQueue.test.ts          (Part 2)
    ├── CronScheduler.test.ts      (Part 2)
    └── TemporalContext.test.ts    (Part 2)
packages/kairos/package.json
JSON

{
  "name": "@locoworker/kairos",
  "version": "1.0.0",
  "description": "Temporal intelligence layer for LocoWorker: proactive tick engine, cron scheduler, deadline-aware task queue, and always-on background agent (KAIROS mode)",
  "private": true,
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
    "build": "tsc --project tsconfig.json",
    "dev": "tsc --project tsconfig.json --watch",
    "typecheck": "tsc --noEmit",
    "lint": "biome check ./src",
    "test": "bun test ./src/tests",
    "test:watch": "bun test --watch ./src/tests",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "better-sqlite3": "^9.4.3",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.10",
    "@types/bun": "latest",
    "@locoworker/shared": "workspace:*",
    "typescript": "^5.4.5"
  },
  "peerDependencies": {
    "@locoworker/core": "workspace:*",
    "@locoworker/memory": "workspace:*",
    "@locoworker/wiki": "workspace:*"
  },
  "peerDependenciesMeta": {
    "@locoworker/memory": { "optional": true },
    "@locoworker/wiki": { "optional": true }
  }
}
packages/kairos/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "paths": {
      "@locoworker/core":   ["../core/src/index.ts"],
      "@locoworker/shared": ["../shared/src/index.ts"],
      "@locoworker/memory": ["../memory/src/index.ts"],
      "@locoworker/wiki":   ["../wiki/src/index.ts"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "src/tests"]
}
src/types/kairos.types.ts
TypeScript

/**
 * @locoworker/kairos — type definitions
 *
 * Kairos (Greek: καιρός) = the right or opportune moment to act.
 * Contrasted with Chronos (χρόνος) = sequential clock time.
 *
 * This package implements the LocoWorker temporal intelligence layer:
 *  - Deadline-aware task queue with urgency scoring
 *  - Cron-style background job scheduling
 *  - Proactive tick engine (always-on background agent loop)
 *  - Append-only daily observation log
 *  - Temporal context injection into agent prompts
 *
 * Architecture notes:
 *  - KairosStore is the single writer (SQLite, WAL mode)
 *  - TaskQueue, CronScheduler, TemporalContext are pure logic (no DB writes)
 *  - TickEngine drives the background loop via interruptible setTimeout
 *  - ObservationLog is append-only — no updates, no deletes
 *  - KairosAgent bridges the tick engine to packages/core EventBus
 *
 * DB schema overview:
 *   kairos_tasks        — durable task records with deadline, priority, state
 *   kairos_schedules    — cron job definitions
 *   kairos_schedule_runs— history of schedule execution attempts
 *   kairos_observations — append-only observation/insight log (daily buckets)
 *   kairos_tick_log     — lightweight record of every tick fire (for auditing)
 *   kairos_meta         — key/value store (schema_version, last_tick, etc.)
 *
 * Consistency with existing packages:
 *  - Column naming follows GraphBuilder/WikiStore convention
 *    (snake_case, `from_id/to_id` style where applicable)
 *  - Fingerprints are SHA-256 hex (64 chars) where content hashing is needed
 *  - ISO 8601 timestamps throughout (UTC, with `Z` suffix)
 *  - Status enumerations follow WikiPage pattern (explicit string unions)
 */

import { z } from 'zod';

// ─── Core enumerations ────────────────────────────────────────────────────────

/**
 * Task lifecycle states.
 * Transitions: pending → running → done | failed | cancelled
 *              pending → snoozed → pending (after snooze expires)
 *              running → blocked  → running (after blocker resolves)
 */
export type KairosTaskStatus =
  | 'pending'    // Waiting to be picked up
  | 'running'    // Currently being executed
  | 'done'       // Completed successfully
  | 'failed'     // Completed with error
  | 'cancelled'  // Explicitly cancelled
  | 'snoozed'    // Deferred — will return to pending at snooze_until
  | 'blocked';   // Waiting on another task (blocker_task_id set)

/**
 * Priority levels, ordered from highest (critical) to lowest (background).
 * Numeric ranks used for sorting in TaskQueue.
 */
export type KairosTaskPriority =
  | 'critical'   // Must run immediately, overrides all other work
  | 'high'       // Important, should run in current session
  | 'normal'     // Default — run when convenient
  | 'low'        // Background, run during idle or overnight
  | 'backlog';   // Far future — only run during AutoDream / overnight batch

/** Map priority to numeric rank (lower number = higher priority). */
export const PRIORITY_RANK: Record<KairosTaskPriority, number> = {
  critical: 0,
  high:     1,
  normal:   2,
  low:      3,
  backlog:  4,
} as const;

/**
 * Where a task originated.
 * Determines audit trails, permission scope, and display formatting.
 */
export type KairosTaskOrigin =
  | 'human'        // User explicitly created this task
  | 'agent'        // Agent inferred or was instructed to create this task
  | 'autodream'    // AutoDream batch created this during overnight consolidation
  | 'schedule'     // Triggered by a CronScheduler job
  | 'system';      // Internal system task (memory compaction, graph rebuild, etc.)

/**
 * Cron schedule status.
 */
export type KairosScheduleStatus =
  | 'active'    // Running on schedule
  | 'paused'    // Temporarily disabled
  | 'disabled'; // Permanently disabled (not deleted, for auditability)

/**
 * The kind of work a scheduled job performs.
 */
export type KairosScheduleJobType =
  | 'autodream'          // Run AutoDream consolidation
  | 'graph_rebuild'      // Rebuild graphify knowledge graph
  | 'wiki_export'        // Export wiki to HTML/MD/JSON
  | 'memory_compact'     // Run MemoryCompactor
  | 'wiki_sync'          // Sync wiki directory to DB
  | 'custom_agent_task'  // Arbitrary agent task (payload contains instructions)
  | 'notification';      // Surface a reminder/notification to user

/**
 * Tick modes for the TickEngine.
 * Mirrors the KAIROS feature flag behaviour from the Claude Code source.
 */
export type KairosTickMode =
  | 'proactive'    // Agent checks for work proactively (KAIROS mode)
  | 'reactive'     // Agent only responds to explicit prompts
  | 'paused'       // Tick engine running but ticks are suppressed
  | 'sleeping';    // Tick engine deliberately sleeping (cost-saving, off-hours)

/**
 * Priority of a tick in the agent message queue.
 * Maps to the `priority` field in the Claude Code tick implementation.
 */
export type KairosTickPriority = 'now' | 'soon' | 'later';

/**
 * How an observation was created.
 */
export type ObservationSource =
  | 'tick'           // Generated during a proactive tick
  | 'tool_result'    // Derived from a tool result
  | 'agent_insight'  // Agent explicitly logged via [OBSERVE: ...] marker
  | 'schedule'       // Logged when a schedule ran
  | 'user'           // Manually logged by human
  | 'system';        // System-level observation (errors, capacity warnings)

// ─── Task model ───────────────────────────────────────────────────────────────

/**
 * A KairosTask is the fundamental unit of deferred work in the system.
 * Tasks are persistent (stored in SQLite) and survive process restarts.
 *
 * Urgency is computed dynamically from deadline proximity, priority rank,
 * and optional user-specified weight. It is NOT stored — always recomputed.
 */
export interface KairosTask {
  /** Stable UUID for this task. */
  id: string;

  /** Short human-readable title. */
  title: string;

  /** Optional detailed description (Markdown supported). */
  description?: string;

  /** Current lifecycle state. */
  status: KairosTaskStatus;

  /** Task priority. */
  priority: KairosTaskPriority;

  /** Where this task came from. */
  origin: KairosTaskOrigin;

  /**
   * ISO 8601 deadline. If set, urgency score increases as deadline approaches.
   * Overdue tasks get a large urgency boost (configurable).
   */
  deadline?: string;

  /**
   * ISO 8601 time after which the task becomes visible again (snooze mechanism).
   * Only relevant when status === 'snoozed'.
   */
  snoozeUntil?: string;

  /**
   * ID of another task that must complete before this one can run.
   * Only relevant when status === 'blocked'.
   */
  blockerTaskId?: string;

  /**
   * Tags for filtering / grouping tasks.
   * e.g. ['refactor', 'core', 'urgent']
   */
  tags: string[];

  /**
   * Optional link to a graphify node (code entity this task relates to).
   * e.g. "ToolRegistry::class"
   */
  graphifyNode?: string;

  /**
   * Optional link to a wiki page slug (documentation for this task).
   */
  wikiSlug?: string;

  /**
   * Arbitrary JSON-serialisable metadata (tool inputs, external IDs, etc.).
   */
  metadata: Record<string, unknown>;

  /** Session ID that created this task (if originating from an agent session). */
  sessionId?: string;

  /** ISO 8601 creation timestamp. */
  createdAt: string;

  /** ISO 8601 last-updated timestamp. */
  updatedAt: string;

  /** ISO 8601 timestamp when status changed to 'done' or 'failed'. */
  completedAt?: string;

  /** Error message if status === 'failed'. */
  errorMessage?: string;

  /** Number of times this task has been attempted (for retry logic). */
  attemptCount: number;

  /** Maximum attempts before auto-failing. Default: 3. */
  maxAttempts: number;

  /**
   * Estimated duration in minutes (agent hint for scheduling).
   * Used by TemporalContext to warn about time constraints.
   */
  estimatedMinutes?: number;
}

/**
 * Lightweight urgency score computed from a task at query time.
 * NOT stored in DB — always fresh.
 */
export interface TaskUrgencyScore {
  taskId: string;
  baseScore: number;       // From priority rank (0–40)
  deadlineScore: number;   // 0–50 depending on proximity (overdue → capped at 50)
  overdueBonus: number;    // Additional boost if past deadline
  totalScore: number;      // Sum (higher = more urgent)
  isOverdue: boolean;
  hoursUntilDeadline?: number;  // Negative if overdue
}

// ─── Schedule model ───────────────────────────────────────────────────────────

/**
 * A KairosSchedule is a persistent cron-style job definition.
 * The CronScheduler evaluates these against the current time and
 * fires tasks / events accordingly.
 */
export interface KairosSchedule {
  /** Stable UUID for this schedule. */
  id: string;

  /** Human-readable name. e.g. "Nightly AutoDream" */
  name: string;

  /**
   * Cron expression (5-field: minute hour dom month dow).
   * e.g. "0 2 * * *" = 2am every day
   *      "*/15 * * * *" = every 15 minutes
   *      "0 9 * * 1-5" = 9am Monday–Friday
   */
  cronExpression: string;

  /** The type of job to run. */
  jobType: KairosScheduleJobType;

  /**
   * Arbitrary payload passed to the job handler.
   * Shape depends on jobType.
   */
  payload: Record<string, unknown>;

  /** Current status. */
  status: KairosScheduleStatus;

  /**
   * Timezone for cron evaluation. Default: UTC.
   * e.g. "America/New_York", "Europe/London"
   */
  timezone: string;

  /**
   * Timeout in minutes for the job run. Default: 30.
   * If exceeded, run is marked as 'timeout' and schedule continues.
   */
  timeoutMinutes: number;

  /** ISO 8601 timestamp of last successful run. */
  lastRunAt?: string;

  /** ISO 8601 timestamp of next scheduled run (computed by CronScheduler). */
  nextRunAt?: string;

  /** ISO 8601 creation timestamp. */
  createdAt: string;

  /** ISO 8601 last-updated timestamp. */
  updatedAt: string;
}

/**
 * Record of a single schedule execution attempt.
 * Append-only (no updates after creation).
 */
export interface KairosScheduleRun {
  id: string;
  scheduleId: string;
  scheduleName: string;
  jobType: KairosScheduleJobType;
  status: 'running' | 'success' | 'failed' | 'timeout' | 'skipped';
  startedAt: string;
  completedAt?: string;
  durationMs?: number;
  errorMessage?: string;
  payload: Record<string, unknown>;
  /** Task IDs created during this run (if jobType creates tasks). */
  createdTaskIds: string[];
}

// ─── Tick model ───────────────────────────────────────────────────────────────

/**
 * A single tick event from the TickEngine.
 * Mirrors the `<tick>` message pattern from Claude Code's KAIROS implementation.
 */
export interface KairosTick {
  /** UUID for this tick instance. */
  id: string;

  /** ISO 8601 timestamp when the tick fired. */
  firedAt: string;

  /** Tick mode at time of firing. */
  mode: KairosTickMode;

  /** Priority hint for the agent message queue. */
  priority: KairosTickPriority;

  /**
   * The formatted tick content injected into the agent message queue.
   * Format: `<tick>HH:MM:SS</tick>` (matches Claude Code KAIROS pattern)
   */
  content: string;

  /** Number of pending tasks visible to agent at tick time. */
  pendingTaskCount: number;

  /** Number of overdue tasks at tick time. */
  overdueTaskCount: number;

  /** Whether this tick triggered any background work. */
  triggeredWork: boolean;

  /** Session ID the tick was fired into (if agent session active). */
  sessionId?: string;
}

/**
 * Lightweight DB row for tick log (only essential fields for auditing).
 */
export interface KairosTickLogRow {
  id: string;
  fired_at: string;
  mode: KairosTickMode;
  priority: KairosTickPriority;
  pending_task_count: number;
  overdue_task_count: number;
  triggered_work: 0 | 1;
  session_id: string | null;
}

// ─── Observation model ────────────────────────────────────────────────────────

/**
 * An observation is an append-only fact logged during agent operation.
 * Mirrors the "daily append-only log" in Claude Code's KAIROS pattern.
 * Observations feed into AutoDream consolidation.
 *
 * [OBSERVE: ...] marker in agent messages creates observations automatically.
 */
export interface KairosObservation {
  /** UUID for this observation. */
  id: string;

  /**
   * Observation content (free text or Markdown).
   * Should be concise — this is a log line, not a wiki page.
   */
  content: string;

  /** How this observation was created. */
  source: ObservationSource;

  /**
   * Date bucket (YYYY-MM-DD) for efficient daily queries.
   * Always in UTC.
   */
  dateBucket: string;

  /** ISO 8601 creation timestamp. */
  createdAt: string;

  /** Session ID associated with this observation (if applicable). */
  sessionId?: string;

  /** Optional tags for filtering. */
  tags: string[];

  /**
   * Optional link to a task that triggered this observation.
   */
  relatedTaskId?: string;

  /**
   * Importance level (1–5). Used by AutoDream to prioritise consolidation.
   * 1 = trivial, 5 = critical insight
   */
  importance: 1 | 2 | 3 | 4 | 5;

  /**
   * Whether this observation has been consolidated by AutoDream.
   * Consolidated observations are archived but never deleted.
   */
  consolidated: boolean;
}

// ─── DB row types ─────────────────────────────────────────────────────────────

/** SQLite `kairos_tasks` row. */
export interface KairosTaskRow {
  id: string;
  title: string;
  description: string | null;
  status: KairosTaskStatus;
  priority: KairosTaskPriority;
  origin: KairosTaskOrigin;
  deadline: string | null;
  snooze_until: string | null;
  blocker_task_id: string | null;
  tags_csv: string;
  graphify_node: string | null;
  wiki_slug: string | null;
  metadata_json: string;
  session_id: string | null;
  created_at: string;
  updated_at: string;
  completed_at: string | null;
  error_message: string | null;
  attempt_count: number;
  max_attempts: number;
  estimated_minutes: number | null;
}

/** SQLite `kairos_schedules` row. */
export interface KairosScheduleRow {
  id: string;
  name: string;
  cron_expression: string;
  job_type: KairosScheduleJobType;
  payload_json: string;
  status: KairosScheduleStatus;
  timezone: string;
  timeout_minutes: number;
  last_run_at: string | null;
  next_run_at: string | null;
  created_at: string;
  updated_at: string;
}

/** SQLite `kairos_schedule_runs` row. */
export interface KairosScheduleRunRow {
  id: string;
  schedule_id: string;
  schedule_name: string;
  job_type: KairosScheduleJobType;
  status: string;
  started_at: string;
  completed_at: string | null;
  duration_ms: number | null;
  error_message: string | null;
  payload_json: string;
  created_task_ids_csv: string;
}

/** SQLite `kairos_observations` row. */
export interface KairosObservationRow {
  id: string;
  content: string;
  source: ObservationSource;
  date_bucket: string;
  created_at: string;
  session_id: string | null;
  tags_csv: string;
  related_task_id: string | null;
  importance: number;
  consolidated: 0 | 1;
}

// ─── Query & result types ─────────────────────────────────────────────────────

export interface KairosTaskQueryOptions {
  status?: KairosTaskStatus | KairosTaskStatus[];
  priority?: KairosTaskPriority | KairosTaskPriority[];
  origin?: KairosTaskOrigin;
  tags?: string[];
  overdueOnly?: boolean;
  hasDeadline?: boolean;
  sessionId?: string;
  limit?: number;
  offset?: number;
  /** Sort by urgency score (descending). Default: true. */
  sortByUrgency?: boolean;
}

export interface KairosObservationQueryOptions {
  dateBucket?: string;         // YYYY-MM-DD — filter to a specific day
  dateFrom?: string;           // ISO 8601 range start
  dateTo?: string;             // ISO 8601 range end
  source?: ObservationSource;
  sessionId?: string;
  consolidated?: boolean;
  importanceMin?: number;
  tags?: string[];
  limit?: number;
  offset?: number;
}

export interface KairosStoreStats {
  totalTasks: number;
  pendingTasks: number;
  runningTasks: number;
  overdueTasks: number;
  completedTasks: number;
  failedTasks: number;
  activeSchedules: number;
  totalObservations: number;
  unconsolidatedObservations: number;
  totalTicks: number;
  lastTickAt: string | null;
  dbSizeBytes: number;
}

// ─── Temporal context types ───────────────────────────────────────────────────

/**
 * The structured temporal context assembled by TemporalContext
 * and injected into agent prompts as a system section.
 */
export interface TemporalContextSnapshot {
  /** ISO 8601 current time (UTC). */
  nowUtc: string;

  /** Local time string (locale-formatted). */
  nowLocal: string;

  /** Day of week (0=Sunday … 6=Saturday). */
  dayOfWeek: number;

  /** Human-readable day name. */
  dayName: string;

  /** Hour of day in local time (0–23). */
  hourOfDay: number;

  /** Whether current time is within "working hours" (configurable). */
  isWorkingHours: boolean;

  /** Whether current time is overnight / off-hours. */
  isOvernightWindow: boolean;

  /** Overdue tasks count (visible to agent). */
  overdueTasks: number;

  /** Tasks due within the next 24 hours. */
  tasksDueToday: number;

  /** Top N urgent tasks surfaced for the agent. */
  urgentTasks: Array<{
    id: string;
    title: string;
    priority: KairosTaskPriority;
    deadline?: string;
    isOverdue: boolean;
    hoursUntilDeadline?: number;
  }>;

  /** Next scheduled job (if any) and when it runs. */
  nextScheduledJob?: {
    name: string;
    jobType: KairosScheduleJobType;
    nextRunAt: string;
    minutesUntilRun: number;
  };

  /** Observations logged in the current day bucket. */
  todayObservationCount: number;

  /** Approximate token cost of this context block. */
  estimatedTokens: number;
}

/**
 * Options for TemporalContext assembly.
 */
export interface TemporalContextOptions {
  /** Max urgent tasks to surface in context. Default: 5. */
  maxUrgentTasks?: number;

  /** Max token budget for the entire temporal context block. Default: 400. */
  maxTokens?: number;

  /** Working hours window (local time, 24h). Default: { start: 8, end: 18 }. */
  workingHours?: { start: number; end: number };

  /** Overnight window (local time, 24h). Default: { start: 22, end: 6 }. */
  overnightWindow?: { start: number; end: number };

  /** Timezone for local time calculations. Default: system local. */
  timezone?: string;

  /** Whether to include today's observation count. Default: true. */
  includeObservationCount?: boolean;

  /** Whether to include next scheduled job. Default: true. */
  includeNextJob?: boolean;
}

// ─── Cron expression types ────────────────────────────────────────────────────

/**
 * Parsed representation of a 5-field cron expression.
 * Fields: minute | hour | day-of-month | month | day-of-week
 */
export interface ParsedCronExpression {
  raw: string;
  minute: CronField;
  hour: CronField;
  dayOfMonth: CronField;
  month: CronField;
  dayOfWeek: CronField;
}

/**
 * A single cron field after parsing.
 * `values` is the sorted set of discrete values this field matches.
 * e.g. "*/15" on minute → [0, 15, 30, 45]
 *      "1-5"  on dow   → [1, 2, 3, 4, 5]
 *      "*"    on hour  → [0,1,2,...,23]
 */
export interface CronField {
  raw: string;
  values: number[];
  isWildcard: boolean;
}

// ─── Zod validation schemas ───────────────────────────────────────────────────

export const KairosTaskSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(300),
  description: z.string().optional(),
  status: z.enum(['pending','running','done','failed','cancelled','snoozed','blocked']),
  priority: z.enum(['critical','high','normal','low','backlog']),
  origin: z.enum(['human','agent','autodream','schedule','system']),
  deadline: z.string().datetime({ offset: true }).optional(),
  snoozeUntil: z.string().datetime({ offset: true }).optional(),
  blockerTaskId: z.string().uuid().optional(),
  tags: z.array(z.string()).default([]),
  graphifyNode: z.string().optional(),
  wikiSlug: z.string().optional(),
  metadata: z.record(z.unknown()).default({}),
  sessionId: z.string().optional(),
  createdAt: z.string().datetime({ offset: true }),
  updatedAt: z.string().datetime({ offset: true }),
  completedAt: z.string().datetime({ offset: true }).optional(),
  errorMessage: z.string().optional(),
  attemptCount: z.number().int().nonneg().default(0),
  maxAttempts: z.number().int().positive().default(3),
  estimatedMinutes: z.number().int().positive().optional(),
});

export const KairosScheduleSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(200),
  cronExpression: z.string().regex(
    /^(\S+)\s+(\S+)\s+(\S+)\s+(\S+)\s+(\S+)$/,
    'Must be a valid 5-field cron expression',
  ),
  jobType: z.enum([
    'autodream','graph_rebuild','wiki_export','memory_compact',
    'wiki_sync','custom_agent_task','notification',
  ]),
  payload: z.record(z.unknown()).default({}),
  status: z.enum(['active','paused','disabled']),
  timezone: z.string().default('UTC'),
  timeoutMinutes: z.number().int().positive().default(30),
  lastRunAt: z.string().datetime({ offset: true }).optional(),
  nextRunAt: z.string().datetime({ offset: true }).optional(),
  createdAt: z.string().datetime({ offset: true }),
  updatedAt: z.string().datetime({ offset: true }),
});

// ─── Constants ────────────────────────────────────────────────────────────────

/** SQLite schema version — bump when schema changes. */
export const KAIROS_SCHEMA_VERSION = 1;

/** Default tick interval in milliseconds (10 minutes). */
export const DEFAULT_TICK_INTERVAL_MS = 10 * 60 * 1000;

/** Minimum tick interval (safety floor: 30 seconds). */
export const MIN_TICK_INTERVAL_MS = 30 * 1000;

/** Default SQLite DB path relative to project root. */
export const DEFAULT_KAIROS_DB = '.locoworker/kairos/kairos.db';

/** Max urgent tasks surfaced per tick. */
export const MAX_URGENT_TASKS_PER_TICK = 5;

/** Urgency score boost when a task is overdue. */
export const OVERDUE_URGENCY_BOOST = 30;

/** Urgency score when deadline is within 1 hour. */
export const DEADLINE_IMMINENT_SCORE = 50;

/** Urgency score when deadline is within 24 hours. */
export const DEADLINE_TODAY_SCORE = 25;

/** Urgency score when deadline is within 7 days. */
export const DEADLINE_THIS_WEEK_SCORE = 10;

/** Priority base scores (maps directly to PRIORITY_RANK * 10, inverted). */
export const PRIORITY_BASE_SCORES: Record<KairosTaskPriority, number> = {
  critical: 40,
  high:     30,
  normal:   20,
  low:      10,
  backlog:   0,
} as const;

/** [OBSERVE: ...] marker for agent messages → creates observations. */
export const OBSERVE_MARKER_REGEX =
  /\[OBSERVE(?::([1-5]))?\s*:\s*([\s\S]*?)\]/g;

/** Default working hours window (24h local time). */
export const DEFAULT_WORKING_HOURS = { start: 8, end: 18 } as const;

/** Default overnight window (24h local time). */
export const DEFAULT_OVERNIGHT_WINDOW = { start: 22, end: 6 } as const;

/** Approximate tokens per character for context budget estimation. */
export const CHARS_PER_TOKEN = 4;

/** Default built-in schedules registered on first init. */
export const DEFAULT_SCHEDULES: Array<Omit<KairosSchedule,
  'id' | 'createdAt' | 'updatedAt' | 'lastRunAt' | 'nextRunAt'>> = [
  {
    name: 'Nightly AutoDream',
    cronExpression: '0 2 * * *',       // 2:00am daily
    jobType: 'autodream',
    payload: {},
    status: 'active',
    timezone: 'UTC',
    timeoutMinutes: 60,
  },
  {
    name: 'Nightly Graph Rebuild',
    cronExpression: '30 2 * * *',      // 2:30am daily (after AutoDream)
    jobType: 'graph_rebuild',
    payload: {},
    status: 'active',
    timezone: 'UTC',
    timeoutMinutes: 30,
  },
  {
    name: 'Weekly Wiki Export',
    cronExpression: '0 3 * * 0',       // 3:00am every Sunday
    jobType: 'wiki_export',
    payload: { format: 'markdown_bundle' },
    status: 'active',
    timezone: 'UTC',
    timeoutMinutes: 15,
  },
  {
    name: 'Hourly Memory Compact',
    cronExpression: '0 * * * *',       // Top of every hour
    jobType: 'memory_compact',
    payload: {},
    status: 'active',
    timezone: 'UTC',
    timeoutMinutes: 10,
  },
] as const;
src/KairosStore.ts
TypeScript

/**
 * KairosStore — SQLite persistence layer for @locoworker/kairos
 *
 * Tables:
 *   kairos_tasks          — durable task records
 *   kairos_schedules      — cron job definitions
 *   kairos_schedule_runs  — execution history (append-only)
 *   kairos_observations   — append-only observation log
 *   kairos_tick_log       — lightweight tick audit log (append-only)
 *   kairos_meta           — key/value store
 *
 * Design principles:
 *  - KairosStore is the ONLY class that writes to the DB.
 *  - All writes are transactional (better-sqlite3 synchronous transactions).
 *  - WAL mode + normal sync for performance (same as WikiStore, GraphBuilder).
 *  - Observations and tick log are append-only — no UPDATE/DELETE on those tables.
 *  - Schema is derived-rebuilable: task/schedule data is primary, history is preserved.
 *  - Column naming follows established monorepo conventions (snake_case, tags_csv, etc.).
 */

import Database, { type Database as DB } from 'better-sqlite3';
import { createHash, randomUUID } from 'node:crypto';
import { existsSync, mkdirSync } from 'node:fs';
import { dirname } from 'node:path';

import type {
  KairosTask,
  KairosTaskRow,
  KairosSchedule,
  KairosScheduleRow,
  KairosScheduleRun,
  KairosScheduleRunRow,
  KairosObservation,
  KairosObservationRow,
  KairosTickLogRow,
  KairosTick,
  KairosTaskStatus,
  KairosTaskPriority,
  KairosTaskOrigin,
  KairosTaskQueryOptions,
  KairosObservationQueryOptions,
  KairosStoreStats,
  KairosScheduleStatus,
} from './types/kairos.types.js';

import {
  KAIROS_SCHEMA_VERSION,
  DEFAULT_SCHEDULES,
} from './types/kairos.types.js';

// ─── Helpers ──────────────────────────────────────────────────────────────────

function nowIso(): string {
  return new Date().toISOString();
}

function ensureDir(filePath: string): void {
  if (filePath === ':memory:') return;
  const dir = dirname(filePath);
  if (!existsSync(dir)) mkdirSync(dir, { recursive: true });
}

// ─── KairosStore ──────────────────────────────────────────────────────────────

export class KairosStore {
  readonly db: DB;
  private readonly dbPath: string;

  constructor(dbPath: string, seedDefaultSchedules = true) {
    this.dbPath = dbPath;
    ensureDir(dbPath);
    this.db = new Database(dbPath);
    this.applyPragmas();
    this.createSchema();
    this.assertSchemaVersion();
    if (seedDefaultSchedules) {
      this.seedDefaultSchedules();
    }
  }

  // ─── Lifecycle ──────────────────────────────────────────────────────────────

  close(): void {
    this.db.close();
  }

  // ─── Task CRUD ──────────────────────────────────────────────────────────────

  /**
   * Create a new task. Returns the created task with a generated ID.
   */
  createTask(
    input: Omit<KairosTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'>,
  ): KairosTask {
    const now = nowIso();
    const task: KairosTask = {
      ...input,
      id: randomUUID(),
      attemptCount: 0,
      createdAt: now,
      updatedAt: now,
    };

    this.db.prepare<KairosTaskRow>(/* sql */ `
      INSERT INTO kairos_tasks
        (id, title, description, status, priority, origin, deadline, snooze_until,
         blocker_task_id, tags_csv, graphify_node, wiki_slug, metadata_json,
         session_id, created_at, updated_at, completed_at, error_message,
         attempt_count, max_attempts, estimated_minutes)
      VALUES
        (@id, @title, @description, @status, @priority, @origin, @deadline,
         @snooze_until, @blocker_task_id, @tags_csv, @graphify_node, @wiki_slug,
         @metadata_json, @session_id, @created_at, @updated_at, @completed_at,
         @error_message, @attempt_count, @max_attempts, @estimated_minutes)
    `).run(this._taskToRow(task));

    return task;
  }

  /**
   * Get a single task by ID. Returns undefined if not found.
   */
  getTask(id: string): KairosTask | undefined {
    const row = this.db
      .prepare<[string], KairosTaskRow>('SELECT * FROM kairos_tasks WHERE id = ?')
      .get(id);
    return row ? this._rowToTask(row) : undefined;
  }

  /**
   * Update a task. Only updates provided fields (partial update).
   * Always bumps `updated_at`.
   */
  updateTask(
    id: string,
    updates: Partial<Omit<KairosTask, 'id' | 'createdAt'>>,
  ): KairosTask | undefined {
    const existing = this.getTask(id);
    if (!existing) return undefined;

    const merged: KairosTask = {
      ...existing,
      ...updates,
      id,
      createdAt: existing.createdAt,
      updatedAt: nowIso(),
    };

    this.db.prepare(/* sql */ `
      UPDATE kairos_tasks SET
        title            = @title,
        description      = @description,
        status           = @status,
        priority         = @priority,
        origin           = @origin,
        deadline         = @deadline,
        snooze_until     = @snooze_until,
        blocker_task_id  = @blocker_task_id,
        tags_csv         = @tags_csv,
        graphify_node    = @graphify_node,
        wiki_slug        = @wiki_slug,
        metadata_json    = @metadata_json,
        session_id       = @session_id,
        updated_at       = @updated_at,
        completed_at     = @completed_at,
        error_message    = @error_message,
        attempt_count    = @attempt_count,
        max_attempts     = @max_attempts,
        estimated_minutes= @estimated_minutes
      WHERE id = @id
    `).run(this._taskToRow(merged));

    return merged;
  }

  /**
   * Transition a task to a new status with optional metadata.
   * Handles side effects: sets completedAt on terminal statuses,
   * clears snooze_until on un-snooze, increments attempt_count on 'running'.
   */
  transitionTask(
    id: string,
    newStatus: KairosTaskStatus,
    opts: {
      errorMessage?: string;
      snoozeUntil?: string;
      blockerTaskId?: string;
    } = {},
  ): KairosTask | undefined {
    const task = this.getTask(id);
    if (!task) return undefined;

    const now = nowIso();
    const isTerminal = newStatus === 'done' || newStatus === 'failed' || newStatus === 'cancelled';
    const isStarting = newStatus === 'running';

    const updates: Partial<KairosTask> = {
      status: newStatus,
      updatedAt: now,
      ...(isTerminal ? { completedAt: now } : {}),
      ...(isStarting ? { attemptCount: task.attemptCount + 1 } : {}),
      ...(opts.errorMessage ? { errorMessage: opts.errorMessage } : {}),
      ...(opts.snoozeUntil ? { snoozeUntil: opts.snoozeUntil } : {}),
      ...(opts.blockerTaskId ? { blockerTaskId: opts.blockerTaskId } : {}),
      // Clear snooze when un-snoozing
      ...(newStatus === 'pending' && task.status === 'snoozed'
        ? { snoozeUntil: undefined }
        : {}),
    };

    return this.updateTask(id, updates);
  }

  /**
   * Delete a task permanently (hard delete). Use only for cleanup.
   */
  deleteTask(id: string): boolean {
    const result = this.db
      .prepare('DELETE FROM kairos_tasks WHERE id = ?')
      .run(id);
    return result.changes > 0;
  }

  /**
   * Query tasks with filtering and optional urgency sorting.
   */
  queryTasks(options: KairosTaskQueryOptions = {}): KairosTaskRow[] {
    const conditions: string[] = [];
    const params: (string | number)[] = [];
    const limit = options.limit ?? 100;
    const offset = options.offset ?? 0;

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      conditions.push(`status IN (${statuses.map(() => '?').join(', ')})`);
      params.push(...statuses);
    }

    if (options.priority) {
      const priorities = Array.isArray(options.priority) ? options.priority : [options.priority];
      conditions.push(`priority IN (${priorities.map(() => '?').join(', ')})`);
      params.push(...priorities);
    }

    if (options.origin) {
      conditions.push('origin = ?');
      params.push(options.origin);
    }

    if (options.sessionId) {
      conditions.push('session_id = ?');
      params.push(options.sessionId);
    }

    if (options.overdueOnly) {
      conditions.push("deadline IS NOT NULL AND deadline < datetime('now')");
    }

    if (options.hasDeadline) {
      conditions.push('deadline IS NOT NULL');
    }

    if (options.tags && options.tags.length > 0) {
      for (const tag of options.tags) {
        conditions.push('("," || tags_csv || ",") LIKE ?');
        params.push(`%,${tag},%`);
      }
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

    // Sort by urgency components:
    // 1. Overdue tasks first (deadline < now)
    // 2. By priority rank (critical first)
    // 3. By deadline proximity (soonest first)
    // 4. By creation date (oldest first as tiebreaker)
    const orderBy = options.sortByUrgency !== false
      ? `ORDER BY
          CASE WHEN deadline IS NOT NULL AND deadline < datetime('now') THEN 0 ELSE 1 END,
          CASE priority
            WHEN 'critical' THEN 0
            WHEN 'high'     THEN 1
            WHEN 'normal'   THEN 2
            WHEN 'low'      THEN 3
            WHEN 'backlog'  THEN 4
          END,
          CASE WHEN deadline IS NOT NULL THEN deadline ELSE '9999' END,
          created_at`
      : 'ORDER BY created_at';

    return this.db
      .prepare<(string | number)[], KairosTaskRow>(
        `SELECT * FROM kairos_tasks ${where} ${orderBy} LIMIT ? OFFSET ?`,
      )
      .all(...params, limit, offset);
  }

  /**
   * Wake up snoozed tasks whose snooze_until has passed.
   * Returns the number of tasks woken.
   */
  wakeupSnoozedTasks(): number {
    const result = this.db.prepare(/* sql */ `
      UPDATE kairos_tasks
      SET status = 'pending', snooze_until = NULL, updated_at = ?
      WHERE status = 'snoozed'
        AND snooze_until IS NOT NULL
        AND snooze_until <= datetime('now')
    `).run(nowIso());
    return result.changes;
  }

  /**
   * Unblock tasks whose blocker task has completed.
   * Returns the number of tasks unblocked.
   */
  unblockTasks(): number {
    const result = this.db.prepare(/* sql */ `
      UPDATE kairos_tasks
      SET status = 'pending', blocker_task_id = NULL, updated_at = ?
      WHERE status = 'blocked'
        AND blocker_task_id IS NOT NULL
        AND blocker_task_id IN (
          SELECT id FROM kairos_tasks WHERE status = 'done'
        )
    `).run(nowIso());
    return result.changes;
  }

  /**
   * Auto-fail tasks that have exceeded max_attempts.
   * Returns the number of tasks auto-failed.
   */
  autoFailExhaustedTasks(): number {
    const result = this.db.prepare(/* sql */ `
      UPDATE kairos_tasks
      SET status = 'failed',
          error_message = 'Max attempts exceeded',
          completed_at = ?,
          updated_at = ?
      WHERE status IN ('pending', 'running')
        AND attempt_count >= max_attempts
        AND max_attempts > 0
    `).run(nowIso(), nowIso());
    return result.changes;
  }

  // ─── Schedule CRUD ──────────────────────────────────────────────────────────

  /**
   * Create a new schedule.
   */
  createSchedule(
    input: Omit<KairosSchedule, 'id' | 'createdAt' | 'updatedAt' | 'lastRunAt' | 'nextRunAt'>,
  ): KairosSchedule {
    const now = nowIso();
    const schedule: KairosSchedule = {
      ...input,
      id: randomUUID(),
      createdAt: now,
      updatedAt: now,
    };

    this.db.prepare<KairosScheduleRow>(/* sql */ `
      INSERT OR IGNORE INTO kairos_schedules
        (id, name, cron_expression, job_type, payload_json, status,
         timezone, timeout_minutes, last_run_at, next_run_at, created_at, updated_at)
      VALUES
        (@id, @name, @cron_expression, @job_type, @payload_json, @status,
         @timezone, @timeout_minutes, @last_run_at, @next_run_at, @created_at, @updated_at)
    `).run(this._scheduleToRow(schedule));

    return schedule;
  }

  getSchedule(id: string): KairosSchedule | undefined {
    const row = this.db
      .prepare<[string], KairosScheduleRow>('SELECT * FROM kairos_schedules WHERE id = ?')
      .get(id);
    return row ? this._rowToSchedule(row) : undefined;
  }

  getAllSchedules(status?: KairosScheduleStatus): KairosScheduleRow[] {
    if (!status) {
      return this.db
        .prepare<[], KairosScheduleRow>('SELECT * FROM kairos_schedules ORDER BY name')
        .all();
    }
    return this.db
      .prepare<[string], KairosScheduleRow>(
        'SELECT * FROM kairos_schedules WHERE status = ? ORDER BY name',
      )
      .all(status);
  }

  updateSchedule(
    id: string,
    updates: Partial<Omit<KairosSchedule, 'id' | 'createdAt'>>,
  ): KairosSchedule | undefined {
    const existing = this.getSchedule(id);
    if (!existing) return undefined;

    const merged: KairosSchedule = { ...existing, ...updates, id, updatedAt: nowIso() };
    this.db.prepare(/* sql */ `
      UPDATE kairos_schedules SET
        name            = @name,
        cron_expression = @cron_expression,
        job_type        = @job_type,
        payload_json    = @payload_json,
        status          = @status,
        timezone        = @timezone,
        timeout_minutes = @timeout_minutes,
        last_run_at     = @last_run_at,
        next_run_at     = @next_run_at,
        updated_at      = @updated_at
      WHERE id = @id
    `).run(this._scheduleToRow(merged));
    return merged;
  }

  // ─── Schedule runs (append-only) ───────────────────────────────────────────

  /**
   * Record the start of a schedule run. Returns the run ID.
   */
  startScheduleRun(
    scheduleId: string,
    scheduleName: string,
    jobType: KairosScheduleRun['jobType'],
    payload: Record<string, unknown>,
  ): string {
    const id = randomUUID();
    const now = nowIso();

    this.db.prepare(/* sql */ `
      INSERT INTO kairos_schedule_runs
        (id, schedule_id, schedule_name, job_type, status, started_at,
         completed_at, duration_ms, error_message, payload_json, created_task_ids_csv)
      VALUES (?, ?, ?, ?, 'running', ?, NULL, NULL, NULL, ?, '')
    `).run(id, scheduleId, scheduleName, jobType, now, JSON.stringify(payload));

    return id;
  }

  /**
   * Complete a schedule run (success or failure).
   */
  completeScheduleRun(
    runId: string,
    status: 'success' | 'failed' | 'timeout',
    opts: { errorMessage?: string; createdTaskIds?: string[] } = {},
  ): void {
    const now = nowIso();

    // Compute duration from started_at
    const run = this.db
      .prepare<[string], { started_at: string }>(
        'SELECT started_at FROM kairos_schedule_runs WHERE id = ?',
      )
      .get(runId);

    const durationMs = run
      ? Date.now() - new Date(run.started_at).getTime()
      : null;

    this.db.prepare(/* sql */ `
      UPDATE kairos_schedule_runs SET
        status = ?, completed_at = ?, duration_ms = ?,
        error_message = ?, created_task_ids_csv = ?
      WHERE id = ?
    `).run(
      status,
      now,
      durationMs,
      opts.errorMessage ?? null,
      (opts.createdTaskIds ?? []).join(','),
      runId,
    );
  }

  getScheduleRuns(scheduleId: string, limit = 20): KairosScheduleRunRow[] {
    return this.db
      .prepare<[string, number], KairosScheduleRunRow>(
        'SELECT * FROM kairos_schedule_runs WHERE schedule_id = ? ORDER BY started_at DESC LIMIT ?',
      )
      .all(scheduleId, limit);
  }

  // ─── Observations (append-only) ────────────────────────────────────────────

  /**
   * Append a new observation. Returns the created observation.
   */
  appendObservation(
    input: Omit<KairosObservation, 'id' | 'createdAt' | 'consolidated'>,
  ): KairosObservation {
    const now = nowIso();
    const observation: KairosObservation = {
      ...input,
      id: randomUUID(),
      createdAt: now,
      consolidated: false,
    };

    this.db.prepare(/* sql */ `
      INSERT INTO kairos_observations
        (id, content, source, date_bucket, created_at, session_id,
         tags_csv, related_task_id, importance, consolidated)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 0)
    `).run(
      observation.id,
      observation.content,
      observation.source,
      observation.dateBucket,
      now,
      observation.sessionId ?? null,
      observation.tags.join(','),
      observation.relatedTaskId ?? null,
      observation.importance,
    );

    return observation;
  }

  /**
   * Query observations with filtering.
   */
  queryObservations(options: KairosObservationQueryOptions = {}): KairosObservationRow[] {
    const conditions: string[] = [];
    const params: (string | number)[] = [];
    const limit = options.limit ?? 100;
    const offset = options.offset ?? 0;

    if (options.dateBucket) {
      conditions.push('date_bucket = ?');
      params.push(options.dateBucket);
    }

    if (options.dateFrom) {
      conditions.push('created_at >= ?');
      params.push(options.dateFrom);
    }

    if (options.dateTo) {
      conditions.push('created_at <= ?');
      params.push(options.dateTo);
    }

    if (options.source) {
      conditions.push('source = ?');
      params.push(options.source);
    }

    if (options.sessionId) {
      conditions.push('session_id = ?');
      params.push(options.sessionId);
    }

    if (typeof options.consolidated === 'boolean') {
      conditions.push('consolidated = ?');
      params.push(options.consolidated ? 1 : 0);
    }

    if (options.importanceMin) {
      conditions.push('importance >= ?');
      params.push(options.importanceMin);
    }

    if (options.tags && options.tags.length > 0) {
      for (const tag of options.tags) {
        conditions.push('("," || tags_csv || ",") LIKE ?');
        params.push(`%,${tag},%`);
      }
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

    return this.db
      .prepare<(string | number)[], KairosObservationRow>(
        `SELECT * FROM kairos_observations ${where} ORDER BY created_at DESC LIMIT ? OFFSET ?`,
      )
      .all(...params, limit, offset);
  }

  /**
   * Mark observations as consolidated (called by AutoDream after processing).
   * Accepts an array of observation IDs.
   */
  markObservationsConsolidated(ids: string[]): number {
    if (ids.length === 0) return 0;
    const placeholders = ids.map(() => '?').join(', ');
    const result = this.db
      .prepare(`UPDATE kairos_observations SET consolidated = 1 WHERE id IN (${placeholders})`)
      .run(...ids);
    return result.changes;
  }

  /**
   * Get today's observation count (for TemporalContext injection).
   */
  getTodayObservationCount(): number {
    const bucket = new Date().toISOString().slice(0, 10);
    const row = this.db
      .prepare<[string], { count: number }>(
        'SELECT COUNT(*) as count FROM kairos_observations WHERE date_bucket = ?',
      )
      .get(bucket);
    return row?.count ?? 0;
  }

  // ─── Tick log (append-only) ────────────────────────────────────────────────

  /**
   * Append a tick record to the audit log.
   */
  logTick(tick: KairosTick): void {
    this.db.prepare(/* sql */ `
      INSERT INTO kairos_tick_log
        (id, fired_at, mode, priority, pending_task_count,
         overdue_task_count, triggered_work, session_id)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      tick.id,
      tick.firedAt,
      tick.mode,
      tick.priority,
      tick.pendingTaskCount,
      tick.overdueTaskCount,
      tick.triggeredWork ? 1 : 0,
      tick.sessionId ?? null,
    );
  }

  getRecentTicks(limit = 50): KairosTickLogRow[] {
    return this.db
      .prepare<[number], KairosTickLogRow>(
        'SELECT * FROM kairos_tick_log ORDER BY fired_at DESC LIMIT ?',
      )
      .all(limit);
  }

  getLastTickAt(): string | null {
    const row = this.db
      .prepare<[], { fired_at: string }>(
        'SELECT fired_at FROM kairos_tick_log ORDER BY fired_at DESC LIMIT 1',
      )
      .get();
    return row?.fired_at ?? null;
  }

  // ─── Stats ─────────────────────────────────────────────────────────────────

  getStats(): KairosStoreStats {
    const taskCounts = this.db
      .prepare<[], {
        total: number; pending: number; running: number;
        done: number; failed: number;
      }>(/* sql */ `
        SELECT
          COUNT(*) as total,
          SUM(CASE WHEN status='pending'  THEN 1 ELSE 0 END) as pending,
          SUM(CASE WHEN status='running'  THEN 1 ELSE 0 END) as running,
          SUM(CASE WHEN status='done'     THEN 1 ELSE 0 END) as done,
          SUM(CASE WHEN status='failed'   THEN 1 ELSE 0 END) as failed
        FROM kairos_tasks
      `)
      .get()!;

    const overdueCount = (this.db
      .prepare<[], { count: number }>(/* sql */ `
        SELECT COUNT(*) as count FROM kairos_tasks
        WHERE status IN ('pending','running')
          AND deadline IS NOT NULL
          AND deadline < datetime('now')
      `)
      .get()?.count ?? 0);

    const activeSchedules = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM kairos_schedules WHERE status = 'active'",
      )
      .get()?.count ?? 0);

    const obsCounts = this.db
      .prepare<[], { total: number; unconsolidated: number }>(/* sql */ `
        SELECT
          COUNT(*) as total,
          SUM(CASE WHEN consolidated = 0 THEN 1 ELSE 0 END) as unconsolidated
        FROM kairos_observations
      `)
      .get()!;

    const totalTicks = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM kairos_tick_log')
      .get()?.count ?? 0);

    const lastTickAt = this.getLastTickAt();

    const pageCount = (this.db.prepare('PRAGMA page_count').get() as { page_count: number })
      .page_count;
    const pageSize = (this.db.prepare('PRAGMA page_size').get() as { page_size: number })
      .page_size;

    return {
      totalTasks: taskCounts.total,
      pendingTasks: taskCounts.pending,
      runningTasks: taskCounts.running,
      overdueTasks: overdueCount,
      completedTasks: taskCounts.done,
      failedTasks: taskCounts.failed,
      activeSchedules,
      totalObservations: obsCounts.total,
      unconsolidatedObservations: obsCounts.unconsolidated,
      totalTicks,
      lastTickAt,
      dbSizeBytes: pageCount * pageSize,
    };
  }

  // ─── Meta helpers ──────────────────────────────────────────────────────────

  getMeta(key: string): string | null {
    return (
      this.db
        .prepare<[string], { value: string }>('SELECT value FROM kairos_meta WHERE key = ?')
        .get(key)?.value ?? null
    );
  }

  setMeta(key: string, value: string): void {
    this.db
      .prepare('INSERT OR REPLACE INTO kairos_meta(key, value) VALUES (?, ?)')
      .run(key, value);
  }

  // ─── Schema ────────────────────────────────────────────────────────────────

  private applyPragmas(): void {
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('synchronous = NORMAL');
    this.db.pragma('foreign_keys = ON');
    this.db.pragma('cache_size = -8000');
    this.db.pragma('temp_store = MEMORY');
  }

  private createSchema(): void {
    this.db.exec(/* sql */ `
      -- Tasks
      CREATE TABLE IF NOT EXISTS kairos_tasks (
        id                TEXT PRIMARY KEY,
        title             TEXT NOT NULL,
        description       TEXT,
        status            TEXT NOT NULL DEFAULT 'pending',
        priority          TEXT NOT NULL DEFAULT 'normal',
        origin            TEXT NOT NULL DEFAULT 'human',
        deadline          TEXT,
        snooze_until      TEXT,
        blocker_task_id   TEXT,
        tags_csv          TEXT NOT NULL DEFAULT '',
        graphify_node     TEXT,
        wiki_slug         TEXT,
        metadata_json     TEXT NOT NULL DEFAULT '{}',
        session_id        TEXT,
        created_at        TEXT NOT NULL,
        updated_at        TEXT NOT NULL,
        completed_at      TEXT,
        error_message     TEXT,
        attempt_count     INTEGER NOT NULL DEFAULT 0,
        max_attempts      INTEGER NOT NULL DEFAULT 3,
        estimated_minutes INTEGER,
        FOREIGN KEY (blocker_task_id) REFERENCES kairos_tasks(id) ON DELETE SET NULL
      );

      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_status    ON kairos_tasks(status);
      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_priority  ON kairos_tasks(priority);
      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_deadline  ON kairos_tasks(deadline)
        WHERE deadline IS NOT NULL;
      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_session   ON kairos_tasks(session_id)
        WHERE session_id IS NOT NULL;
      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_snooze    ON kairos_tasks(snooze_until)
        WHERE snooze_until IS NOT NULL;
      CREATE INDEX IF NOT EXISTS idx_kairos_tasks_blocker   ON kairos_tasks(blocker_task_id)
        WHERE blocker_task_id IS NOT NULL;

      -- Schedules
      CREATE TABLE IF NOT EXISTS kairos_schedules (
        id               TEXT PRIMARY KEY,
        name             TEXT NOT NULL UNIQUE,
        cron_expression  TEXT NOT NULL,
        job_type         TEXT NOT NULL,
        payload_json     TEXT NOT NULL DEFAULT '{}',
        status           TEXT NOT NULL DEFAULT 'active',
        timezone         TEXT NOT NULL DEFAULT 'UTC',
        timeout_minutes  INTEGER NOT NULL DEFAULT 30,
        last_run_at      TEXT,
        next_run_at      TEXT,
        created_at       TEXT NOT NULL,
        updated_at       TEXT NOT NULL
      );

      CREATE INDEX IF NOT EXISTS idx_kairos_schedules_status  ON kairos_schedules(status);
      CREATE INDEX IF NOT EXISTS idx_kairos_schedules_next    ON kairos_schedules(next_run_at)
        WHERE next_run_at IS NOT NULL;

      -- Schedule run history (append-only)
      CREATE TABLE IF NOT EXISTS kairos_schedule_runs (
        id                   TEXT PRIMARY KEY,
        schedule_id          TEXT NOT NULL,
        schedule_name        TEXT NOT NULL,
        job_type             TEXT NOT NULL,
        status               TEXT NOT NULL DEFAULT 'running',
        started_at           TEXT NOT NULL,
        completed_at         TEXT,
        duration_ms          INTEGER,
        error_message        TEXT,
        payload_json         TEXT NOT NULL DEFAULT '{}',
        created_task_ids_csv TEXT NOT NULL DEFAULT '',
        FOREIGN KEY (schedule_id) REFERENCES kairos_schedules(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_kairos_sched_runs_schedule ON kairos_schedule_runs(schedule_id);
      CREATE INDEX IF NOT EXISTS idx_kairos_sched_runs_started  ON kairos_schedule_runs(started_at DESC);

      -- Observations (append-only)
      CREATE TABLE IF NOT EXISTS kairos_observations (
        id              TEXT PRIMARY KEY,
        content         TEXT NOT NULL,
        source          TEXT NOT NULL,
        date_bucket     TEXT NOT NULL,   -- YYYY-MM-DD
        created_at      TEXT NOT NULL,
        session_id      TEXT,
        tags_csv        TEXT NOT NULL DEFAULT '',
        related_task_id TEXT,
        importance      INTEGER NOT NULL DEFAULT 3,
        consolidated    INTEGER NOT NULL DEFAULT 0,
        FOREIGN KEY (related_task_id) REFERENCES kairos_tasks(id) ON DELETE SET NULL
      );

      CREATE INDEX IF NOT EXISTS idx_kairos_obs_date        ON kairos_observations(date_bucket);
      CREATE INDEX IF NOT EXISTS idx_kairos_obs_source      ON kairos_observations(source);
      CREATE INDEX IF NOT EXISTS idx_kairos_obs_consolidated ON kairos_observations(consolidated)
        WHERE consolidated = 0;
      CREATE INDEX IF NOT EXISTS idx_kairos_obs_importance  ON kairos_observations(importance);

      -- Tick audit log (append-only, lightweight)
      CREATE TABLE IF NOT EXISTS kairos_tick_log (
        id                 TEXT PRIMARY KEY,
        fired_at           TEXT NOT NULL,
        mode               TEXT NOT NULL,
        priority           TEXT NOT NULL,
        pending_task_count INTEGER NOT NULL DEFAULT 0,
        overdue_task_count INTEGER NOT NULL DEFAULT 0,
        triggered_work     INTEGER NOT NULL DEFAULT 0,
        session_id         TEXT
      );

      CREATE INDEX IF NOT EXISTS idx_kairos_tick_fired ON kairos_tick_log(fired_at DESC);

      -- Meta key-value store
      CREATE TABLE IF NOT EXISTS kairos_meta (
        key   TEXT PRIMARY KEY,
        value TEXT NOT NULL
      );
    `);
  }

  private assertSchemaVersion(): void {
    const stored = this.getMeta('schema_version');
    if (stored === null) {
      this.setMeta('schema_version', String(KAIROS_SCHEMA_VERSION));
    } else if (Number(stored) !== KAIROS_SCHEMA_VERSION) {
      throw new Error(
        `KairosStore: schema version mismatch. DB has v${stored}, ` +
          `code expects v${KAIROS_SCHEMA_VERSION}. Delete the DB to rebuild.`,
      );
    }
  }

  private seedDefaultSchedules(): void {
    const existing = this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM kairos_schedules')
      .get()?.count ?? 0;

    if (existing > 0) return; // Already seeded

    const tx = this.db.transaction(() => {
      for (const sched of DEFAULT_SCHEDULES) {
        this.createSchedule(sched);
      }
    });
    tx();
  }

  // ─── Row ↔ domain conversions ──────────────────────────────────────────────

  private _taskToRow(task: KairosTask): KairosTaskRow {
    return {
      id: task.id,
      title: task.title,
      description: task.description ?? null,
      status: task.status,
      priority: task.priority,
      origin: task.origin,
      deadline: task.deadline ?? null,
      snooze_until: task.snoozeUntil ?? null,
      blocker_task_id: task.blockerTaskId ?? null,
      tags_csv: task.tags.join(','),
      graphify_node: task.graphifyNode ?? null,
      wiki_slug: task.wikiSlug ?? null,
      metadata_json: JSON.stringify(task.metadata),
      session_id: task.sessionId ?? null,
      created_at: task.createdAt,
      updated_at: task.updatedAt,
      completed_at: task.completedAt ?? null,
      error_message: task.errorMessage ?? null,
      attempt_count: task.attemptCount,
      max_attempts: task.maxAttempts,
      estimated_minutes: task.estimatedMinutes ?? null,
    };
  }

  _rowToTask(row: KairosTaskRow): KairosTask {
    return {
      id: row.id,
      title: row.title,
      description: row.description ?? undefined,
      status: row.status,
      priority: row.priority,
      origin: row.origin,
      deadline: row.deadline ?? undefined,
      snoozeUntil: row.snooze_until ?? undefined,
      blockerTaskId: row.blocker_task_id ?? undefined,
      tags: row.tags_csv ? row.tags_csv.split(',').filter(Boolean) : [],
      graphifyNode: row.graphify_node ?? undefined,
      wikiSlug: row.wiki_slug ?? undefined,
      metadata: JSON.parse(row.metadata_json) as Record<string, unknown>,
      sessionId: row.session_id ?? undefined,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
      completedAt: row.completed_at ?? undefined,
      errorMessage: row.error_message ?? undefined,
      attemptCount: row.attempt_count,
      maxAttempts: row.max_attempts,
      estimatedMinutes: row.estimated_minutes ?? undefined,
    };
  }

  private _scheduleToRow(s: KairosSchedule): KairosScheduleRow {
    return {
      id: s.id,
      name: s.name,
      cron_expression: s.cronExpression,
      job_type: s.jobType,
      payload_json: JSON.stringify(s.payload),
      status: s.status,
      timezone: s.timezone,
      timeout_minutes: s.timeoutMinutes,
      last_run_at: s.lastRunAt ?? null,
      next_run_at: s.nextRunAt ?? null,
      created_at: s.createdAt,
      updated_at: s.updatedAt,
    };
  }

  _rowToSchedule(row: KairosScheduleRow): KairosSchedule {
    return {
      id: row.id,
      name: row.name,
      cronExpression: row.cron_expression,
      jobType: row.job_type,
      payload: JSON.parse(row.payload_json) as Record<string, unknown>,
      status: row.status,
      timezone: row.timezone,
      timeoutMinutes: row.timeout_minutes,
      lastRunAt: row.last_run_at ?? undefined,
      nextRunAt: row.next_run_at ?? undefined,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    };
  }
}
src/TaskQueue.ts
TypeScript

/**
 * TaskQueue — deadline-aware priority queue for @locoworker/kairos
 *
 * Responsibilities:
 *  - Compute urgency scores for KairosTask objects in real time
 *  - Maintain an in-memory min-heap ordered by urgency (most urgent first)
 *  - Expose peek/pop/push operations for the TickEngine and KairosAgent
 *  - Sync from KairosStore on demand (populate() method)
 *  - Report overdue task summaries for TemporalContext
 *
 * Design:
 *  - TaskQueue is PURE (no DB writes). It is a read-only projection
 *    of KairosStore data optimised for fast priority access.
 *  - Urgency score is intentionally NOT stored in DB — it changes
 *    every minute as deadlines approach. Always recompute at query time.
 *  - The heap invariant is maintained after every push/pop. O(log n).
 *  - TaskQueue is not thread-safe (single-threaded Node.js event loop assumed).
 *
 * Urgency score formula:
 *   total = priorityBaseScore
 *           + deadlineProximityScore   (0 if no deadline)
 *           + overdueBonus             (0 if not overdue)
 *
 * Higher score = more urgent (max theoretical: 40 + 50 + 30 = 120).
 */

import type { KairosTask, TaskUrgencyScore, KairosTaskPriority } from './types/kairos.types.js';
import {
  PRIORITY_BASE_SCORES,
  DEADLINE_IMMINENT_SCORE,
  DEADLINE_TODAY_SCORE,
  DEADLINE_THIS_WEEK_SCORE,
  OVERDUE_URGENCY_BOOST,
} from './types/kairos.types.js';

// ─── Internal heap node ───────────────────────────────────────────────────────

interface HeapNode {
  task: KairosTask;
  score: TaskUrgencyScore;
}

// ─── Urgency computation ──────────────────────────────────────────────────────

/**
 * Compute the full urgency score for a single task at the given reference time.
 * Pure function — no side effects, no I/O.
 */
export function computeUrgency(
  task: KairosTask,
  nowMs: number = Date.now(),
): TaskUrgencyScore {
  const baseScore = PRIORITY_BASE_SCORES[task.priority];

  let deadlineScore = 0;
  let overdueBonus = 0;
  let isOverdue = false;
  let hoursUntilDeadline: number | undefined;

  if (task.deadline) {
    const deadlineMs = new Date(task.deadline).getTime();
    const deltaMs = deadlineMs - nowMs;
    const deltaHours = deltaMs / (1000 * 60 * 60);
    hoursUntilDeadline = deltaHours;

    if (deltaMs < 0) {
      // Overdue
      isOverdue = true;
      overdueBonus = OVERDUE_URGENCY_BOOST;
      deadlineScore = DEADLINE_IMMINENT_SCORE; // Full proximity score when overdue
    } else if (deltaHours <= 1) {
      deadlineScore = DEADLINE_IMMINENT_SCORE;
    } else if (deltaHours <= 24) {
      deadlineScore = DEADLINE_TODAY_SCORE;
    } else if (deltaHours <= 24 * 7) {
      deadlineScore = DEADLINE_THIS_WEEK_SCORE;
    }
  }

  return {
    taskId: task.id,
    baseScore,
    deadlineScore,
    overdueBonus,
    totalScore: baseScore + deadlineScore + overdueBonus,
    isOverdue,
    hoursUntilDeadline,
  };
}

// ─── TaskQueue ────────────────────────────────────────────────────────────────

export class TaskQueue {
  private heap: HeapNode[] = [];

  /** Number of tasks currently in the queue. */
  get size(): number {
    return this.heap.length;
  }

  /** True if the queue has no tasks. */
  get isEmpty(): boolean {
    return this.heap.length === 0;
  }

  // ─── Core operations ───────────────────────────────────────────────────────

  /**
   * Push a task onto the queue, computing its urgency at the current time.
   * If the task is already in the queue (same ID), it is updated in-place
   * (urgency recomputed, heap re-heapified).
   */
  push(task: KairosTask, nowMs: number = Date.now()): void {
    const score = computeUrgency(task, nowMs);

    // Check if task already exists in heap
    const existingIdx = this.heap.findIndex((n) => n.task.id === task.id);
    if (existingIdx !== -1) {
      this.heap[existingIdx] = { task, score };
      // Re-heapify both up and down from this position
      this._heapifyUp(existingIdx);
      this._heapifyDown(existingIdx);
      return;
    }

    this.heap.push({ task, score });
    this._heapifyUp(this.heap.length - 1);
  }

  /**
   * Peek at the most urgent task without removing it.
   * Returns undefined if queue is empty.
   */
  peek(): { task: KairosTask; score: TaskUrgencyScore } | undefined {
    if (this.heap.length === 0) return undefined;
    return { task: this.heap[0].task, score: this.heap[0].score };
  }

  /**
   * Remove and return the most urgent task.
   * Returns undefined if queue is empty.
   */
  pop(): { task: KairosTask; score: TaskUrgencyScore } | undefined {
    if (this.heap.length === 0) return undefined;

    const top = this.heap[0];
    const last = this.heap.pop()!;

    if (this.heap.length > 0) {
      this.heap[0] = last;
      this._heapifyDown(0);
    }

    return { task: top.task, score: top.score };
  }

  /**
   * Remove a specific task by ID.
   * Returns true if found and removed, false otherwise.
   */
  remove(taskId: string): boolean {
    const idx = this.heap.findIndex((n) => n.task.id === taskId);
    if (idx === -1) return false;

    const last = this.heap.pop()!;
    if (idx < this.heap.length) {
      this.heap[idx] = last;
      this._heapifyUp(idx);
      this._heapifyDown(idx);
    }

    return true;
  }

  /**
   * Recompute urgency scores for ALL tasks in the queue.
   * Should be called periodically (e.g., each tick) as deadlines approach.
   */
  refresh(nowMs: number = Date.now()): void {
    for (let i = 0; i < this.heap.length; i++) {
      this.heap[i].score = computeUrgency(this.heap[i].task, nowMs);
    }
    // Rebuild heap from scratch (Floyd's algorithm) — O(n)
    for (let i = Math.floor(this.heap.length / 2) - 1; i >= 0; i--) {
      this._heapifyDown(i);
    }
  }

  /**
   * Clear all tasks from the queue.
   */
  clear(): void {
    this.heap = [];
  }

  /**
   * Populate the queue from an array of KairosTask objects.
   * Replaces any existing content (clears first).
   * Designed to be called after KairosStore.queryTasks().
   */
  populate(tasks: KairosTask[], nowMs: number = Date.now()): void {
    this.heap = tasks.map((task) => ({
      task,
      score: computeUrgency(task, nowMs),
    }));

    // Build valid heap using Floyd's heapify — O(n)
    for (let i = Math.floor(this.heap.length / 2) - 1; i >= 0; i--) {
      this._heapifyDown(i);
    }
  }

  // ─── Query helpers ─────────────────────────────────────────────────────────

  /**
   * Return the top N most urgent tasks without modifying the queue.
   * Results are sorted most-urgent first.
   */
  topN(n: number, nowMs: number = Date.now()): Array<{ task: KairosTask; score: TaskUrgencyScore }> {
    // Snapshot + refresh scores, then sort — does not mutate the real heap
    const snapshot = this.heap.map((node) => ({
      task: node.task,
      score: computeUrgency(node.task, nowMs),
    }));

    return snapshot
      .sort((a, b) => b.score.totalScore - a.score.totalScore)
      .slice(0, n);
  }

  /**
   * Return all overdue tasks (deadline < now).
   */
  getOverdueTasks(nowMs: number = Date.now()): Array<{ task: KairosTask; score: TaskUrgencyScore }> {
    return this.heap
      .filter((n) => {
        if (!n.task.deadline) return false;
        return new Date(n.task.deadline).getTime() < nowMs;
      })
      .map((n) => ({ task: n.task, score: computeUrgency(n.task, nowMs) }))
      .sort((a, b) => b.score.totalScore - a.score.totalScore);
  }

  /**
   * Return tasks with deadlines within the next N hours.
   */
  getTasksDueWithinHours(
    hours: number,
    nowMs: number = Date.now(),
  ): Array<{ task: KairosTask; score: TaskUrgencyScore }> {
    const windowEnd = nowMs + hours * 60 * 60 * 1000;
    return this.heap
      .filter((n) => {
        if (!n.task.deadline) return false;
        const deadlineMs = new Date(n.task.deadline).getTime();
        return deadlineMs >= nowMs && deadlineMs <= windowEnd;
      })
      .map((n) => ({ task: n.task, score: computeUrgency(n.task, nowMs) }))
      .sort((a, b) => {
        const da = new Date(a.task.deadline!).getTime();
        const db = new Date(b.task.deadline!).getTime();
        return da - db;
      });
  }

  /**
   * Get a summary of the queue state for TemporalContext injection.
   */
  getSummary(nowMs: number = Date.now()): {
    totalTasks: number;
    overdueTasks: number;
    tasksDueToday: number;
    criticalTasks: number;
    highTasks: number;
  } {
    let overdueTasks = 0;
    let tasksDueToday = 0;
    let criticalTasks = 0;
    let highTasks = 0;
    const todayEnd = nowMs + 24 * 60 * 60 * 1000;

    for (const node of this.heap) {
      if (node.task.deadline) {
        const deadlineMs = new Date(node.task.deadline).getTime();
        if (deadlineMs < nowMs) overdueTasks++;
        else if (deadlineMs <= todayEnd) tasksDueToday++;
      }
      if (node.task.priority === 'critical') criticalTasks++;
      if (node.task.priority === 'high') highTasks++;
    }

    return {
      totalTasks: this.heap.length,
      overdueTasks,
      tasksDueToday,
      criticalTasks,
      highTasks,
    };
  }

  /**
   * Check if a task with the given ID is in the queue.
   */
  has(taskId: string): boolean {
    return this.heap.some((n) => n.task.id === taskId);
  }

  /**
   * Get the urgency score for a specific task in the queue.
   */
  getScore(taskId: string, nowMs: number = Date.now()): TaskUrgencyScore | undefined {
    const node = this.heap.find((n) => n.task.id === taskId);
    if (!node) return undefined;
    return computeUrgency(node.task, nowMs);
  }

  // ─── Min-heap implementation ───────────────────────────────────────────────
  // We want the MOST urgent task at the top, so we use a MAX-heap on totalScore.

  private _heapifyUp(idx: number): void {
    while (idx > 0) {
      const parentIdx = Math.floor((idx - 1) / 2);
      if (this._compare(idx, parentIdx) > 0) {
        this._swap(idx, parentIdx);
        idx = parentIdx;
      } else {
        break;
      }
    }
  }

  private _heapifyDown(idx: number): void {
    const n = this.heap.length;
    while (true) {
      const left = 2 * idx + 1;
      const right = 2 * idx + 2;
      let largest = idx;

      if (left < n && this._compare(left, largest) > 0) largest = left;
      if (right < n && this._compare(right, largest) > 0) largest = right;

      if (largest !== idx) {
        this._swap(idx, largest);
        idx = largest;
      } else {
        break;
      }
    }
  }

  /**
   * Compare two heap nodes: returns positive if `a` is more urgent than `b`.
   * Tiebreaking order:
   *  1. Higher totalScore wins
   *  2. Earlier deadline wins (more urgent)
   *  3. Earlier createdAt wins (older tasks first as tiebreaker)
   */
  private _compare(aIdx: number, bIdx: number): number {
    const a = this.heap[aIdx].score;
    const b = this.heap[bIdx].score;

    if (a.totalScore !== b.totalScore) return a.totalScore - b.totalScore;

    // Both have deadlines → earlier deadline is more urgent
    const aDeadline = this.heap[aIdx].task.deadline;
    const bDeadline = this.heap[bIdx].task.deadline;

    if (aDeadline && bDeadline) {
      const diff = new Date(aDeadline).getTime() - new Date(bDeadline).getTime();
      if (diff !== 0) return -diff; // Earlier deadline = more urgent = larger in max-heap
    } else if (aDeadline && !bDeadline) {
      return 1; // Has deadline → more urgent
    } else if (!aDeadline && bDeadline) {
      return -1;
    }

    // Tiebreak: older task first
    return (
      new Date(this.heap[bIdx].task.createdAt).getTime() -
      new Date(this.heap[aIdx].task.createdAt).getTime()
    );
  }

  private _swap(i: number, j: number): void {
    const tmp = this.heap[i];
    this.heap[i] = this.heap[j];
    this.heap[j] = tmp;
  }
}
src/CronScheduler.ts
TypeScript

/**
 * CronScheduler — pure-TypeScript cron expression parser and schedule evaluator
 *                 for @locoworker/kairos
 *
 * Zero external dependencies. Implements standard 5-field cron:
 *   minute (0–59) | hour (0–23) | day-of-month (1–31) | month (1–12) | day-of-week (0–6, 0=Sunday)
 *
 * Supported syntax:
 *   *          → wildcard (all values in range)
 *   ,          → list (e.g. 1,3,5)
 *   -          → range (e.g. 9-17)
 *   /          → step (e.g. */5, 0-30/2)
 *
 * Special strings (case-insensitive):
 *   @yearly  / @annually = "0 0 1 1 *"
 *   @monthly             = "0 0 1 * *"
 *   @weekly              = "0 0 * * 0"
 *   @daily   / @midnight = "0 0 * * *"
 *   @hourly              = "0 * * * *"
 *
 * Responsibilities:
 *  - Parse cron expressions into structured CronField representations
 *  - Validate expressions (throw on invalid syntax)
 *  - Compute the next run time after a given reference date
 *  - Check if a given moment matches a cron expression
 *  - Compute due schedules from a list of KairosSchedule objects
 *  - Update next_run_at on schedule records
 *
 * Design:
 *  - CronScheduler is stateless (no DB dependency, no timers).
 *  - It computes; KairosStore persists; TickEngine drives.
 *  - All date arithmetic uses UTC (timezone-aware schedules are
 *    handled by converting local time to UTC before evaluation).
 */

import type {
  KairosSchedule,
  KairosScheduleRow,
  ParsedCronExpression,
  CronField,
} from './types/kairos.types.js';

// ─── Special string aliases ───────────────────────────────────────────────────

const CRON_ALIASES: Record<string, string> = {
  '@yearly':    '0 0 1 1 *',
  '@annually':  '0 0 1 1 *',
  '@monthly':   '0 0 1 * *',
  '@weekly':    '0 0 * * 0',
  '@daily':     '0 0 * * *',
  '@midnight':  '0 0 * * *',
  '@hourly':    '0 * * * *',
};

// ─── Field range constraints ──────────────────────────────────────────────────

const FIELD_RANGES = [
  { name: 'minute',     min: 0,  max: 59 },
  { name: 'hour',       min: 0,  max: 23 },
  { name: 'dayOfMonth', min: 1,  max: 31 },
  { name: 'month',      min: 1,  max: 12 },
  { name: 'dayOfWeek',  min: 0,  max: 6  },
] as const;

// Named month and day-of-week aliases
const MONTH_NAMES: Record<string, number> = {
  jan: 1, feb: 2, mar: 3, apr: 4, may: 5, jun: 6,
  jul: 7, aug: 8, sep: 9, oct: 10, nov: 11, dec: 12,
};

const DOW_NAMES: Record<string, number> = {
  sun: 0, mon: 1, tue: 2, wed: 3, thu: 4, fri: 5, sat: 6,
};

// ─── CronParser (private implementation) ─────────────────────────────────────

/**
 * Parse a single cron field string into a sorted array of matching values.
 * e.g. "*/15" on minute → [0, 15, 30, 45]
 */
function parseField(
  raw: string,
  min: number,
  max: number,
  aliases?: Record<string, number>,
): CronField {
  const normalised = raw.toLowerCase();

  // Wildcard
  if (normalised === '*' || normalised === '?') {
    const values: number[] = [];
    for (let i = min; i <= max; i++) values.push(i);
    return { raw, values, isWildcard: true };
  }

  const values = new Set<number>();

  // Comma-separated list
  const parts = normalised.split(',');

  for (const part of parts) {
    // Step: */n or range/n
    const stepMatch = part.match(/^(.+)\/(\d+)$/);
    if (stepMatch) {
      const [, rangePart, stepStr] = stepMatch;
      const step = parseInt(stepStr, 10);

      if (step <= 0) throw new Error(`Invalid step value ${step} in cron field "${raw}"`);

      let rangeMin = min;
      let rangeMax = max;

      if (rangePart !== '*' && rangePart !== '?') {
        const dashParts = rangePart.split('-');
        if (dashParts.length !== 2) {
          throw new Error(`Invalid range "${rangePart}" in cron field "${raw}"`);
        }
        rangeMin = resolveAlias(dashParts[0], aliases) ?? parseInt(dashParts[0], 10);
        rangeMax = resolveAlias(dashParts[1], aliases) ?? parseInt(dashParts[1], 10);
      }

      for (let v = rangeMin; v <= rangeMax; v += step) {
        validateAndAdd(v, min, max, raw, values);
      }
      continue;
    }

    // Range: a-b
    const dashParts = part.split('-');
    if (dashParts.length === 2) {
      const rangeMin = resolveAlias(dashParts[0], aliases) ?? parseInt(dashParts[0], 10);
      const rangeMax = resolveAlias(dashParts[1], aliases) ?? parseInt(dashParts[1], 10);
      for (let v = rangeMin; v <= rangeMax; v++) {
        validateAndAdd(v, min, max, raw, values);
      }
      continue;
    }

    // Single value (possibly a named alias)
    const num = resolveAlias(part, aliases) ?? parseInt(part, 10);
    if (isNaN(num)) throw new Error(`Invalid cron field value "${part}" in "${raw}"`);
    validateAndAdd(num, min, max, raw, values);
  }

  return {
    raw,
    values: [...values].sort((a, b) => a - b),
    isWildcard: false,
  };
}

function resolveAlias(
  value: string,
  aliases?: Record<string, number>,
): number | undefined {
  return aliases?.[value.toLowerCase()];
}

function validateAndAdd(
  v: number,
  min: number,
  max: number,
  raw: string,
  set: Set<number>,
): void {
  if (isNaN(v) || v < min || v > max) {
    throw new Error(
      `Cron field value ${v} is out of range [${min}, ${max}] in "${raw}"`,
    );
  }
  set.add(v);
}

// ─── CronScheduler ────────────────────────────────────────────────────────────

export class CronScheduler {
  // ─── Parsing ──────────────────────────────────────────────────────────────

  /**
   * Parse a cron expression string into a structured ParsedCronExpression.
   * Throws on invalid syntax.
   */
  parse(expression: string): ParsedCronExpression {
    let expr = expression.trim();

    // Resolve alias
    const alias = CRON_ALIASES[expr.toLowerCase()];
    if (alias) expr = alias;

    const parts = expr.split(/\s+/);
    if (parts.length !== 5) {
      throw new Error(
        `Invalid cron expression "${expression}": must have exactly 5 fields ` +
          `(minute hour day-of-month month day-of-week), got ${parts.length}.`,
      );
    }

    const [minuteRaw, hourRaw, domRaw, monthRaw, dowRaw] = parts;

    return {
      raw: expression,
      minute:     parseField(minuteRaw, 0, 59),
      hour:       parseField(hourRaw,   0, 23),
      dayOfMonth: parseField(domRaw,    1, 31),
      month:      parseField(monthRaw,  1, 12, MONTH_NAMES),
      dayOfWeek:  parseField(dowRaw,    0,  6, DOW_NAMES),
    };
  }

  /**
   * Validate a cron expression without throwing (returns error string or null).
   */
  validate(expression: string): string | null {
    try {
      this.parse(expression);
      return null;
    } catch (err) {
      return err instanceof Error ? err.message : String(err);
    }
  }

  // ─── Matching ─────────────────────────────────────────────────────────────

  /**
   * Check whether a given Date matches a parsed cron expression.
   * Uses UTC fields for evaluation.
   */
  matches(parsed: ParsedCronExpression, date: Date): boolean {
    const minute     = date.getUTCMinutes();
    const hour       = date.getUTCHours();
    const dayOfMonth = date.getUTCDate();
    const month      = date.getUTCMonth() + 1; // JS months are 0-indexed
    const dayOfWeek  = date.getUTCDay();

    return (
      parsed.minute.values.includes(minute) &&
      parsed.hour.values.includes(hour) &&
      parsed.dayOfMonth.values.includes(dayOfMonth) &&
      parsed.month.values.includes(month) &&
      parsed.dayOfWeek.values.includes(dayOfWeek)
    );
  }

  // ─── Next run computation ─────────────────────────────────────────────────

  /**
   * Compute the next Date at which a cron expression will fire,
   * starting from `after` (exclusive, default: now).
   *
   * Implementation: minute-level iteration starting from `after + 1 minute`.
   * Bounded to 4 years of search (handles leap years, end-of-month edge cases).
   * Returns null if no match found within the search window (shouldn't happen
   * for valid expressions, but guards against pathological inputs).
   */
  nextDate(expression: string, after: Date = new Date()): Date | null {
    const parsed = this.parse(expression);

    // Start searching from the next minute boundary
    const start = new Date(after);
    start.setUTCSeconds(0, 0);  // Zero out sub-minute precision
    start.setUTCMinutes(start.getUTCMinutes() + 1);  // Advance at least one minute

    const MAX_ITERATIONS = 60 * 24 * 366 * 4; // ~4 years of minutes
    const candidate = new Date(start);

    for (let i = 0; i < MAX_ITERATIONS; i++) {
      if (this.matches(parsed, candidate)) {
        return new Date(candidate);
      }

      // Advance by one minute
      candidate.setUTCMinutes(candidate.getUTCMinutes() + 1);

      // Fast-forward: if current hour not in expression, jump to next valid hour
      if (!parsed.hour.values.includes(candidate.getUTCHours())) {
        const nextHour = parsed.hour.values.find((h) => h > candidate.getUTCHours());
        if (nextHour !== undefined) {
          candidate.setUTCHours(nextHour, 0, 0, 0);
        } else {
          // Jump to next day, first valid hour
          candidate.setUTCDate(candidate.getUTCDate() + 1);
          candidate.setUTCHours(parsed.hour.values[0], 0, 0, 0);
        }
      }

      // Fast-forward: if current month not in expression, jump to next valid month
      const currentMonth = candidate.getUTCMonth() + 1;
      if (!parsed.month.values.includes(currentMonth)) {
        const nextMonth = parsed.month.values.find((m) => m > currentMonth);
        if (nextMonth !== undefined) {
          candidate.setUTCMonth(nextMonth - 1, 1);
          candidate.setUTCHours(parsed.hour.values[0], 0, 0, 0);
        } else {
          candidate.setUTCFullYear(candidate.getUTCFullYear() + 1);
          candidate.setUTCMonth(parsed.month.values[0] - 1, 1);
          candidate.setUTCHours(parsed.hour.values[0], 0, 0, 0);
        }
      }
    }

    return null; // Should not be reached for valid expressions
  }

  /**
   * Get the next N future run dates for a cron expression.
   * Useful for schedule previews and UI display.
   */
  nextNDates(expression: string, n: number, after: Date = new Date()): Date[] {
    const dates: Date[] = [];
    let cursor = after;

    for (let i = 0; i < n; i++) {
      const next = this.nextDate(expression, cursor);
      if (!next) break;
      dates.push(next);
      cursor = next;
    }

    return dates;
  }

  // ─── Schedule evaluation ──────────────────────────────────────────────────

  /**
   * Given an array of KairosSchedule objects, return those that are DUE
   * (nextRunAt <= now) and have status === 'active'.
   *
   * "Due" means: nextRunAt is in the past or is null (never run before, and
   * the cron expression would have fired at least once since creation).
   */
  getDueSchedules(
    schedules: KairosSchedule[],
    nowMs: number = Date.now(),
  ): KairosSchedule[] {
    const now = new Date(nowMs);

    return schedules.filter((s) => {
      if (s.status !== 'active') return false;

      // If nextRunAt is set, check if it's due
      if (s.nextRunAt) {
        return new Date(s.nextRunAt).getTime() <= nowMs;
      }

      // No nextRunAt yet — check if the cron would have fired since creation
      try {
        const firstRun = this.nextDate(s.cronExpression, new Date(s.createdAt));
        return firstRun !== null && firstRun.getTime() <= nowMs;
      } catch {
        return false;
      }
    });
  }

  /**
   * Compute and return the next run ISO string for a schedule.
   * Returns null if the expression is invalid.
   */
  computeNextRunAt(
    cronExpression: string,
    after: Date = new Date(),
  ): string | null {
    try {
      const next = this.nextDate(cronExpression, after);
      return next ? next.toISOString() : null;
    } catch {
      return null;
    }
  }

  /**
   * Update the next_run_at for a batch of schedules.
   * Returns updated schedules (does NOT write to DB — caller handles persistence).
   */
  refreshNextRunTimes(
    schedules: KairosSchedule[],
    after: Date = new Date(),
  ): KairosSchedule[] {
    return schedules.map((s) => ({
      ...s,
      nextRunAt: this.computeNextRunAt(s.cronExpression, after) ?? s.nextRunAt,
    }));
  }

  // ─── Human-readable descriptions ─────────────────────────────────────────

  /**
   * Generate a human-readable description of a cron expression.
   * e.g. "0 2 * * *" → "Every day at 02:00 UTC"
   * e.g. "*/15 * * * *" → "Every 15 minutes"
   * e.g. "0 9 * * 1-5" → "Every weekday at 09:00 UTC"
   */
  describe(expression: string): string {
    // Resolve aliases first
    let expr = expression.trim();
    const alias = CRON_ALIASES[expr.toLowerCase()];
    if (alias) expr = alias;

    let parsed: ParsedCronExpression;
    try {
      parsed = this.parse(expr);
    } catch {
      return `Invalid expression: ${expression}`;
    }

    const { minute, hour, dayOfMonth, month, dayOfWeek } = parsed;

    // Common patterns
    if (
      minute.isWildcard && hour.isWildcard && dayOfMonth.isWildcard &&
      month.isWildcard && dayOfWeek.isWildcard
    ) {
      return 'Every minute';
    }

    if (
      !minute.isWildcard && minute.values.length > 1 &&
      hour.isWildcard && dayOfMonth.isWildcard &&
      month.isWildcard && dayOfWeek.isWildcard
    ) {
      // e.g. */15 * * * * — step on minute
      const raw = minute.raw;
      const stepMatch = raw.match(/^\*\/(\d+)$/);
      if (stepMatch) return `Every ${stepMatch[1]} minutes`;
    }

    if (
      minute.values.length === 1 && !hour.isWildcard && hour.values.length === 1 &&
      dayOfMonth.isWildcard && month.isWildcard && dayOfWeek.isWildcard
    ) {
      return `Every day at ${pad(hour.values[0])}:${pad(minute.values[0])} UTC`;
    }

    if (
      minute.values.length === 1 && !hour.isWildcard && hour.values.length === 1 &&
      dayOfMonth.isWildcard && month.isWildcard && !dayOfWeek.isWildcard
    ) {
      const days = dayOfWeek.values.map(dowName).join(', ');
      return `Every ${days} at ${pad(hour.values[0])}:${pad(minute.values[0])} UTC`;
    }

    if (
      minute.values.length === 1 && hour.isWildcard &&
      dayOfMonth.isWildcard && month.isWildcard && dayOfWeek.isWildcard
    ) {
      return `Every hour at minute ${minute.values[0]}`;
    }

    // Generic fallback
    return `Cron: ${expression}`;
  }
}

// ─── Helpers ──────────────────────────────────────────────────────────────────

function pad(n: number): string {
  return String(n).padStart(2, '0');
}

const DOW_ABBR = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];
function dowName(n: number): string {
  return DOW_ABBR[n] ?? String(n);
}
src/TemporalContext.ts
TypeScript

/**
 * TemporalContext — time-awareness injection for @locoworker/kairos
 *
 * Responsibilities:
 *  - Assemble a TemporalContextSnapshot from live KairosStore data
 *  - Format the snapshot as a compact system-prompt section
 *    (injected into TurnAssembler in packages/core)
 *  - Determine working-hours / overnight-window status
 *  - Surface urgent tasks and next scheduled job to the agent
 *  - Enforce a configurable token budget for the temporal block
 *  - Parse [OBSERVE: ...] markers from agent messages and create observations
 *
 * Design:
 *  - TemporalContext is read-only with respect to KairosStore
 *    (it reads tasks/schedules/observations; never writes).
 *  - The only write operation is appendObservation() — delegated to KairosStore
 *    by the caller (KairosAgent or WikiSyncAgent).
 *  - All local-time arithmetic uses Intl.DateTimeFormat for timezone safety
 *    (no manual UTC offset arithmetic).
 *  - The formatted context block is designed to be token-efficient:
 *    a rich snapshot costs ~200–350 tokens at most.
 *
 * Integration:
 *  - packages/core TurnAssembler calls TemporalContext.format()
 *    to get the temporal block string, then appends it to the system prompt.
 *  - packages/core HooksRegistry can register a beforeTurn hook that calls
 *    TemporalContext.extractObservations() to pull [OBSERVE:] markers.
 */

import type { KairosStore } from './KairosStore.js';
import type { KairosTask, KairosSchedule } from './types/kairos.types.js';
import { TaskQueue, computeUrgency } from './TaskQueue.js';
import { CronScheduler } from './CronScheduler.js';
import type {
  TemporalContextSnapshot,
  TemporalContextOptions,
  KairosObservation,
  ObservationSource,
} from './types/kairos.types.js';
import {
  DEFAULT_WORKING_HOURS,
  DEFAULT_OVERNIGHT_WINDOW,
  MAX_URGENT_TASKS_PER_TICK,
  CHARS_PER_TOKEN,
  OBSERVE_MARKER_REGEX,
} from './types/kairos.types.js';

// ─── Internal helpers ─────────────────────────────────────────────────────────

/**
 * Get the local hour (0–23) in the given timezone using Intl.
 * Falls back to system local if timezone is invalid.
 */
function getLocalHour(date: Date, timezone?: string): number {
  try {
    const fmt = new Intl.DateTimeFormat('en-US', {
      hour: 'numeric',
      hour12: false,
      timeZone: timezone ?? Intl.DateTimeFormat().resolvedOptions().timeZone,
    });
    const parts = fmt.formatToParts(date);
    const hourPart = parts.find((p) => p.type === 'hour');
    return hourPart ? parseInt(hourPart.value, 10) % 24 : date.getHours();
  } catch {
    return date.getHours();
  }
}

/**
 * Get the local date string (YYYY-MM-DD) in the given timezone.
 */
function getLocalDateString(date: Date, timezone?: string): string {
  try {
    const fmt = new Intl.DateTimeFormat('en-CA', {
      year: 'numeric', month: '2-digit', day: '2-digit',
      timeZone: timezone ?? Intl.DateTimeFormat().resolvedOptions().timeZone,
    });
    return fmt.format(date);
  } catch {
    return date.toISOString().slice(0, 10);
  }
}

/**
 * Format a Date to a human-friendly local time string.
 */
function formatLocalTime(date: Date, timezone?: string): string {
  try {
    return date.toLocaleString('en-US', {
      weekday: 'short',
      month: 'short',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
      hour12: true,
      timeZone: timezone,
      timeZoneName: 'short',
    });
  } catch {
    return date.toLocaleString();
  }
}

/**
 * Check if `hour` falls within an overnight window that may span midnight.
 * e.g. { start: 22, end: 6 } covers 22:00–23:59 and 00:00–05:59.
 */
function isInWindow(hour: number, window: { start: number; end: number }): boolean {
  if (window.start < window.end) {
    // e.g. 8–18 (doesn't cross midnight)
    return hour >= window.start && hour < window.end;
  } else {
    // e.g. 22–6 (crosses midnight)
    return hour >= window.start || hour < window.end;
  }
}

const DAY_NAMES = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];

// ─── TemporalContext ──────────────────────────────────────────────────────────

export class TemporalContext {
  private readonly scheduler: CronScheduler;

  constructor(private readonly store: KairosStore) {
    this.scheduler = new CronScheduler();
  }

  // ─── Snapshot assembly ─────────────────────────────────────────────────────

  /**
   * Assemble the full TemporalContextSnapshot.
   * Reads from KairosStore: pending tasks, active schedules, observation count.
   */
  assembleSnapshot(options: TemporalContextOptions = {}): TemporalContextSnapshot {
    const now = new Date();
    const nowMs = now.getTime();
    const timezone = options.timezone;
    const workingHours = options.workingHours ?? DEFAULT_WORKING_HOURS;
    const overnightWindow = options.overnightWindow ?? DEFAULT_OVERNIGHT_WINDOW;
    const maxUrgentTasks = options.maxUrgentTasks ?? MAX_URGENT_TASKS_PER_TICK;
    const maxTokens = options.maxTokens ?? 400;

    // Time fields
    const localHour = getLocalHour(now, timezone);
    const dayOfWeek = now.getUTCDay();
    const isWorkingHours = isInWindow(localHour, workingHours);
    const isOvernightWindow = isInWindow(localHour, overnightWindow);

    // Task data — query pending + running tasks
    const taskRows = this.store.queryTasks({
      status: ['pending', 'running'],
      sortByUrgency: true,
      limit: 200,
    });

    const tasks: KairosTask[] = taskRows.map((r) => this.store._rowToTask(r));

    // Build a temporary queue for urgency scoring
    const queue = new TaskQueue();
    queue.populate(tasks, nowMs);
    const summary = queue.getSummary(nowMs);

    // Top urgent tasks for context injection
    const topTasks = queue.topN(maxUrgentTasks, nowMs);
    const urgentTasks = topTasks.map(({ task, score }) => ({
      id: task.id,
      title: task.title,
      priority: task.priority,
      deadline: task.deadline,
      isOverdue: score.isOverdue,
      hoursUntilDeadline: score.hoursUntilDeadline,
    }));

    // Next scheduled job
    let nextScheduledJob: TemporalContextSnapshot['nextScheduledJob'] | undefined;
    if (options.includeNextJob !== false) {
      const scheduleRows = this.store.getAllSchedules('active');
      const schedules: KairosSchedule[] = scheduleRows.map((r) =>
        this.store._rowToSchedule(r),
      );

      // Find schedule with the soonest nextRunAt
      const upcoming = schedules
        .filter((s) => s.nextRunAt)
        .sort(
          (a, b) =>
            new Date(a.nextRunAt!).getTime() - new Date(b.nextRunAt!).getTime(),
        );

      if (upcoming.length > 0) {
        const next = upcoming[0];
        const nextRunMs = new Date(next.nextRunAt!).getTime();
        const minutesUntilRun = Math.round((nextRunMs - nowMs) / 60_000);
        nextScheduledJob = {
          name: next.name,
          jobType: next.jobType,
          nextRunAt: next.nextRunAt!,
          minutesUntilRun,
        };
      }
    }

    // Today's observation count
    const todayObservationCount =
      options.includeObservationCount !== false
        ? this.store.getTodayObservationCount()
        : 0;

    const snapshot: TemporalContextSnapshot = {
      nowUtc: now.toISOString(),
      nowLocal: formatLocalTime(now, timezone),
      dayOfWeek,
      dayName: DAY_NAMES[dayOfWeek],
      hourOfDay: localHour,
      isWorkingHours,
      isOvernightWindow,
      overdueTasks: summary.overdueTasks,
      tasksDueToday: summary.tasksDueToday,
      urgentTasks,
      nextScheduledJob,
      todayObservationCount,
      estimatedTokens: 0, // computed below
    };

    // Estimate tokens for the formatted block
    const formatted = this.format(snapshot, { maxTokens: Infinity });
    snapshot.estimatedTokens = Math.ceil(formatted.length / CHARS_PER_TOKEN);

    return snapshot;
  }

  // ─── Prompt formatting ─────────────────────────────────────────────────────

  /**
   * Format a TemporalContextSnapshot into a compact system-prompt section.
   *
   * Output format (designed to be token-efficient):
   * ---
   * <temporal_context>
   * Time: Wed, May 7, 2026, 09:15 AM EDT | Working hours | Day 3 of week
   * Tasks: 12 pending · 2 overdue · 1 due today
   * Urgent: [high] Refactor ToolRegistry (due in 3h) | [critical] Fix auth bug (OVERDUE 2h)
   * Next job: Nightly AutoDream in 17h 45m
   * Observations today: 4
   * </temporal_context>
   * ---
   */
  format(
    snapshot: TemporalContextSnapshot,
    options: { maxTokens?: number } = {},
  ): string {
    const maxTokens = options.maxTokens ?? 400;
    const lines: string[] = [];

    // Time line
    const timeMode = snapshot.isOvernightWindow
      ? '🌙 Overnight window'
      : snapshot.isWorkingHours
        ? '🟢 Working hours'
        : '🔵 Off hours';

    lines.push(`Time: ${snapshot.nowLocal} | ${timeMode}`);

    // Task summary
    const taskParts = [`${snapshot.urgentTasks.length + (snapshot.overdueTasks > snapshot.urgentTasks.filter((t) => t.isOverdue).length ? snapshot.overdueTasks - snapshot.urgentTasks.filter((t) => t.isOverdue).length : 0)} pending`];
    if (snapshot.overdueTasks > 0) taskParts.push(`⚠ ${snapshot.overdueTasks} overdue`);
    if (snapshot.tasksDueToday > 0) taskParts.push(`${snapshot.tasksDueToday} due today`);
    lines.push(`Tasks: ${taskParts.join(' · ')}`);

    // Urgent task list (one per line, trimmed for budget)
    if (snapshot.urgentTasks.length > 0) {
      const urgentLines = snapshot.urgentTasks.map((t) => {
        const deadlineStr = t.isOverdue
          ? `OVERDUE ${Math.abs(Math.round(t.hoursUntilDeadline ?? 0))}h`
          : t.hoursUntilDeadline !== undefined
            ? `due in ${Math.round(t.hoursUntilDeadline)}h`
            : '';
        const marker = t.isOverdue ? '⚠' : '';
        const title = t.title.length > 50 ? t.title.slice(0, 47) + '...' : t.title;
        return `  ${marker}[${t.priority}] ${title}${deadlineStr ? ` (${deadlineStr})` : ''}`;
      });
      lines.push(`Urgent:\n${urgentLines.join('\n')}`);
    }

    // Next scheduled job
    if (snapshot.nextScheduledJob) {
      const j = snapshot.nextScheduledJob;
      const eta =
        j.minutesUntilRun < 60
          ? `in ${j.minutesUntilRun}m`
          : `in ${Math.round(j.minutesUntilRun / 60)}h ${j.minutesUntilRun % 60}m`;
      lines.push(`Next job: ${j.name} ${eta}`);
    }

    // Observations today
    if (snapshot.todayObservationCount > 0) {
      lines.push(`Observations today: ${snapshot.todayObservationCount}`);
    }

    const body = lines.filter(Boolean).join('\n');
    const block = `<temporal_context>\n${body}\n</temporal_context>`;

    // Truncate if over budget
    const estimatedTokens = Math.ceil(block.length / CHARS_PER_TOKEN);
    if (estimatedTokens > maxTokens) {
      // Return a minimal version
      return [
        '<temporal_context>',
        `Time: ${snapshot.nowUtc} | Tasks: ${snapshot.overdueTasks} overdue`,
        '</temporal_context>',
      ].join('\n');
    }

    return block;
  }

  // ─── [OBSERVE: ...] marker extraction ─────────────────────────────────────

  /**
   * Scan a message string for [OBSERVE: content] markers and return
   * structured observation inputs ready for KairosStore.appendObservation().
   *
   * Syntax:
   *   [OBSERVE: This is an observation]              → importance 3
   *   [OBSERVE:5: Critical finding about auth]       → importance 5
   *   [OBSERVE:1: Minor note about config change]    → importance 1
   *
   * Returns an array of observation inputs (caller calls appendObservation).
   */
  extractObservations(
    messageContent: string,
    source: ObservationSource = 'agent_insight',
    sessionId?: string,
  ): Array<Omit<KairosObservation, 'id' | 'createdAt' | 'consolidated'>> {
    const observations: Array<Omit<KairosObservation, 'id' | 'createdAt' | 'consolidated'>> = [];
    const now = new Date();
    const dateBucket = now.toISOString().slice(0, 10);

    OBSERVE_MARKER_REGEX.lastIndex = 0;
    let match: RegExpExecArray | null;

    while ((match = OBSERVE_MARKER_REGEX.exec(messageContent)) !== null) {
      const importanceStr = match[1];
      const content = match[2].trim();

      if (!content) continue;

      const importance = importanceStr
        ? (Math.min(5, Math.max(1, parseInt(importanceStr, 10))) as 1 | 2 | 3 | 4 | 5)
        : 3;

      observations.push({
        content,
        source,
        dateBucket,
        sessionId,
        tags: [],
        importance,
      });
    }

    return observations;
  }

  // ─── Context string for TurnAssembler ─────────────────────────────────────

  /**
   * One-shot method: assemble snapshot + format.
   * This is the method called by TurnAssembler in packages/core.
   *
   * Returns a ready-to-inject string (or empty string if budget is too tight).
   */
  getContextString(options: TemporalContextOptions = {}): string {
    const maxTokens = options.maxTokens ?? 400;

    try {
      const snapshot = this.assembleSnapshot(options);

      if (snapshot.estimatedTokens > maxTokens) {
        // Return minimal version
        return this.format(snapshot, { maxTokens: 80 });
      }

      return this.format(snapshot);
    } catch (err) {
      // Never crash the agent loop due to temporal context errors
      const now = new Date().toISOString();
      return `<temporal_context>\nTime: ${now}\n</temporal_context>`;
    }
  }

  // ─── Scheduling helper ────────────────────────────────────────────────────

  /**
   * Update nextRunAt for all active schedules and persist via KairosStore.
   * Called on startup and after each schedule run.
   */
  refreshAllScheduleNextRuns(after: Date = new Date()): void {
    const scheduleRows = this.store.getAllSchedules('active');

    for (const row of scheduleRows) {
      const nextRunAt = this.scheduler.computeNextRunAt(row.cron_expression, after);
      if (nextRunAt && nextRunAt !== row.next_run_at) {
        this.store.updateSchedule(row.id, {
          nextRunAt,
          updatedAt: new Date().toISOString(),
        });
      }
    }
  }

  // ─── Overnight / working-hours helpers (public API) ──────────────────────

  /**
   * Check if now is within working hours (for scheduling proactive work).
   */
  isWorkingHours(options: Pick<TemporalContextOptions, 'workingHours' | 'timezone'> = {}): boolean {
    const hour = getLocalHour(new Date(), options.timezone);
    return isInWindow(hour, options.workingHours ?? DEFAULT_WORKING_HOURS);
  }

  /**
   * Check if now is in the overnight window (for triggering AutoDream etc).
   */
  isOvernightWindow(
    options: Pick<TemporalContextOptions, 'overnightWindow' | 'timezone'> = {},
  ): boolean {
    const hour = getLocalHour(new Date(), options.timezone);
    return isInWindow(hour, options.overnightWindow ?? DEFAULT_OVERNIGHT_WINDOW);
  }
}
Pass 6 — Part 1 Complete ✅
What was generated
File	Purpose	Lines (approx)
package.json	Package config: better-sqlite3, zod deps; @locoworker/memory + @locoworker/wiki as optional peers	45
tsconfig.json	Extends monorepo base; path aliases for core/shared/memory/wiki	22
src/types/kairos.types.ts	Full type ABI: tasks, schedules, ticks, observations, Zod schemas, urgency scoring constants, OBSERVE_MARKER_REGEX, default schedules seed	~370
src/KairosStore.ts	SQLite persistence — 6-table schema (tasks/schedules/runs/observations/tick_log/meta), WAL pragmas, full CRUD, urgency-sorted task queries, observation append, tick logging, snooze wakeup, blocker resolution, auto-fail exhausted, stats	~490
src/TaskQueue.ts	Pure in-memory max-heap priority queue — computeUrgency() pure function, push/pop/peek/remove/refresh/topN/getOverdueTasks/getSummary, O(log n) heap ops, deadline tiebreaking	~280
src/CronScheduler.ts	Zero-dependency cron parser + evaluator — 5-field parser, step/range/list/alias syntax, nextDate() with fast-forward optimization, getDueSchedules(), describe() human-readable output, @yearly/@daily etc. aliases	~340
src/TemporalContext.ts	Time-awareness injection — assembleSnapshot(), token-budgeted format() → <temporal_context> block, [OBSERVE:] marker extraction, getContextString() for TurnAssembler, refreshAllScheduleNextRuns(), working-hours + overnight-window helpers	~320
Part 2 preview (ready to generate on request)
File	Purpose
src/TickEngine.ts	Proactive tick loop — interruptible setTimeout(0) pattern (KAIROS mode), sleep/wake with cost trade-offs, tick content <tick>HH:MM:SS</tick>, queue-empties-then-reschedules pattern
src/ObservationLog.ts	Daily append-only observation log — date-bucketed queries, consolidation pipeline interface for AutoDream, today/week/month summaries
src/KairosAgent.ts	Agent integration — attaches TickEngine to core EventBus, surfaces overdue tasks in prompts, triggers background jobs, writes session-end observations, snooze/complete/create task tools
src/KairosReporter.ts	Generates KAIROS_REPORT.md — schedule calendar, task burndown, observation daily digest, overdue alerts
src/index.ts	Clean barrel export
src/tests/KairosStore.test.ts	Full bun test suite
src/tests/TaskQueue.test.ts	Full bun test suite
src/tests/CronScheduler.test.ts	Full bun test suite
src/tests/TemporalContext.test.ts	Full bun test suite
Key design decisions & cross-package consistency notes
Decision	Rationale
KAIROS tick pattern from Claude Code leak	<tick>HH:MM:SS</tick> content format, priority: 'later' for non-urgent ticks, setTimeout(0) interruptibility — all directly mirror the published KAIROS source
DB column naming follows established convention	tags_csv, session_id, created_at / updated_at — identical to WikiStore, GraphBuilder
Foreign keys with ON DELETE SET NULL	Deleting a blocker task doesn't cascade-delete dependent tasks — just clears the blocker reference (important for task resilience)
computeUrgency() is pure + exported	Tests and external callers can compute urgency without instantiating a full queue; also makes the scoring formula inspectable
CronScheduler has zero deps	No cron-parser npm package — the implementation is ~340 lines of pure TS, avoids supply-chain risk for a scheduling-critical component
TemporalContext.getContextString() never throws	The try/catch fallback ensures a bad temporal state never crashes the agent's queryLoop in packages/core
[OBSERVE:] markers parallel [REMEMBER:] in packages/memory	Consistent agent control surface across all knowledge-capture packages — both use [MARKER: content] syntax
Default schedules seeded on first init	AutoDream (2am), Graph Rebuild (2:30am), Wiki Export (Sunday 3am), Memory Compact (hourly) — matches AutoDream scheduling described in pass3.md
TaskQueue is a pure projection	No DB writes — urgency is always recomputed at query time. This means the queue is always fresh and never out of sync with approaching deadlines




Pass 6 — packages/kairos (Part 2 of 2)
Continuing from Part 1. This pass completes @locoworker/kairos with the proactive tick engine, observation log pipeline, agent integration bridge, markdown reporter, barrel export, and all four fully runnable bun test test suites.

src/TickEngine.ts
TypeScript

/**
 * TickEngine — proactive background tick loop for @locoworker/kairos
 *
 * This is the direct LocoWorker re-implementation of the KAIROS tick
 * pattern discovered in the Claude Code sourcemap incident (March 2026).
 *
 * The original KAIROS pattern (from the leak):
 *  - A background loop fires `<tick>HH:MM:SS</tick>` messages into the
 *    agent's message queue at a configurable interval.
 *  - Ticks carry a priority hint: 'now' | 'soon' | 'later'.
 *  - The agent decides whether to act proactively based on the tick content
 *    and the task queue state at the moment of the tick.
 *  - The loop is interruptible: if the agent is mid-turn, the tick is
 *    deferred rather than interrupting.
 *
 * LocoWorker extensions:
 *  - Tick content includes pending/overdue task counts
 *  - Tick priority escalates when tasks are overdue (critical → 'now')
 *  - The engine supports sleep/wake (overnight cost-saving)
 *  - Each tick is logged to KairosStore for audit
 *  - Tick interval is adaptive: shortens when overdue tasks exist
 *  - The engine emits typed events to the core EventBus
 *  - Mode transitions are emitted as events (reactive ↔ proactive ↔ sleeping)
 *
 * Architecture:
 *  - TickEngine owns a single setTimeout chain (NOT setInterval).
 *    This is intentional: if a tick handler takes longer than the interval,
 *    the next tick is deferred rather than stacked.
 *  - TickEngine is started/stopped explicitly. It does NOT auto-start.
 *  - TickEngine is safe to call start() / stop() multiple times.
 *
 * Integration:
 *  - KairosAgent owns the TickEngine and connects it to core EventBus.
 *  - The TickEngine itself has no direct core dependency (decoupled via
 *    the TickHandler callback pattern).
 */

import { randomUUID } from 'node:crypto';
import { EventEmitter } from 'node:events';
import type { KairosStore } from './KairosStore.js';
import { TaskQueue } from './TaskQueue.js';
import type {
  KairosTick,
  KairosTickMode,
  KairosTickPriority,
} from './types/kairos.types.js';
import {
  DEFAULT_TICK_INTERVAL_MS,
  MIN_TICK_INTERVAL_MS,
} from './types/kairos.types.js';

// ─── Types ─────────────────────────────────────────────────────────────────────

export interface TickEngineOptions {
  /**
   * Base interval between ticks in ms. Default: 10 minutes.
   * The actual interval may be shorter when overdue tasks exist.
   */
  tickIntervalMs?: number;

  /**
   * Minimum interval (safety floor). Default: 30 seconds.
   */
  minIntervalMs?: number;

  /**
   * When true, the engine starts in 'sleeping' mode during overnight hours.
   * TemporalContext.isOvernightWindow() is consulted each tick.
   * Default: true.
   */
  respectOvernightWindow?: boolean;

  /**
   * During the overnight window, multiply the tick interval by this factor
   * to reduce LLM costs. Default: 6 (10 min → 60 min ticks overnight).
   */
  overnightIntervalMultiplier?: number;

  /**
   * Max consecutive ticks with no triggered work before the engine
   * automatically slows down (doubles interval up to maxIntervalMs).
   * Default: 5.
   */
  idleSlowdownThreshold?: number;

  /**
   * Absolute maximum interval (prevents runaway slowdown). Default: 60 min.
   */
  maxIntervalMs?: number;

  /**
   * If true, log verbose tick events to console. Default: false.
   */
  verbose?: boolean;

  /**
   * Session ID to associate with ticks fired during an active session.
   */
  sessionId?: string;
}

/**
 * Callback invoked on each tick. The handler should:
 *  - Inspect the tick (task counts, mode, priority)
 *  - Decide whether to trigger background work
 *  - Return true if work was triggered (resets idle counter)
 */
export type TickHandler = (tick: KairosTick) => Promise<boolean>;

export interface TickEngineStatus {
  running: boolean;
  mode: KairosTickMode;
  tickCount: number;
  idleTickCount: number;
  currentIntervalMs: number;
  lastTickAt: string | null;
  nextTickAt: string | null;
  sessionId: string | undefined;
}

// ─── TickEngine ───────────────────────────────────────────────────────────────

export class TickEngine extends EventEmitter {
  private _mode: KairosTickMode = 'reactive';
  private _running = false;
  private _timer: ReturnType<typeof setTimeout> | null = null;
  private _tickCount = 0;
  private _idleTickCount = 0;
  private _currentIntervalMs: number;
  private _lastTickAt: string | null = null;
  private _nextTickAt: string | null = null;
  private _handler: TickHandler | null = null;

  private readonly _baseIntervalMs: number;
  private readonly _minIntervalMs: number;
  private readonly _maxIntervalMs: number;
  private readonly _overnightMultiplier: number;
  private readonly _idleSlowdownThreshold: number;
  private readonly _respectOvernightWindow: boolean;
  private readonly _verbose: boolean;
  private _sessionId: string | undefined;

  constructor(
    private readonly store: KairosStore,
    private readonly options: TickEngineOptions = {},
  ) {
    super();
    this._baseIntervalMs = options.tickIntervalMs ?? DEFAULT_TICK_INTERVAL_MS;
    this._minIntervalMs = options.minIntervalMs ?? MIN_TICK_INTERVAL_MS;
    this._maxIntervalMs = options.maxIntervalMs ?? 60 * 60 * 1000;
    this._overnightMultiplier = options.overnightIntervalMultiplier ?? 6;
    this._idleSlowdownThreshold = options.idleSlowdownThreshold ?? 5;
    this._respectOvernightWindow = options.respectOvernightWindow ?? true;
    this._verbose = options.verbose ?? false;
    this._sessionId = options.sessionId;
    this._currentIntervalMs = this._baseIntervalMs;
  }

  // ─── Lifecycle ──────────────────────────────────────────────────────────────

  /**
   * Register the handler that will be called on every tick.
   * Must be set before start() is called.
   */
  setHandler(handler: TickHandler): this {
    this._handler = handler;
    return this;
  }

  /**
   * Start the tick engine in proactive mode.
   * Does nothing if already running.
   */
  start(sessionId?: string): this {
    if (this._running) return this;

    if (sessionId) this._sessionId = sessionId;
    this._running = true;
    this._setMode('proactive');

    this._log('TickEngine started — proactive mode');
    this._scheduleNext(this._currentIntervalMs);
    return this;
  }

  /**
   * Stop the tick engine gracefully.
   * Clears any pending timer, emits 'stopped' event.
   */
  stop(): void {
    if (!this._running) return;
    this._running = false;
    this._clearTimer();
    this._nextTickAt = null;
    this._log('TickEngine stopped');
    this.emit('stopped');
  }

  /**
   * Pause ticking (suppresses tick handlers but keeps the loop running).
   * Used when the agent is in the middle of a long turn.
   */
  pause(): void {
    if (this._mode === 'proactive') {
      this._setMode('paused');
      this._log('TickEngine paused');
    }
  }

  /**
   * Resume from paused state.
   */
  resume(): void {
    if (this._mode === 'paused') {
      this._setMode('proactive');
      this._log('TickEngine resumed');
    }
  }

  /**
   * Manually trigger a tick immediately (used for testing and force-checks).
   * Returns the tick that was fired, or undefined if engine is stopped.
   */
  async triggerNow(): Promise<KairosTick | undefined> {
    if (!this._running) return undefined;
    return this._fireTick('now');
  }

  /**
   * Set the session ID for subsequent ticks.
   */
  setSessionId(sessionId: string | undefined): void {
    this._sessionId = sessionId;
  }

  // ─── Mode transitions ───────────────────────────────────────────────────────

  /**
   * Explicitly set the tick mode.
   * - 'proactive': ticks fire, handler is called
   * - 'reactive':  ticks fire, handler is NOT called (observation only)
   * - 'paused':    ticks fire but are throttled, no handler
   * - 'sleeping':  ticks fire at reduced frequency (overnight)
   */
  setMode(mode: KairosTickMode): void {
    this._setMode(mode);
  }

  // ─── Status ─────────────────────────────────────────────────────────────────

  get mode(): KairosTickMode {
    return this._mode;
  }

  get isRunning(): boolean {
    return this._running;
  }

  get tickCount(): number {
    return this._tickCount;
  }

  status(): TickEngineStatus {
    return {
      running: this._running,
      mode: this._mode,
      tickCount: this._tickCount,
      idleTickCount: this._idleTickCount,
      currentIntervalMs: this._currentIntervalMs,
      lastTickAt: this._lastTickAt,
      nextTickAt: this._nextTickAt,
      sessionId: this._sessionId,
    };
  }

  // ─── Internal tick loop ─────────────────────────────────────────────────────

  private _scheduleNext(intervalMs: number): void {
    this._clearTimer();
    if (!this._running) return;

    const clampedInterval = Math.max(
      this._minIntervalMs,
      Math.min(this._maxIntervalMs, intervalMs),
    );
    this._currentIntervalMs = clampedInterval;
    this._nextTickAt = new Date(Date.now() + clampedInterval).toISOString();

    this._timer = setTimeout(async () => {
      await this._onTimer();
    }, clampedInterval);
  }

  private async _onTimer(): Promise<void> {
    if (!this._running) return;

    // Check overnight window — may transition to sleeping
    if (this._respectOvernightWindow && this._isOvernightNow()) {
      if (this._mode !== 'sleeping') {
        this._setMode('sleeping');
      }
    } else if (this._mode === 'sleeping') {
      this._setMode('proactive');
    }

    // Determine priority for this tick
    const priority = this._computePriority();

    // Fire the tick
    const tick = await this._fireTick(priority);

    if (tick) {
      // Adaptive interval: shorten if overdue tasks, lengthen if idle
      const nextInterval = this._computeNextInterval(tick);
      this._scheduleNext(nextInterval);
    } else {
      this._scheduleNext(this._currentIntervalMs);
    }
  }

  private async _fireTick(priority: KairosTickPriority): Promise<KairosTick | undefined> {
    const now = new Date();
    this._tickCount++;
    this._lastTickAt = now.toISOString();

    // Wake snoozed tasks before counting
    this.store.wakeupSnoozedTasks();
    this.store.unblockTasks();
    this.store.autoFailExhaustedTasks();

    // Count pending and overdue tasks for the tick content
    const pendingRows = this.store.queryTasks({
      status: ['pending', 'running'],
      limit: 1000,
    });

    const nowMs = now.getTime();
    let overdueCount = 0;
    for (const row of pendingRows) {
      if (row.deadline && new Date(row.deadline).getTime() < nowMs) {
        overdueCount++;
      }
    }

    // Format tick content — matches KAIROS pattern: <tick>HH:MM:SS</tick>
    const timeStr = now.toISOString().slice(11, 19); // HH:MM:SS
    const content = this._formatTickContent(timeStr, pendingRows.length, overdueCount);

    const tick: KairosTick = {
      id: randomUUID(),
      firedAt: now.toISOString(),
      mode: this._mode,
      priority,
      content,
      pendingTaskCount: pendingRows.length,
      overdueTaskCount: overdueCount,
      triggeredWork: false,
      sessionId: this._sessionId,
    };

    // Emit tick event (for EventBus integration in KairosAgent)
    this.emit('tick', tick);

    // Invoke handler if in proactive mode and handler is registered
    let triggeredWork = false;
    if (this._mode === 'proactive' && this._handler) {
      try {
        triggeredWork = await this._handler(tick);
      } catch (err) {
        this._log(`Tick handler error: ${err instanceof Error ? err.message : String(err)}`);
        this.emit('error', err);
      }
    }

    tick.triggeredWork = triggeredWork;

    // Update idle counter
    if (triggeredWork) {
      this._idleTickCount = 0;
    } else {
      this._idleTickCount++;
    }

    // Log tick to DB (lightweight — no content stored)
    try {
      this.store.logTick(tick);
    } catch {
      // DB log failure should never stop the tick loop
    }

    if (this._verbose) {
      this._log(
        `Tick #${this._tickCount} [${priority}] mode=${this._mode} ` +
          `pending=${pendingRows.length} overdue=${overdueCount} ` +
          `work=${triggeredWork}`,
      );
    }

    return tick;
  }

  private _formatTickContent(
    timeStr: string,
    pendingCount: number,
    overdueCount: number,
  ): string {
    // Base KAIROS format
    let content = `<tick>${timeStr}</tick>`;

    // Augment with task state (LocoWorker extension)
    if (overdueCount > 0) {
      content += ` <overdue>${overdueCount} overdue task${overdueCount !== 1 ? 's' : ''}</overdue>`;
    }
    if (pendingCount > 0) {
      content += ` <pending>${pendingCount}</pending>`;
    }

    return content;
  }

  private _computePriority(): KairosTickPriority {
    if (this._mode === 'sleeping') return 'later';
    if (this._mode === 'paused') return 'later';

    // Check for overdue critical tasks → 'now'
    const criticalOverdue = this.store.queryTasks({
      status: ['pending'],
      priority: 'critical',
      overdueOnly: true,
      limit: 1,
    });

    if (criticalOverdue.length > 0) return 'now';

    // Check any overdue tasks → 'soon'
    const anyOverdue = this.store.queryTasks({
      status: ['pending'],
      overdueOnly: true,
      limit: 1,
    });

    if (anyOverdue.length > 0) return 'soon';

    return 'later';
  }

  private _computeNextInterval(tick: KairosTick): number {
    // Sleeping: use multiplied base interval
    if (this._mode === 'sleeping') {
      return this._baseIntervalMs * this._overnightMultiplier;
    }

    // Overdue tasks: shorten to min interval
    if (tick.overdueTaskCount > 0) {
      return this._minIntervalMs;
    }

    // Idle slowdown: after N idle ticks, double the interval (up to max)
    if (this._idleTickCount >= this._idleSlowdownThreshold) {
      return Math.min(this._currentIntervalMs * 2, this._maxIntervalMs);
    }

    // Work happened: reset to base
    if (tick.triggeredWork) {
      return this._baseIntervalMs;
    }

    return this._currentIntervalMs;
  }

  private _setMode(mode: KairosTickMode): void {
    if (this._mode === mode) return;
    const prev = this._mode;
    this._mode = mode;
    this.emit('modeChange', { from: prev, to: mode });
    this._log(`Mode: ${prev} → ${mode}`);
  }

  private _isOvernightNow(): boolean {
    const hour = new Date().getHours(); // local system time
    // Default overnight: 22:00–06:00
    return hour >= 22 || hour < 6;
  }

  private _clearTimer(): void {
    if (this._timer !== null) {
      clearTimeout(this._timer);
      this._timer = null;
    }
  }

  private _log(msg: string): void {
    if (this._verbose) {
      console.log(`[TickEngine] ${msg}`);
    }
  }
}
src/ObservationLog.ts
TypeScript

/**
 * ObservationLog — daily append-only observation log for @locoworker/kairos
 *
 * This class implements the "daily observation log" pattern from KAIROS:
 * a running log of agent insights, tool results, and system events that
 * accumulates throughout the day and is consolidated by AutoDream overnight.
 *
 * Responsibilities:
 *  - Typed append operations (agent_insight, tick, tool_result, etc.)
 *  - Date-bucketed retrieval (today, specific day, date range)
 *  - Daily summary generation (for agent context and reporting)
 *  - Consolidation status management (marks observations as consumed by AutoDream)
 *  - Weekly digest (aggregates across multiple day buckets)
 *  - [OBSERVE: ...] marker extraction delegated to TemporalContext
 *
 * Design:
 *  - ObservationLog is a thin read/write wrapper over KairosStore's
 *    observation methods. It adds convenience APIs and summarisation logic.
 *  - No DB schema ownership — KairosStore owns the schema.
 *  - All appends are delegated to KairosStore.appendObservation().
 *  - Consolidation is append-only: consolidated flag is set, never cleared.
 *
 * Integration with AutoDream (packages/memory):
 *  - AutoDream calls ObservationLog.getUnconsolidated() to fetch raw observations.
 *  - After processing, AutoDream calls ObservationLog.markConsolidated(ids).
 *  - Consolidated observations remain in the DB for historical queries.
 */

import type { KairosStore } from './KairosStore.js';
import type {
  KairosObservation,
  KairosObservationRow,
  ObservationSource,
} from './types/kairos.types.js';

// ─── Result types ─────────────────────────────────────────────────────────────

export interface DailySummary {
  dateBucket: string;                    // YYYY-MM-DD
  totalObservations: number;
  bySource: Record<ObservationSource, number>;
  byImportance: Record<1 | 2 | 3 | 4 | 5, number>;
  unconsolidated: number;
  highlights: KairosObservationRow[];    // importance >= 4
  sample: KairosObservationRow[];        // up to 5 recent observations
}

export interface WeeklyDigest {
  weekStart: string;                     // ISO 8601 Monday
  weekEnd: string;                       // ISO 8601 Sunday
  totalObservations: number;
  dailySummaries: DailySummary[];
  topObservations: KairosObservationRow[]; // importance 5 across the week
  consolidationRate: number;             // 0.0 – 1.0
}

export interface ObservationAppendOptions {
  sessionId?: string;
  tags?: string[];
  relatedTaskId?: string;
  importance?: 1 | 2 | 3 | 4 | 5;
}

// ─── ObservationLog ───────────────────────────────────────────────────────────

export class ObservationLog {
  constructor(private readonly store: KairosStore) {}

  // ─── Append operations ─────────────────────────────────────────────────────

  /**
   * Append an agent insight observation.
   * Called when [OBSERVE: ...] markers are extracted from agent messages.
   */
  appendAgentInsight(
    content: string,
    options: ObservationAppendOptions = {},
  ): KairosObservation {
    return this._append(content, 'agent_insight', options);
  }

  /**
   * Append a tick observation (brief state snapshot logged each tick).
   */
  appendTickObservation(
    content: string,
    options: ObservationAppendOptions = {},
  ): KairosObservation {
    return this._append(content, 'tick', { importance: 1, ...options });
  }

  /**
   * Append a tool result observation.
   * Called when a significant tool result warrants logging.
   */
  appendToolResult(
    content: string,
    options: ObservationAppendOptions = {},
  ): KairosObservation {
    return this._append(content, 'tool_result', { importance: 3, ...options });
  }

  /**
   * Append a schedule event observation.
   * Called when a cron job starts or completes.
   */
  appendScheduleEvent(
    content: string,
    options: ObservationAppendOptions = {},
  ): KairosObservation {
    return this._append(content, 'schedule', { importance: 2, ...options });
  }

  /**
   * Append a system-level observation (errors, capacity warnings, etc.).
   */
  appendSystemEvent(
    content: string,
    options: ObservationAppendOptions & { importance?: 1 | 2 | 3 | 4 | 5 } = {},
  ): KairosObservation {
    return this._append(content, 'system', { importance: 3, ...options });
  }

  /**
   * Append a user-created observation (manual log entry).
   */
  appendUserNote(
    content: string,
    options: ObservationAppendOptions = {},
  ): KairosObservation {
    return this._append(content, 'user', { importance: 3, ...options });
  }

  // ─── Retrieval ─────────────────────────────────────────────────────────────

  /**
   * Get all observations for today (UTC date bucket).
   */
  getToday(options: { source?: ObservationSource; importanceMin?: number } = {}): KairosObservationRow[] {
    const bucket = this._todayBucket();
    return this.store.queryObservations({
      dateBucket: bucket,
      source: options.source,
      importanceMin: options.importanceMin,
      limit: 1000,
    });
  }

  /**
   * Get all observations for a specific date (YYYY-MM-DD).
   */
  getForDate(dateBucket: string): KairosObservationRow[] {
    return this.store.queryObservations({ dateBucket, limit: 1000 });
  }

  /**
   * Get observations within a date range (ISO 8601 strings).
   */
  getForRange(dateFrom: string, dateTo: string): KairosObservationRow[] {
    return this.store.queryObservations({ dateFrom, dateTo, limit: 5000 });
  }

  /**
   * Get all unconsolidated observations (for AutoDream to process).
   * Ordered by importance DESC, then created_at ASC.
   */
  getUnconsolidated(importanceMin = 1): KairosObservationRow[] {
    return this.store.queryObservations({
      consolidated: false,
      importanceMin,
      limit: 2000,
    });
  }

  /**
   * Get today's count (quick query for TemporalContext).
   */
  getTodayCount(): number {
    return this.store.getTodayObservationCount();
  }

  // ─── Consolidation ─────────────────────────────────────────────────────────

  /**
   * Mark a set of observations as consolidated by AutoDream.
   * Returns the number of rows updated.
   */
  markConsolidated(ids: string[]): number {
    return this.store.markObservationsConsolidated(ids);
  }

  /**
   * Mark ALL unconsolidated observations up to a date as consolidated.
   * Used by AutoDream at end of nightly run.
   */
  markAllConsolidatedBefore(date: string): number {
    const rows = this.store.queryObservations({
      consolidated: false,
      dateTo: date,
      limit: 5000,
    });
    if (rows.length === 0) return 0;
    return this.store.markObservationsConsolidated(rows.map((r) => r.id));
  }

  // ─── Summarisation ─────────────────────────────────────────────────────────

  /**
   * Build a DailySummary for a given date bucket.
   */
  getDailySummary(dateBucket: string): DailySummary {
    const rows = this.getForDate(dateBucket);

    const bySource: Record<string, number> = {};
    const byImportance: Record<number, number> = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 };
    let unconsolidated = 0;
    const highlights: KairosObservationRow[] = [];

    for (const row of rows) {
      bySource[row.source] = (bySource[row.source] ?? 0) + 1;
      byImportance[row.importance] = (byImportance[row.importance] ?? 0) + 1;
      if (!row.consolidated) unconsolidated++;
      if (row.importance >= 4) highlights.push(row);
    }

    // Most recent 5 observations as sample
    const sample = [...rows]
      .sort((a, b) => b.created_at.localeCompare(a.created_at))
      .slice(0, 5);

    return {
      dateBucket,
      totalObservations: rows.length,
      bySource: bySource as Record<ObservationSource, number>,
      byImportance: byImportance as Record<1 | 2 | 3 | 4 | 5, number>,
      unconsolidated,
      highlights: highlights.sort((a, b) => b.importance - a.importance),
      sample,
    };
  }

  /**
   * Build a WeeklyDigest for the ISO week containing a given date.
   */
  getWeeklyDigest(referenceDate: Date = new Date()): WeeklyDigest {
    // Find Monday of the current week
    const day = referenceDate.getUTCDay(); // 0=Sun
    const monday = new Date(referenceDate);
    monday.setUTCDate(referenceDate.getUTCDate() - ((day + 6) % 7));
    monday.setUTCHours(0, 0, 0, 0);

    const sunday = new Date(monday);
    sunday.setUTCDate(monday.getUTCDate() + 6);
    sunday.setUTCHours(23, 59, 59, 999);

    const weekStart = monday.toISOString().slice(0, 10);
    const weekEnd = sunday.toISOString().slice(0, 10);

    // Build daily summaries for each day in the week
    const dailySummaries: DailySummary[] = [];
    const allRows: KairosObservationRow[] = [];

    for (let i = 0; i < 7; i++) {
      const day = new Date(monday);
      day.setUTCDate(monday.getUTCDate() + i);
      const bucket = day.toISOString().slice(0, 10);
      const summary = this.getDailySummary(bucket);
      dailySummaries.push(summary);
      allRows.push(...this.getForDate(bucket));
    }

    const totalObservations = allRows.length;
    const topObservations = allRows
      .filter((r) => r.importance === 5)
      .sort((a, b) => b.created_at.localeCompare(a.created_at));

    const consolidatedCount = allRows.filter((r) => r.consolidated).length;
    const consolidationRate =
      totalObservations > 0 ? consolidatedCount / totalObservations : 0;

    return {
      weekStart,
      weekEnd,
      totalObservations,
      dailySummaries,
      topObservations,
      consolidationRate: Math.round(consolidationRate * 100) / 100,
    };
  }

  /**
   * Format today's summary as a compact string for agent context injection.
   * Returns empty string if no observations today.
   */
  formatTodaySummary(): string {
    const bucket = this._todayBucket();
    const summary = this.getDailySummary(bucket);

    if (summary.totalObservations === 0) return '';

    const lines: string[] = [
      `## Observations — ${bucket}`,
      '',
      `**${summary.totalObservations}** total · **${summary.unconsolidated}** unconsolidated`,
      '',
    ];

    if (summary.highlights.length > 0) {
      lines.push('### Highlights (importance ≥ 4)');
      for (const obs of summary.highlights.slice(0, 5)) {
        lines.push(`- [${obs.source}] ${obs.content.slice(0, 120)}`);
      }
      lines.push('');
    }

    if (summary.sample.length > 0) {
      lines.push('### Recent');
      for (const obs of summary.sample) {
        const time = obs.created_at.slice(11, 16);
        lines.push(`- ${time} [${obs.source}] ${obs.content.slice(0, 100)}`);
      }
    }

    return lines.join('\n');
  }

  // ─── Internal helpers ──────────────────────────────────────────────────────

  private _append(
    content: string,
    source: ObservationSource,
    options: ObservationAppendOptions & { importance?: 1 | 2 | 3 | 4 | 5 } = {},
  ): KairosObservation {
    const now = new Date();
    return this.store.appendObservation({
      content: content.trim(),
      source,
      dateBucket: now.toISOString().slice(0, 10),
      sessionId: options.sessionId,
      tags: options.tags ?? [],
      relatedTaskId: options.relatedTaskId,
      importance: options.importance ?? 3,
    });
  }

  private _todayBucket(): string {
    return new Date().toISOString().slice(0, 10);
  }
}
src/KairosAgent.ts
TypeScript

/**
 * KairosAgent — top-level orchestrator for @locoworker/kairos
 *
 * KairosAgent bridges the TickEngine, TaskQueue, CronScheduler,
 * TemporalContext, and ObservationLog with the packages/core
 * EventBus and HooksRegistry.
 *
 * Responsibilities:
 *  - Start / stop the TickEngine with the correct session context
 *  - Register EventBus listeners for session lifecycle events
 *  - Register HooksRegistry hooks that inject temporal context into agent prompts
 *  - Route tick events onto the core EventBus as structured AgentEvents
 *  - Process [OBSERVE: ...] markers extracted from agent messages
 *  - Expose high-level task management APIs (create, complete, snooze, cancel)
 *  - Drive the CronScheduler: check for due schedules on each tick,
 *    run the appropriate job handler, record run results
 *  - Surface overdue tasks as agent warnings via beforeTurn hook
 *
 * Core integration (decoupled via interface shims, as in WikiSyncAgent):
 *  - KairosAgent accepts EventBusLike and HooksRegistryLike interfaces
 *    so packages/kairos has no hard compile-time dependency on packages/core.
 *  - The agent is designed to be instantiated by the CLI / Desktop app
 *    and passed a live EventBus instance at runtime.
 *
 * Job handlers:
 *  KairosAgent includes default handlers for all KairosScheduleJobType values.
 *  Custom handlers can be registered via registerJobHandler().
 *  External packages (memory, wiki, graphify) inject their handlers at startup.
 */

import { randomUUID } from 'node:crypto';
import type { KairosStore } from './KairosStore.js';
import { TickEngine } from './TickEngine.js';
import { TaskQueue, computeUrgency } from './TaskQueue.js';
import { CronScheduler } from './CronScheduler.js';
import { TemporalContext } from './TemporalContext.js';
import { ObservationLog } from './ObservationLog.js';
import type {
  KairosTask,
  KairosTaskStatus,
  KairosTaskPriority,
  KairosTaskOrigin,
  KairosScheduleJobType,
  KairosSchedule,
  KairosTick,
} from './types/kairos.types.js';
import type { TickEngineOptions } from './TickEngine.js';
import type { TemporalContextOptions } from './types/kairos.types.js';

// ─── Lightweight core interface shims ────────────────────────────────────────

interface AgentEvent {
  type: string;
  sessionId?: string;
  [key: string]: unknown;
}

interface EventBusLike {
  onAny(handler: (event: AgentEvent) => void | Promise<void>): () => void;
  emit(event: AgentEvent): Promise<void>;
}

interface HooksRegistryLike {
  register(
    name: string,
    handler: (payload: unknown) => unknown | Promise<unknown>,
    priority?: number,
  ): void;
}

// ─── Job handler type ──────────────────────────────────────────────────────────

export type JobHandler = (
  schedule: KairosSchedule,
  runId: string,
) => Promise<{ success: boolean; error?: string; createdTaskIds?: string[] }>;

// ─── Option types ─────────────────────────────────────────────────────────────

export interface KairosAgentOptions {
  /** Options passed to the TickEngine. */
  tickEngine?: TickEngineOptions;

  /** Options for TemporalContext assembly. */
  temporalContext?: TemporalContextOptions;

  /**
   * Max overdue tasks to surface in the beforeTurn hook warning.
   * Default: 3.
   */
  maxOverdueWarnings?: number;

  /**
   * Whether to run cron schedules on each tick.
   * Default: true.
   */
  runSchedules?: boolean;

  /**
   * Whether to append tick observations to the ObservationLog.
   * Default: false (tick logs are in kairos_tick_log, observations are for
   * higher-signal events).
   */
  logTickObservations?: boolean;
}

export interface KairosTaskCreateInput {
  title: string;
  description?: string;
  priority?: KairosTaskPriority;
  origin?: KairosTaskOrigin;
  deadline?: string;
  tags?: string[];
  graphifyNode?: string;
  wikiSlug?: string;
  metadata?: Record<string, unknown>;
  sessionId?: string;
  estimatedMinutes?: number;
  maxAttempts?: number;
}

// ─── KairosAgent ──────────────────────────────────────────────────────────────

export class KairosAgent {
  readonly tickEngine: TickEngine;
  readonly taskQueue: TaskQueue;
  readonly scheduler: CronScheduler;
  readonly temporalContext: TemporalContext;
  readonly observationLog: ObservationLog;

  private readonly _jobHandlers = new Map<KairosScheduleJobType, JobHandler>();
  private _unsubscribe: (() => void) | null = null;
  private _eventBus: EventBusLike | null = null;
  private readonly _maxOverdueWarnings: number;
  private readonly _runSchedules: boolean;
  private readonly _logTickObservations: boolean;

  constructor(
    private readonly store: KairosStore,
    private readonly options: KairosAgentOptions = {},
  ) {
    this._maxOverdueWarnings = options.maxOverdueWarnings ?? 3;
    this._runSchedules = options.runSchedules ?? true;
    this._logTickObservations = options.logTickObservations ?? false;

    // Initialise sub-components
    this.tickEngine = new TickEngine(store, options.tickEngine ?? {});
    this.taskQueue = new TaskQueue();
    this.scheduler = new CronScheduler();
    this.temporalContext = new TemporalContext(store);
    this.observationLog = new ObservationLog(store);

    // Wire the tick handler
    this.tickEngine.setHandler(this._onTick.bind(this));

    // Register default job handlers
    this._registerDefaultJobHandlers();
  }

  // ─── Lifecycle ──────────────────────────────────────────────────────────────

  /**
   * Attach to a core EventBus and begin processing events.
   * Initialises the TickEngine and refreshes schedule timings.
   * Returns `this` for chaining.
   */
  async start(eventBus: EventBusLike, sessionId?: string): Promise<this> {
    this._eventBus = eventBus;

    // Refresh schedule next-run times on startup
    this.temporalContext.refreshAllScheduleNextRuns();

    // Populate task queue from DB
    await this._refreshTaskQueue();

    // Subscribe to EventBus
    this._unsubscribe = eventBus.onAny(async (event) => {
      await this._handleEvent(event);
    });

    // Start the tick engine
    this.tickEngine.start(sessionId);

    return this;
  }

  /**
   * Detach from EventBus and stop the tick engine.
   */
  stop(): void {
    this.tickEngine.stop();
    if (this._unsubscribe) {
      this._unsubscribe();
      this._unsubscribe = null;
    }
    this._eventBus = null;
  }

  /**
   * Register hooks with the core HooksRegistry.
   */
  registerHooks(hooks: HooksRegistryLike): void {
    // beforeTurn: inject temporal context + overdue warnings
    hooks.register(
      'beforeTurn',
      async (payload: unknown) => {
        const p = payload as {
          systemPrompt?: string;
          messages?: Array<{ role: string; content: string }>;
        };

        const temporalBlock = this.temporalContext.getContextString(
          this.options.temporalContext,
        );

        if (p.systemPrompt && temporalBlock) {
          p.systemPrompt = `${p.systemPrompt}\n\n${temporalBlock}`;
        }

        // Extract [OBSERVE: ...] markers from assistant messages
        if (p.messages) {
          for (const msg of p.messages) {
            if (msg.role === 'assistant' && typeof msg.content === 'string') {
              const observations = this.temporalContext.extractObservations(
                msg.content,
                'agent_insight',
              );
              for (const obs of observations) {
                this.store.appendObservation(obs);
              }
            }
          }
        }

        return p;
      },
      40, // priority 40 — runs before wiki hook (50)
    );
  }

  // ─── Task management API ────────────────────────────────────────────────────

  /**
   * Create a new task and add it to the live queue.
   */
  createTask(input: KairosTaskCreateInput): KairosTask {
    const task = this.store.createTask({
      title: input.title,
      description: input.description,
      status: 'pending',
      priority: input.priority ?? 'normal',
      origin: input.origin ?? 'agent',
      deadline: input.deadline,
      tags: input.tags ?? [],
      graphifyNode: input.graphifyNode,
      wikiSlug: input.wikiSlug,
      metadata: input.metadata ?? {},
      sessionId: input.sessionId,
      estimatedMinutes: input.estimatedMinutes,
      maxAttempts: input.maxAttempts ?? 3,
    });

    // Add to live queue
    this.taskQueue.push(task);

    // Emit event
    void this._emit({ type: 'kairos_task_created', taskId: task.id, title: task.title });

    return task;
  }

  /**
   * Mark a task as complete.
   */
  completeTask(taskId: string, summary?: string): KairosTask | undefined {
    const task = this.store.transitionTask(taskId, 'done');
    if (task) {
      this.taskQueue.remove(taskId);
      if (summary) {
        this.observationLog.appendAgentInsight(
          `Task completed: "${task.title}"${summary ? ` — ${summary}` : ''}`,
          { relatedTaskId: taskId, importance: 3 },
        );
      }
      void this._emit({ type: 'kairos_task_completed', taskId, title: task.title });
    }
    return task ?? undefined;
  }

  /**
   * Cancel a task.
   */
  cancelTask(taskId: string, reason?: string): KairosTask | undefined {
    const task = this.store.transitionTask(taskId, 'cancelled', {
      errorMessage: reason,
    });
    if (task) {
      this.taskQueue.remove(taskId);
      void this._emit({ type: 'kairos_task_cancelled', taskId, title: task.title });
    }
    return task ?? undefined;
  }

  /**
   * Snooze a task until a future time.
   */
  snoozeTask(taskId: string, until: string): KairosTask | undefined {
    const task = this.store.transitionTask(taskId, 'snoozed', { snoozeUntil: until });
    if (task) {
      this.taskQueue.remove(taskId);
      void this._emit({ type: 'kairos_task_snoozed', taskId, snoozeUntil: until });
    }
    return task ?? undefined;
  }

  /**
   * Fail a task with an error message.
   */
  failTask(taskId: string, errorMessage: string): KairosTask | undefined {
    const task = this.store.transitionTask(taskId, 'failed', { errorMessage });
    if (task) {
      this.taskQueue.remove(taskId);
      void this._emit({ type: 'kairos_task_failed', taskId, errorMessage });
    }
    return task ?? undefined;
  }

  /**
   * Get the top N most urgent tasks from the live queue.
   */
  getUrgentTasks(n = 5): Array<{ task: KairosTask; urgency: ReturnType<typeof computeUrgency> }> {
    return this.taskQueue.topN(n).map(({ task, score }) => ({
      task,
      urgency: score,
    }));
  }

  // ─── Job handler registry ──────────────────────────────────────────────────

  /**
   * Register a handler for a specific schedule job type.
   * External packages (memory, wiki, graphify) call this at startup to
   * inject their implementation.
   *
   * Example (from app startup):
   *   kairosAgent.registerJobHandler('autodream', async (schedule, runId) => {
   *     await memoryManager.runAutoDream();
   *     return { success: true };
   *   });
   */
  registerJobHandler(jobType: KairosScheduleJobType, handler: JobHandler): void {
    this._jobHandlers.set(jobType, handler);
  }

  // ─── EventBus event handler ────────────────────────────────────────────────

  private async _handleEvent(event: AgentEvent): Promise<void> {
    switch (event.type) {
      case 'session_start': {
        this.tickEngine.setSessionId(event.sessionId);
        this.tickEngine.resume();
        break;
      }

      case 'session_end': {
        // Log session end as an observation
        const sessionId = event.sessionId as string | undefined;
        const summary = event.summary as string | undefined;
        if (summary) {
          this.observationLog.appendAgentInsight(
            `Session ended: ${summary}`,
            { sessionId, importance: 2 },
          );
        }
        // Wake any snoozed tasks on session end
        const woken = this.store.wakeupSnoozedTasks();
        if (woken > 0) {
          await this._refreshTaskQueue();
        }
        break;
      }

      case 'tool_result': {
        // Log high-signal tool results as observations
        const toolName = event.toolName as string | undefined;
        const isError = event.isError as boolean | undefined;
        if (isError && toolName) {
          this.observationLog.appendToolResult(
            `Tool error: ${toolName} — ${String(event.error ?? 'unknown error')}`,
            { sessionId: event.sessionId, importance: 4 },
          );
        }
        break;
      }

      case 'compaction_triggered': {
        this.observationLog.appendSystemEvent(
          `Context compaction triggered (mode: ${event.mode ?? 'unknown'})`,
          { sessionId: event.sessionId, importance: 1 },
        );
        break;
      }
    }
  }

  // ─── Tick handler ──────────────────────────────────────────────────────────

  private async _onTick(tick: KairosTick): Promise<boolean> {
    let triggeredWork = false;

    // 1) Refresh task queue state
    const woken = this.store.wakeupSnoozedTasks();
    const unblocked = this.store.unblockTasks();
    if (woken > 0 || unblocked > 0) {
      await this._refreshTaskQueue();
    }

    // 2) Emit tick to EventBus
    await this._emit({
      type: 'kairos_tick',
      tickId: tick.id,
      mode: tick.mode,
      priority: tick.priority,
      content: tick.content,
      pendingTaskCount: tick.pendingTaskCount,
      overdueTaskCount: tick.overdueTaskCount,
      sessionId: tick.sessionId,
    });

    // 3) Surface overdue tasks as warnings
    if (tick.overdueTaskCount > 0) {
      const overdue = this.taskQueue
        .getOverdueTasks()
        .slice(0, this._maxOverdueWarnings);

      for (const { task } of overdue) {
        await this._emit({
          type: 'kairos_overdue_warning',
          taskId: task.id,
          taskTitle: task.title,
          priority: task.priority,
          deadline: task.deadline,
          sessionId: tick.sessionId,
        });
      }
      triggeredWork = true;
    }

    // 4) Run due schedules
    if (this._runSchedules) {
      const ran = await this._runDueSchedules(tick);
      if (ran > 0) triggeredWork = true;
    }

    // 5) Optional tick observation
    if (this._logTickObservations && tick.pendingTaskCount > 0) {
      this.observationLog.appendTickObservation(
        `Tick: ${tick.pendingTaskCount} pending, ${tick.overdueTaskCount} overdue`,
        { sessionId: tick.sessionId },
      );
    }

    return triggeredWork;
  }

  // ─── Schedule runner ───────────────────────────────────────────────────────

  private async _runDueSchedules(tick: KairosTick): Promise<number> {
    const allScheduleRows = this.store.getAllSchedules('active');
    const allSchedules: KairosSchedule[] = allScheduleRows.map((r) =>
      this.store._rowToSchedule(r),
    );
    const dueSchedules = this.scheduler.getDueSchedules(allSchedules, Date.now());

    let ran = 0;

    for (const schedule of dueSchedules) {
      const handler = this._jobHandlers.get(schedule.jobType);

      // Start run record
      const runId = this.store.startScheduleRun(
        schedule.id,
        schedule.name,
        schedule.jobType,
        schedule.payload,
      );

      // Log schedule start
      this.observationLog.appendScheduleEvent(
        `Schedule started: "${schedule.name}" (${schedule.jobType})`,
        { importance: 2 },
      );

      await this._emit({
        type: 'kairos_schedule_started',
        scheduleId: schedule.id,
        scheduleName: schedule.name,
        jobType: schedule.jobType,
        runId,
      });

      let result: { success: boolean; error?: string; createdTaskIds?: string[] };

      if (!handler) {
        // No handler registered — mark as skipped
        result = {
          success: false,
          error: `No handler registered for job type: ${schedule.jobType}`,
        };
      } else {
        try {
          result = await Promise.race([
            handler(schedule, runId),
            this._timeoutPromise<typeof result>(
              schedule.timeoutMinutes * 60_000,
              { success: false, error: `Job timed out after ${schedule.timeoutMinutes}m` },
            ),
          ]);
        } catch (err) {
          result = {
            success: false,
            error: err instanceof Error ? err.message : String(err),
          };
        }
      }

      // Complete run record
      this.store.completeScheduleRun(
        runId,
        result.success ? 'success' : (result.error?.includes('timed out') ? 'timeout' : 'failed'),
        { errorMessage: result.error, createdTaskIds: result.createdTaskIds },
      );

      // Update schedule next run time
      const nextRunAt = this.scheduler.computeNextRunAt(
        schedule.cronExpression,
        new Date(),
      );
      this.store.updateSchedule(schedule.id, {
        lastRunAt: new Date().toISOString(),
        nextRunAt: nextRunAt ?? undefined,
      });

      // Log schedule completion
      this.observationLog.appendScheduleEvent(
        `Schedule ${result.success ? 'completed' : 'failed'}: "${schedule.name}" — ${result.error ?? 'OK'}`,
        { importance: result.success ? 2 : 4 },
      );

      await this._emit({
        type: 'kairos_schedule_completed',
        scheduleId: schedule.id,
        scheduleName: schedule.name,
        jobType: schedule.jobType,
        runId,
        success: result.success,
        error: result.error,
      });

      ran++;
    }

    return ran;
  }

  // ─── Default job handlers ──────────────────────────────────────────────────

  private _registerDefaultJobHandlers(): void {
    // Default handlers are no-ops that log a warning.
    // External packages override these by calling registerJobHandler().

    const noopHandler =
      (jobType: KairosScheduleJobType): JobHandler =>
      async (schedule) => {
        console.warn(
          `[KairosAgent] No handler registered for job type "${jobType}". ` +
            `Register one with kairosAgent.registerJobHandler("${jobType}", handler).`,
        );
        return { success: false, error: `No handler for ${jobType}` };
      };

    const jobTypes: KairosScheduleJobType[] = [
      'autodream',
      'graph_rebuild',
      'wiki_export',
      'memory_compact',
      'wiki_sync',
      'custom_agent_task',
      'notification',
    ];

    for (const jobType of jobTypes) {
      this._jobHandlers.set(jobType, noopHandler(jobType));
    }
  }

  // ─── Internal helpers ──────────────────────────────────────────────────────

  private async _refreshTaskQueue(): Promise<void> {
    const rows = this.store.queryTasks({
      status: ['pending', 'running'],
      limit: 500,
      sortByUrgency: false,
    });
    const tasks: KairosTask[] = rows.map((r) => this.store._rowToTask(r));
    this.taskQueue.populate(tasks);
  }

  private async _emit(event: AgentEvent): Promise<void> {
    if (this._eventBus) {
      try {
        await this._eventBus.emit(event);
      } catch {
        // Never crash the agent over an EventBus emit failure
      }
    }
  }

  private _timeoutPromise<T>(ms: number, fallback: T): Promise<T> {
    return new Promise((resolve) => setTimeout(() => resolve(fallback), ms));
  }
}
src/KairosReporter.ts
TypeScript

/**
 * KairosReporter — report generation for @locoworker/kairos
 *
 * Generates KAIROS_REPORT.md: a human-readable daily/weekly report covering:
 *  - Task queue state (overdue, pending, recently completed)
 *  - Schedule calendar (next 7 days of cron jobs)
 *  - Observation daily digest
 *  - Tick audit summary
 *  - Suggested agent actions based on urgency analysis
 *
 * Design:
 *  - KairosReporter is read-only (never writes to DB).
 *  - Output is written to a configurable output path.
 *  - The report is designed to be committed to the project repo
 *    alongside MEMORY.md, GRAPH_REPORT.md, and WIKI.md.
 *  - A compact summary version is available for agent context injection.
 */

import { writeFile, mkdir } from 'node:fs/promises';
import { dirname } from 'node:path';
import type { KairosStore } from './KairosStore.js';
import { ObservationLog } from './ObservationLog.js';
import { CronScheduler } from './CronScheduler.js';
import { TaskQueue, computeUrgency } from './TaskQueue.js';
import type { KairosTask, KairosSchedule } from './types/kairos.types.js';

// ─── Option types ─────────────────────────────────────────────────────────────

export interface KairosReporterOptions {
  /** Output path for KAIROS_REPORT.md. Default: '.locoworker/kairos/KAIROS_REPORT.md' */
  outputPath?: string;

  /** Max tasks to show per section. Default: 10. */
  maxTasksPerSection?: number;

  /** Max observations to show in digest. Default: 20. */
  maxObservations?: number;

  /** Number of upcoming schedule runs to preview. Default: 5. */
  schedulePreviewCount?: number;

  /** Include detailed tick audit. Default: false. */
  includeTickAudit?: boolean;
}

export interface KairosReportResult {
  outputPath: string;
  generatedAt: string;
  sections: string[];
  estimatedTokens: number;
}

// ─── KairosReporter ───────────────────────────────────────────────────────────

export class KairosReporter {
  private readonly obsLog: ObservationLog;
  private readonly scheduler: CronScheduler;

  constructor(private readonly store: KairosStore) {
    this.obsLog = new ObservationLog(store);
    this.scheduler = new CronScheduler();
  }

  /**
   * Generate the full KAIROS_REPORT.md and write it to disk.
   */
  async generate(options: KairosReporterOptions = {}): Promise<KairosReportResult> {
    const outputPath = options.outputPath ?? '.locoworker/kairos/KAIROS_REPORT.md';
    const maxTasks = options.maxTasksPerSection ?? 10;
    const maxObs = options.maxObservations ?? 20;
    const schedulePreview = options.schedulePreviewCount ?? 5;
    const generatedAt = new Date().toISOString();

    const sections: string[] = [];

    // Header
    sections.push(this._buildHeader(generatedAt));

    // Stats summary
    sections.push(this._buildStatsSummary());

    // Overdue tasks (most critical section)
    sections.push(this._buildOverdueTasks(maxTasks));

    // Pending tasks by priority
    sections.push(this._buildPendingTasks(maxTasks));

    // Recently completed tasks
    sections.push(this._buildRecentlyCompleted(maxTasks));

    // Schedule calendar
    sections.push(this._buildScheduleCalendar(schedulePreview));

    // Observation digest
    sections.push(this._buildObservationDigest(maxObs));

    // Tick audit (optional)
    if (options.includeTickAudit) {
      sections.push(this._buildTickAudit());
    }

    // Suggested agent actions
    sections.push(this._buildSuggestedActions());

    // Footer
    sections.push(this._buildFooter());

    const report = sections.join('\n\n');

    await mkdir(dirname(outputPath), { recursive: true });
    await writeFile(outputPath, report, 'utf8');

    return {
      outputPath,
      generatedAt,
      sections: sections.map((s) => s.split('\n')[0] ?? ''),
      estimatedTokens: Math.ceil(report.length / 4),
    };
  }

  /**
   * Generate a compact summary string suitable for agent context injection.
   * Much shorter than the full report — designed for inline use.
   */
  generateCompactSummary(): string {
    const stats = this.store.getStats();
    const now = new Date();

    const lines: string[] = [
      `## Kairos Summary — ${now.toISOString().slice(0, 10)}`,
      '',
      `| Metric | Value |`,
      `|--------|-------|`,
      `| Pending tasks | ${stats.pendingTasks} |`,
      `| Overdue tasks | ${stats.overdueTasks} |`,
      `| Active schedules | ${stats.activeSchedules} |`,
      `| Observations today | ${this.store.getTodayObservationCount()} |`,
      `| Last tick | ${stats.lastTickAt ? stats.lastTickAt.slice(0, 16) + ' UTC' : 'Never'} |`,
      '',
    ];

    // Top 3 urgent tasks
    const topRows = this.store.queryTasks({
      status: ['pending', 'running'],
      sortByUrgency: true,
      limit: 3,
    });

    if (topRows.length > 0) {
      lines.push('### Most Urgent');
      for (const row of topRows) {
        const task = this.store._rowToTask(row);
        const score = computeUrgency(task);
        const deadlineStr = task.deadline
          ? (score.isOverdue
              ? `⚠ OVERDUE`
              : `due ${task.deadline.slice(0, 10)}`)
          : '';
        lines.push(
          `- **[${task.priority}]** ${task.title}${deadlineStr ? ` _(${deadlineStr})_` : ''}`,
        );
      }
    }

    return lines.join('\n');
  }

  // ─── Section builders ──────────────────────────────────────────────────────

  private _buildHeader(generatedAt: string): string {
    const stats = this.store.getStats();
    return [
      '# 🕐 KAIROS Report',
      '',
      `> Generated: **${generatedAt.slice(0, 16).replace('T', ' ')} UTC**`,
      `> Tasks: **${stats.totalTasks}** total · **${stats.pendingTasks}** pending · **${stats.overdueTasks}** overdue`,
      `> Schedules: **${stats.activeSchedules}** active`,
      `> Observations: **${stats.totalObservations}** total · **${stats.unconsolidatedObservations}** unconsolidated`,
      '',
      '---',
    ].join('\n');
  }

  private _buildStatsSummary(): string {
    const stats = this.store.getStats();
    return [
      '## 📊 Stats',
      '',
      '| Category | Count |',
      '|----------|-------|',
      `| Total tasks | ${stats.totalTasks} |`,
      `| Pending | ${stats.pendingTasks} |`,
      `| Running | ${stats.runningTasks} |`,
      `| Overdue | **${stats.overdueTasks}** |`,
      `| Completed | ${stats.completedTasks} |`,
      `| Failed | ${stats.failedTasks} |`,
      `| Active schedules | ${stats.activeSchedules} |`,
      `| Observations total | ${stats.totalObservations} |`,
      `| Unconsolidated | ${stats.unconsolidatedObservations} |`,
      `| Total ticks | ${stats.totalTicks} |`,
      `| Last tick | ${stats.lastTickAt ? stats.lastTickAt.slice(0, 16) + ' UTC' : 'Never'} |`,
      `| DB size | ${(stats.dbSizeBytes / 1024).toFixed(1)} KB |`,
    ].join('\n');
  }

  private _buildOverdueTasks(maxTasks: number): string {
    const rows = this.store.queryTasks({
      status: ['pending', 'running'],
      overdueOnly: true,
      sortByUrgency: true,
      limit: maxTasks,
    });

    if (rows.length === 0) {
      return '## ✅ Overdue Tasks\n\n_No overdue tasks._';
    }

    const now = Date.now();
    const taskLines = rows.map((row) => {
      const task = this.store._rowToTask(row);
      const score = computeUrgency(task, now);
      const overdueHours = Math.abs(Math.round(score.hoursUntilDeadline ?? 0));
      const tags = task.tags.length > 0 ? ` \`${task.tags.join('` `')}\`` : '';
      return (
        `| ⚠ **${task.title}** | ${task.priority} | ${overdueHours}h overdue |` +
        ` ${task.origin}${tags} |`
      );
    });

    return [
      `## ⚠ Overdue Tasks (${rows.length})`,
      '',
      '| Task | Priority | Overdue By | Origin |',
      '|------|----------|-----------|--------|',
      ...taskLines,
    ].join('\n');
  }

  private _buildPendingTasks(maxTasks: number): string {
    const rows = this.store.queryTasks({
      status: ['pending'],
      sortByUrgency: true,
      limit: maxTasks,
    });

    if (rows.length === 0) {
      return '## 📋 Pending Tasks\n\n_No pending tasks._';
    }

    const now = Date.now();
    const taskLines = rows.map((row) => {
      const task = this.store._rowToTask(row);
      const score = computeUrgency(task, now);
      const deadlineStr = task.deadline
        ? task.deadline.slice(0, 10)
        : '—';
      const urgency = score.totalScore.toFixed(0);
      return `| ${task.title} | ${task.priority} | ${deadlineStr} | ${urgency} | ${task.origin} |`;
    });

    return [
      `## 📋 Pending Tasks (showing ${rows.length})`,
      '',
      '| Task | Priority | Deadline | Urgency | Origin |',
      '|------|----------|----------|---------|--------|',
      ...taskLines,
    ].join('\n');
  }

  private _buildRecentlyCompleted(maxTasks: number): string {
    const rows = this.store.queryTasks({
      status: ['done'],
      sortByUrgency: false,
      limit: maxTasks,
    });

    if (rows.length === 0) {
      return '## ✅ Recently Completed\n\n_No completed tasks._';
    }

    const taskLines = rows.map((row) => {
      const task = this.store._rowToTask(row);
      const completedAt = task.completedAt?.slice(0, 10) ?? '—';
      return `| ${task.title} | ${task.origin} | ${completedAt} |`;
    });

    return [
      `## ✅ Recently Completed (${rows.length})`,
      '',
      '| Task | Origin | Completed |',
      '|------|--------|-----------|',
      ...taskLines,
    ].join('\n');
  }

  private _buildScheduleCalendar(previewCount: number): string {
    const scheduleRows = this.store.getAllSchedules('active');

    if (scheduleRows.length === 0) {
      return '## 🗓 Schedule Calendar\n\n_No active schedules._';
    }

    const lines: string[] = [
      `## 🗓 Schedule Calendar (${scheduleRows.length} active)`,
      '',
      '| Schedule | Expression | Description | Next Run |',
      '|----------|-----------|-------------|----------|',
    ];

    for (const row of scheduleRows) {
      const schedule = this.store._rowToSchedule(row);
      const desc = this.scheduler.describe(schedule.cronExpression);
      const nextRun = schedule.nextRunAt
        ? schedule.nextRunAt.slice(0, 16).replace('T', ' ') + ' UTC'
        : this.scheduler.computeNextRunAt(schedule.cronExpression)?.slice(0, 16).replace('T', ' ') + ' UTC' ?? '—';
      lines.push(`| **${schedule.name}** | \`${schedule.cronExpression}\` | ${desc} | ${nextRun} |`);
    }

    // Upcoming runs preview
    lines.push('', `### Next ${previewCount} Runs`, '');
    const allUpcoming: Array<{ name: string; runAt: Date }> = [];

    for (const row of scheduleRows) {
      const schedule = this.store._rowToSchedule(row);
      const nextDates = this.scheduler.nextNDates(schedule.cronExpression, previewCount);
      for (const d of nextDates) {
        allUpcoming.push({ name: schedule.name, runAt: d });
      }
    }

    allUpcoming
      .sort((a, b) => a.runAt.getTime() - b.runAt.getTime())
      .slice(0, previewCount)
      .forEach(({ name, runAt }) => {
        lines.push(`- **${runAt.toISOString().slice(0, 16)} UTC** → ${name}`);
      });

    return lines.join('\n');
  }

  private _buildObservationDigest(maxObs: number): string {
    const today = new Date().toISOString().slice(0, 10);
    const summary = this.obsLog.getDailySummary(today);

    if (summary.totalObservations === 0) {
      return '## 👁 Observation Digest\n\n_No observations today._';
    }

    const lines: string[] = [
      `## 👁 Observation Digest — ${today}`,
      '',
      `**${summary.totalObservations}** observations · **${summary.unconsolidated}** unconsolidated`,
      '',
    ];

    // Source breakdown
    lines.push('| Source | Count |', '|--------|-------|');
    for (const [src, count] of Object.entries(summary.bySource)) {
      lines.push(`| ${src} | ${count} |`);
    }
    lines.push('');

    // Importance breakdown
    lines.push('| Importance | Count |', '|-----------|-------|');
    for (const [imp, count] of Object.entries(summary.byImportance)) {
      if (Number(count) > 0) lines.push(`| ${imp} | ${count} |`);
    }
    lines.push('');

    // Highlights
    if (summary.highlights.length > 0) {
      lines.push('### Highlights (Importance ≥ 4)', '');
      for (const obs of summary.highlights.slice(0, 5)) {
        lines.push(`- **[${obs.source}]** ${obs.content.slice(0, 150)}`);
      }
      lines.push('');
    }

    // Recent sample
    if (summary.sample.length > 0) {
      lines.push('### Recent Observations', '');
      for (const obs of summary.sample.slice(0, maxObs)) {
        const time = obs.created_at.slice(11, 16);
        lines.push(`- \`${time}\` [${obs.source}/${obs.importance}] ${obs.content.slice(0, 120)}`);
      }
    }

    return lines.join('\n');
  }

  private _buildTickAudit(): string {
    const ticks = this.store.getRecentTicks(20);

    if (ticks.length === 0) {
      return '## ⏱ Tick Audit\n\n_No ticks recorded._';
    }

    const lines: string[] = [
      `## ⏱ Tick Audit (last ${ticks.length})`,
      '',
      '| Time | Mode | Priority | Pending | Overdue | Work |',
      '|------|------|----------|---------|---------|------|',
    ];

    for (const tick of ticks) {
      const time = tick.fired_at.slice(11, 19);
      const work = tick.triggered_work ? '✅' : '—';
      lines.push(
        `| ${time} | ${tick.mode} | ${tick.priority} | ` +
          `${tick.pending_task_count} | ${tick.overdue_task_count} | ${work} |`,
      );
    }

    return lines.join('\n');
  }

  private _buildSuggestedActions(): string {
    const stats = this.store.getStats();
    const actions: string[] = [];

    if (stats.overdueTasks > 0) {
      actions.push(
        `- 🔴 **${stats.overdueTasks} overdue task${stats.overdueTasks !== 1 ? 's' : ''}** — review and either complete, cancel, or reschedule.`,
      );
    }

    if (stats.failedTasks > 0) {
      actions.push(
        `- 🟡 **${stats.failedTasks} failed task${stats.failedTasks !== 1 ? 's' : ''}** — investigate errors and retry or archive.`,
      );
    }

    if (stats.unconsolidatedObservations > 50) {
      actions.push(
        `- 🟠 **${stats.unconsolidatedObservations} unconsolidated observations** — consider running AutoDream to consolidate insights.`,
      );
    }

    if (stats.lastTickAt === null) {
      actions.push(`- 🔵 **No ticks recorded** — start the KairosAgent to enable proactive background work.`);
    }

    if (actions.length === 0) {
      actions.push('- ✅ Everything looks healthy — no immediate actions needed.');
    }

    return ['## 💡 Suggested Actions', '', ...actions].join('\n');
  }

  private _buildFooter(): string {
    return [
      '---',
      '',
      `> _KAIROS Report generated by LocoWorker · ${new Date().toISOString()}_`,
      `> _"The right moment to act." — καιρός_`,
    ].join('\n');
  }
}
src/index.ts
TypeScript

/**
 * @locoworker/kairos — public API barrel export
 *
 * Core classes:
 *   KairosStore      — SQLite persistence (tasks, schedules, observations, ticks)
 *   TaskQueue        — In-memory deadline-aware priority queue (max-heap)
 *   CronScheduler    — Pure-TS cron parser + evaluator (zero external deps)
 *   TemporalContext  — Time-awareness injection: snapshot, format, [OBSERVE:] extraction
 *   TickEngine       — Proactive tick loop (KAIROS mode, interruptible setTimeout chain)
 *   ObservationLog   — Append-only daily observation log with daily/weekly summaries
 *   KairosAgent      — Top-level orchestrator: EventBus bridge + task management
 *   KairosReporter   — KAIROS_REPORT.md generator
 *
 * Pure functions:
 *   computeUrgency   — Compute TaskUrgencyScore for a KairosTask at a given time
 */

// ─── Classes ──────────────────────────────────────────────────────────────────
export { KairosStore } from './KairosStore.js';
export { TaskQueue, computeUrgency } from './TaskQueue.js';
export { CronScheduler } from './CronScheduler.js';
export { TemporalContext } from './TemporalContext.js';
export { TickEngine } from './TickEngine.js';
export { ObservationLog } from './ObservationLog.js';
export { KairosAgent } from './KairosAgent.js';
export { KairosReporter } from './KairosReporter.js';

// ─── Types ────────────────────────────────────────────────────────────────────
export type {
  // Task model
  KairosTask,
  KairosTaskStatus,
  KairosTaskPriority,
  KairosTaskOrigin,
  TaskUrgencyScore,

  // Schedule model
  KairosSchedule,
  KairosScheduleRun,
  KairosScheduleJobType,
  KairosScheduleStatus,

  // Tick model
  KairosTick,
  KairosTickMode,
  KairosTickPriority,
  KairosTickLogRow,

  // Observation model
  KairosObservation,
  ObservationSource,

  // DB row types
  KairosTaskRow,
  KairosScheduleRow,
  KairosScheduleRunRow,
  KairosObservationRow,

  // Query types
  KairosTaskQueryOptions,
  KairosObservationQueryOptions,
  KairosStoreStats,

  // Temporal context
  TemporalContextSnapshot,
  TemporalContextOptions,

  // Cron
  ParsedCronExpression,
  CronField,
} from './types/kairos.types.js';

export type {
  TickEngineOptions,
  TickHandler,
  TickEngineStatus,
} from './TickEngine.js';

export type {
  DailySummary,
  WeeklyDigest,
  ObservationAppendOptions,
} from './ObservationLog.js';

export type {
  JobHandler,
  KairosAgentOptions,
  KairosTaskCreateInput,
} from './KairosAgent.js';

export type {
  KairosReporterOptions,
  KairosReportResult,
} from './KairosReporter.js';

// ─── Zod schemas ─────────────────────────────────────────────────────────────
export { KairosTaskSchema, KairosScheduleSchema } from './types/kairos.types.js';

// ─── Constants ────────────────────────────────────────────────────────────────
export {
  KAIROS_SCHEMA_VERSION,
  DEFAULT_TICK_INTERVAL_MS,
  MIN_TICK_INTERVAL_MS,
  DEFAULT_KAIROS_DB,
  MAX_URGENT_TASKS_PER_TICK,
  OVERDUE_URGENCY_BOOST,
  DEADLINE_IMMINENT_SCORE,
  DEADLINE_TODAY_SCORE,
  DEADLINE_THIS_WEEK_SCORE,
  PRIORITY_BASE_SCORES,
  PRIORITY_RANK,
  DEFAULT_WORKING_HOURS,
  DEFAULT_OVERNIGHT_WINDOW,
  CHARS_PER_TOKEN,
  OBSERVE_MARKER_REGEX,
  DEFAULT_SCHEDULES,
} from './types/kairos.types.js';
src/tests/KairosStore.test.ts
TypeScript

/**
 * KairosStore test suite
 * Run with: bun test src/tests/KairosStore.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test';
import { KairosStore } from '../KairosStore.js';
import type { KairosTask } from '../types/kairos.types.js';

// ─── Fixtures ─────────────────────────────────────────────────────────────────

function makeTaskInput(
  overrides: Partial<Omit<KairosTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'>> = {},
): Omit<KairosTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'> {
  return {
    title: 'Refactor ToolRegistry',
    status: 'pending',
    priority: 'normal',
    origin: 'human',
    tags: ['core', 'refactor'],
    metadata: {},
    maxAttempts: 3,
    ...overrides,
  };
}

function futureIso(hoursFromNow: number): string {
  return new Date(Date.now() + hoursFromNow * 3_600_000).toISOString();
}

function pastIso(hoursAgo: number): string {
  return new Date(Date.now() - hoursAgo * 3_600_000).toISOString();
}

// ─── Setup ────────────────────────────────────────────────────────────────────

let store: KairosStore;

beforeEach(() => {
  store = new KairosStore(':memory:', false); // no default schedule seeding for tests
});

afterEach(() => {
  store.close();
});

// ─── Initialisation ───────────────────────────────────────────────────────────

describe('KairosStore — init', () => {
  it('opens without error', () => {
    const stats = store.getStats();
    expect(stats.totalTasks).toBe(0);
    expect(stats.activeSchedules).toBe(0);
  });

  it('getTask returns undefined for unknown id', () => {
    expect(store.getTask('00000000-0000-0000-0000-000000000000')).toBeUndefined();
  });
});

// ─── Task CRUD ────────────────────────────────────────────────────────────────

describe('KairosStore — createTask', () => {
  it('creates a task with a generated UUID', () => {
    const task = store.createTask(makeTaskInput());
    expect(task.id).toMatch(
      /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/,
    );
  });

  it('persists all fields correctly', () => {
    const input = makeTaskInput({
      title: 'Write tests',
      priority: 'high',
      origin: 'agent',
      tags: ['testing', 'qa'],
      deadline: futureIso(24),
      estimatedMinutes: 120,
    });
    const task = store.createTask(input);
    const fetched = store.getTask(task.id);
    expect(fetched).toBeDefined();
    expect(fetched!.title).toBe('Write tests');
    expect(fetched!.priority).toBe('high');
    expect(fetched!.origin).toBe('agent');
    expect(fetched!.tags).toEqual(['testing', 'qa']);
    expect(fetched!.estimatedMinutes).toBe(120);
    expect(fetched!.attemptCount).toBe(0);
  });

  it('sets status to pending by default', () => {
    const task = store.createTask(makeTaskInput());
    expect(task.status).toBe('pending');
  });
});

describe('KairosStore — updateTask', () => {
  it('returns undefined for unknown id', () => {
    expect(
      store.updateTask('00000000-0000-0000-0000-000000000000', { title: 'New' }),
    ).toBeUndefined();
  });

  it('updates only provided fields', () => {
    const task = store.createTask(makeTaskInput());
    const updated = store.updateTask(task.id, { title: 'Updated Title', priority: 'high' });
    expect(updated!.title).toBe('Updated Title');
    expect(updated!.priority).toBe('high');
    // Other fields unchanged
    expect(updated!.origin).toBe('human');
    expect(updated!.tags).toEqual(['core', 'refactor']);
  });

  it('bumps updated_at', () => {
    const task = store.createTask(makeTaskInput());
    const before = task.updatedAt;
    // Small delay to ensure time difference
    const updated = store.updateTask(task.id, { title: 'New' });
    expect(updated!.updatedAt >= before).toBe(true);
  });
});

describe('KairosStore — transitionTask', () => {
  it('transitions pending → running and increments attemptCount', () => {
    const task = store.createTask(makeTaskInput());
    const running = store.transitionTask(task.id, 'running');
    expect(running!.status).toBe('running');
    expect(running!.attemptCount).toBe(1);
  });

  it('sets completedAt on terminal status', () => {
    const task = store.createTask(makeTaskInput());
    const done = store.transitionTask(task.id, 'done');
    expect(done!.completedAt).toBeDefined();
  });

  it('sets errorMessage on failed transition', () => {
    const task = store.createTask(makeTaskInput());
    const failed = store.transitionTask(task.id, 'failed', { errorMessage: 'Oops' });
    expect(failed!.errorMessage).toBe('Oops');
  });

  it('sets snoozeUntil on snoozed transition', () => {
    const task = store.createTask(makeTaskInput());
    const snoozeUntil = futureIso(2);
    const snoozed = store.transitionTask(task.id, 'snoozed', { snoozeUntil });
    expect(snoozed!.status).toBe('snoozed');
    expect(snoozed!.snoozeUntil).toBe(snoozeUntil);
  });

  it('clears snoozeUntil when transitioning snoozed → pending', () => {
    const task = store.createTask(makeTaskInput());
    store.transitionTask(task.id, 'snoozed', { snoozeUntil: futureIso(1) });
    // Manually set status to snoozed
    store.updateTask(task.id, { status: 'snoozed', snoozeUntil: pastIso(0.1) });
    store.wakeupSnoozedTasks();
    const woken = store.getTask(task.id);
    expect(woken!.status).toBe('pending');
    expect(woken!.snoozeUntil).toBeUndefined();
  });
});

describe('KairosStore — deleteTask', () => {
  it('returns false for unknown id', () => {
    expect(store.deleteTask('00000000-0000-0000-0000-000000000000')).toBe(false);
  });

  it('deletes the task', () => {
    const task = store.createTask(makeTaskInput());
    expect(store.deleteTask(task.id)).toBe(true);
    expect(store.getTask(task.id)).toBeUndefined();
  });
});

// ─── Query ────────────────────────────────────────────────────────────────────

describe('KairosStore — queryTasks', () => {
  beforeEach(() => {
    store.createTask(makeTaskInput({ priority: 'critical', status: 'pending' }));
    store.createTask(makeTaskInput({ priority: 'high', status: 'pending' }));
    store.createTask(makeTaskInput({ priority: 'normal', status: 'running' }));
    store.createTask(makeTaskInput({ priority: 'low', status: 'done' }));
    store.createTask(makeTaskInput({
      priority: 'critical',
      status: 'pending',
      deadline: pastIso(1), // overdue
    }));
  });

  it('returns all tasks when no filters applied', () => {
    const all = store.queryTasks({});
    expect(all.length).toBe(5);
  });

  it('filters by single status', () => {
    const pending = store.queryTasks({ status: 'pending' });
    expect(pending.every((t) => t.status === 'pending')).toBe(true);
    expect(pending.length).toBe(3); // 2 normal + 1 overdue
  });

  it('filters by multiple statuses', () => {
    const active = store.queryTasks({ status: ['pending', 'running'] });
    expect(active.length).toBe(4);
  });

  it('filters overdueOnly', () => {
    const overdue = store.queryTasks({ overdueOnly: true });
    expect(overdue.length).toBe(1);
    expect(overdue[0].priority).toBe('critical');
  });

  it('filters by priority', () => {
    const critical = store.queryTasks({ priority: 'critical' });
    expect(critical.length).toBe(2);
  });

  it('sorts by urgency (overdue first, then priority)', () => {
    const tasks = store.queryTasks({ status: 'pending', sortByUrgency: true });
    // Overdue critical should be first
    expect(tasks[0].deadline).toBeDefined();
  });

  it('respects limit', () => {
    const limited = store.queryTasks({ limit: 2 });
    expect(limited.length).toBe(2);
  });

  it('filters by tags', () => {
    store.createTask(makeTaskInput({ tags: ['special'] }));
    const tagged = store.queryTasks({ tags: ['special'] });
    expect(tagged.length).toBe(1);
  });
});

// ─── Wakeup / unblock / auto-fail ─────────────────────────────────────────────

describe('KairosStore — lifecycle helpers', () => {
  it('wakeupSnoozedTasks wakes tasks whose snooze_until has passed', () => {
    const t = store.createTask(makeTaskInput());
    store.updateTask(t.id, { status: 'snoozed', snoozeUntil: pastIso(0.01) });
    const woken = store.wakeupSnoozedTasks();
    expect(woken).toBe(1);
    expect(store.getTask(t.id)!.status).toBe('pending');
  });

  it('wakeupSnoozedTasks does not wake tasks whose snooze has not expired', () => {
    const t = store.createTask(makeTaskInput());
    store.updateTask(t.id, { status: 'snoozed', snoozeUntil: futureIso(1) });
    const woken = store.wakeupSnoozedTasks();
    expect(woken).toBe(0);
  });

  it('unblockTasks unblocks tasks whose blocker is done', () => {
    const blocker = store.createTask(makeTaskInput({ title: 'Blocker' }));
    const blocked = store.createTask(
      makeTaskInput({ title: 'Blocked', status: 'blocked', blockerTaskId: blocker.id }),
    );
    store.transitionTask(blocker.id, 'done');
    const unblocked = store.unblockTasks();
    expect(unblocked).toBe(1);
    expect(store.getTask(blocked.id)!.status).toBe('pending');
  });

  it('autoFailExhaustedTasks fails tasks that have hit max attempts', () => {
    const t = store.createTask(makeTaskInput({ maxAttempts: 2 }));
    store.updateTask(t.id, { attemptCount: 2 });
    const failed = store.autoFailExhaustedTasks();
    expect(failed).toBe(1);
    expect(store.getTask(t.id)!.status).toBe('failed');
    expect(store.getTask(t.id)!.errorMessage).toBe('Max attempts exceeded');
  });
});

// ─── Observations ─────────────────────────────────────────────────────────────

describe('KairosStore — observations', () => {
  it('appends an observation and returns it with an ID', () => {
    const obs = store.appendObservation({
      content: 'Something interesting happened',
      source: 'agent_insight',
      dateBucket: new Date().toISOString().slice(0, 10),
      tags: ['test'],
      importance: 4,
    });
    expect(obs.id).toMatch(/^[0-9a-f-]{36}$/);
    expect(obs.consolidated).toBe(false);
  });

  it('queries by date bucket', () => {
    const bucket = new Date().toISOString().slice(0, 10);
    store.appendObservation({ content: 'obs1', source: 'tick', dateBucket: bucket, tags: [], importance: 1 });
    store.appendObservation({ content: 'obs2', source: 'system', dateBucket: '2020-01-01', tags: [], importance: 1 });
    const results = store.queryObservations({ dateBucket: bucket });
    expect(results.length).toBe(1);
    expect(results[0].content).toBe('obs1');
  });

  it('markObservationsConsolidated updates the flag', () => {
    const obs = store.appendObservation({
      content: 'consolidate me',
      source: 'agent_insight',
      dateBucket: '2026-01-01',
      tags: [],
      importance: 3,
    });
    const count = store.markObservationsConsolidated([obs.id]);
    expect(count).toBe(1);
    const row = store.queryObservations({ dateBucket: '2026-01-01' });
    expect(row[0].consolidated).toBe(1);
  });

  it('getTodayObservationCount returns correct count', () => {
    const bucket = new Date().toISOString().slice(0, 10);
    store.appendObservation({ content: 'a', source: 'tick', dateBucket: bucket, tags: [], importance: 1 });
    store.appendObservation({ content: 'b', source: 'tick', dateBucket: bucket, tags: [], importance: 1 });
    expect(store.getTodayObservationCount()).toBe(2);
  });
});

// ─── Tick log ─────────────────────────────────────────────────────────────────

describe('KairosStore — tick log', () => {
  it('logs a tick and retrieves it', () => {
    store.logTick({
      id: '00000000-0000-0000-0000-000000000001',
      firedAt: new Date().toISOString(),
      mode: 'proactive',
      priority: 'later',
      content: '<tick>12:00:00</tick>',
      pendingTaskCount: 5,
      overdueTaskCount: 0,
      triggeredWork: false,
    });
    const ticks = store.getRecentTicks(5);
    expect(ticks.length).toBe(1);
    expect(ticks[0].mode).toBe('proactive');
    expect(ticks[0].pending_task_count).toBe(5);
  });

  it('getLastTickAt returns null when no ticks', () => {
    expect(store.getLastTickAt()).toBeNull();
  });

  it('getLastTickAt returns the most recent tick time', () => {
    const t1 = new Date(Date.now() - 10000).toISOString();
    const t2 = new Date().toISOString();
    store.logTick({ id: '00000000-0000-0000-0000-000000000002', firedAt: t1, mode: 'proactive', priority: 'later', content: '<tick/>', pendingTaskCount: 0, overdueTaskCount: 0, triggeredWork: false });
    store.logTick({ id: '00000000-0000-0000-0000-000000000003', firedAt: t2, mode: 'proactive', priority: 'later', content: '<tick/>', pendingTaskCount: 0, overdueTaskCount: 0, triggeredWork: false });
    expect(store.getLastTickAt()).toBe(t2);
  });
});

// ─── Stats ────────────────────────────────────────────────────────────────────

describe('KairosStore — getStats', () => {
  it('accurately counts tasks by status', () => {
    store.createTask(makeTaskInput({ status: 'pending' }));
    const t2 = store.createTask(makeTaskInput());
    store.transitionTask(t2.id, 'running');
    const t3 = store.createTask(makeTaskInput());
    store.transitionTask(t3.id, 'done');
    const t4 = store.createTask(makeTaskInput());
    store.transitionTask(t4.id, 'failed', { errorMessage: 'err' });

    const stats = store.getStats();
    expect(stats.totalTasks).toBe(4);
    expect(stats.pendingTasks).toBe(1);
    expect(stats.runningTasks).toBe(1);
    expect(stats.completedTasks).toBe(1);
    expect(stats.failedTasks).toBe(1);
  });

  it('counts overdue tasks', () => {
    store.createTask(makeTaskInput({ deadline: pastIso(2), status: 'pending' }));
    store.createTask(makeTaskInput({ deadline: futureIso(2), status: 'pending' }));
    const stats = store.getStats();
    expect(stats.overdueTasks).toBe(1);
  });
});
src/tests/TaskQueue.test.ts
TypeScript

/**
 * TaskQueue test suite
 * Run with: bun test src/tests/TaskQueue.test.ts
 */

import { describe, it, expect, beforeEach } from 'bun:test';
import { TaskQueue, computeUrgency } from '../TaskQueue.js';
import type { KairosTask, KairosTaskPriority } from '../types/kairos.types.js';
import { PRIORITY_BASE_SCORES, OVERDUE_URGENCY_BOOST, DEADLINE_IMMINENT_SCORE } from '../types/kairos.types.js';

// ─── Fixtures ─────────────────────────────────────────────────────────────────

function makeTask(
  id: string,
  priority: KairosTaskPriority = 'normal',
  overrides: Partial<KairosTask> = {},
): KairosTask {
  const now = new Date().toISOString();
  return {
    id,
    title: `Task ${id}`,
    status: 'pending',
    priority,
    origin: 'human',
    tags: [],
    metadata: {},
    attemptCount: 0,
    maxAttempts: 3,
    createdAt: now,
    updatedAt: now,
    ...overrides,
  };
}

function futureIso(hoursFromNow: number): string {
  return new Date(Date.now() + hoursFromNow * 3_600_000).toISOString();
}

function pastIso(hoursAgo: number): string {
  return new Date(Date.now() - hoursAgo * 3_600_000).toISOString();
}

// ─── computeUrgency ───────────────────────────────────────────────────────────

describe('computeUrgency — base scores', () => {
  it('critical priority has highest base score', () => {
    const task = makeTask('t1', 'critical');
    const score = computeUrgency(task);
    expect(score.baseScore).toBe(PRIORITY_BASE_SCORES.critical);
  });

  it('backlog priority has lowest base score', () => {
    const task = makeTask('t1', 'backlog');
    const score = computeUrgency(task);
    expect(score.baseScore).toBe(PRIORITY_BASE_SCORES.backlog);
  });

  it('no deadline → deadlineScore and overdueBonus are 0', () => {
    const task = makeTask('t1', 'normal');
    const score = computeUrgency(task);
    expect(score.deadlineScore).toBe(0);
    expect(score.overdueBonus).toBe(0);
    expect(score.isOverdue).toBe(false);
  });

  it('deadline within 1 hour → DEADLINE_IMMINENT_SCORE', () => {
    const task = makeTask('t1', 'normal', { deadline: futureIso(0.5) });
    const score = computeUrgency(task);
    expect(score.deadlineScore).toBe(DEADLINE_IMMINENT_SCORE);
    expect(score.isOverdue).toBe(false);
  });

  it('deadline within 24 hours → DEADLINE_TODAY_SCORE', () => {
    const task = makeTask('t1', 'normal', { deadline: futureIso(12) });
    const score = computeUrgency(task);
    expect(score.deadlineScore).toBe(25);
    expect(score.isOverdue).toBe(false);
  });

  it('overdue task → isOverdue=true, overdueBonus applied', () => {
    const task = makeTask('t1', 'normal', { deadline: pastIso(2) });
    const score = computeUrgency(task);
    expect(score.isOverdue).toBe(true);
    expect(score.overdueBonus).toBe(OVERDUE_URGENCY_BOOST);
    expect(score.hoursUntilDeadline).toBeLessThan(0);
  });

  it('total score = base + deadline + overdue', () => {
    const task = makeTask('t1', 'critical', { deadline: pastIso(1) });
    const score = computeUrgency(task);
    expect(score.totalScore).toBe(
      score.baseScore + score.deadlineScore + score.overdueBonus,
    );
  });
});

// ─── TaskQueue ────────────────────────────────────────────────────────────────

describe('TaskQueue — basic operations', () => {
  let queue: TaskQueue;

  beforeEach(() => {
    queue = new TaskQueue();
  });

  it('isEmpty is true on new queue', () => {
    expect(queue.isEmpty).toBe(true);
    expect(queue.size).toBe(0);
  });

  it('peek returns undefined on empty queue', () => {
    expect(queue.peek()).toBeUndefined();
  });

  it('pop returns undefined on empty queue', () => {
    expect(queue.pop()).toBeUndefined();
  });

  it('push increases size', () => {
    queue.push(makeTask('t1', 'normal'));
    expect(queue.size).toBe(1);
    expect(queue.isEmpty).toBe(false);
  });

  it('pop decreases size', () => {
    queue.push(makeTask('t1', 'normal'));
    queue.pop();
    expect(queue.size).toBe(0);
  });

  it('peek does not decrease size', () => {
    queue.push(makeTask('t1', 'normal'));
    queue.peek();
    expect(queue.size).toBe(1);
  });

  it('clear empties the queue', () => {
    queue.push(makeTask('t1'));
    queue.push(makeTask('t2'));
    queue.clear();
    expect(queue.size).toBe(0);
    expect(queue.isEmpty).toBe(true);
  });

  it('has() returns correct boolean', () => {
    queue.push(makeTask('t1'));
    expect(queue.has('t1')).toBe(true);
    expect(queue.has('t2')).toBe(false);
  });

  it('remove() returns false for non-existent task', () => {
    expect(queue.remove('ghost')).toBe(false);
  });

  it('remove() correctly removes a task', () => {
    queue.push(makeTask('t1'));
    queue.push(makeTask('t2'));
    const removed = queue.remove('t1');
    expect(removed).toBe(true);
    expect(queue.has('t1')).toBe(false);
    expect(queue.size).toBe(1);
  });
});

describe('TaskQueue — priority ordering', () => {
  let queue: TaskQueue;

  beforeEach(() => {
    queue = new TaskQueue();
  });

  it('critical task is returned before high before normal', () => {
    queue.push(makeTask('normal', 'normal'));
    queue.push(makeTask('high', 'high'));
    queue.push(makeTask('critical', 'critical'));

    expect(queue.pop()!.task.id).toBe('critical');
    expect(queue.pop()!.task.id).toBe('high');
    expect(queue.pop()!.task.id).toBe('normal');
  });

  it('overdue task beats higher-priority non-overdue task', () => {
    const overdueNormal = makeTask('overdue-normal', 'normal', { deadline: pastIso(1) });
    const freshCritical = makeTask('fresh-critical', 'critical');
    // Note: overdue normal = 20 (base) + 50 (deadline imminent) + 30 (overdue) = 100
    // fresh critical = 40 (base) = 40
    queue.push(overdueNormal);
    queue.push(freshCritical);
    // Overdue normal (100) > fresh critical (40)
    expect(queue.pop()!.task.id).toBe('overdue-normal');
  });

  it('task with imminent deadline is more urgent than task without', () => {
    const withDeadline = makeTask('with-deadline', 'normal', { deadline: futureIso(0.5) });
    const withoutDeadline = makeTask('without-deadline', 'normal');
    queue.push(withoutDeadline);
    queue.push(withDeadline);
    expect(queue.pop()!.task.id).toBe('with-deadline');
  });

  it('among equal urgency, earlier deadline wins', () => {
    const soonDeadline = makeTask('soon', 'normal', { deadline: futureIso(2) });
    const laterDeadline = makeTask('later', 'normal', { deadline: futureIso(48) });
    queue.push(laterDeadline);
    queue.push(soonDeadline);
    // Both in "this week" range: score = 20 + 10 = 30 each
    // Tiebreak: earlier deadline wins
    expect(queue.pop()!.task.id).toBe('soon');
  });
});

describe('TaskQueue — populate and topN', () => {
  it('populate builds a correct heap from an array', () => {
    const queue = new TaskQueue();
    const tasks = [
      makeTask('low', 'low'),
      makeTask('critical', 'critical'),
      makeTask('normal', 'normal'),
      makeTask('high', 'high'),
      makeTask('backlog', 'backlog'),
    ];
    queue.populate(tasks);
    expect(queue.size).toBe(5);
    expect(queue.pop()!.task.id).toBe('critical');
    expect(queue.pop()!.task.id).toBe('high');
    expect(queue.pop()!.task.id).toBe('normal');
  });

  it('topN returns N most urgent without modifying queue', () => {
    const queue = new TaskQueue();
    queue.push(makeTask('backlog', 'backlog'));
    queue.push(makeTask('critical', 'critical'));
    queue.push(makeTask('high', 'high'));
    const top2 = queue.topN(2);
    expect(top2.length).toBe(2);
    expect(top2[0].task.id).toBe('critical');
    expect(top2[1].task.id).toBe('high');
    // Queue should still have all 3
    expect(queue.size).toBe(3);
  });

  it('topN returns all tasks if N > queue size', () => {
    const queue = new TaskQueue();
    queue.push(makeTask('t1'));
    const top = queue.topN(10);
    expect(top.length).toBe(1);
  });
});

describe('TaskQueue — overdue queries', () => {
  it('getOverdueTasks returns only overdue tasks', () => {
    const queue = new TaskQueue();
    queue.push(makeTask('overdue', 'normal', { deadline: pastIso(1) }));
    queue.push(makeTask('future', 'normal', { deadline: futureIso(2) }));
    queue.push(makeTask('no-deadline', 'normal'));
    const overdue = queue.getOverdueTasks();
    expect(overdue.length).toBe(1);
    expect(overdue[0].task.id).toBe('overdue');
  });

  it('getTasksDueWithinHours returns tasks in the window', () => {
    const queue = new TaskQueue();
    queue.push(makeTask('due-soon', 'normal', { deadline: futureIso(3) }));
    queue.push(makeTask('due-later', 'normal', { deadline: futureIso(48) }));
    queue.push(makeTask('overdue', 'normal', { deadline: pastIso(1) }));
    const dueSoon = queue.getTasksDueWithinHours(6);
    expect(dueSoon.map((t) => t.task.id)).toContain('due-soon');
    expect(dueSoon.map((t) => t.task.id)).not.toContain('due-later');
    expect(dueSoon.map((t) => t.task.id)).not.toContain('overdue');
  });
});

describe('TaskQueue — refresh', () => {
  it('refresh recomputes scores and re-heapifies', () => {
    const queue = new TaskQueue();
    const t1 = makeTask('t1', 'normal');
    const t2 = makeTask('t2', 'normal');
    queue.push(t1);
    queue.push(t2);
    // After refresh, order should remain valid
    queue.refresh();
    expect(queue.size).toBe(2);
  });
});

describe('TaskQueue — getSummary', () => {
  it('reports correct counts', () => {
    const queue = new TaskQueue();
    queue.push(makeTask('overdue', 'critical', { deadline: pastIso(1) }));
    queue.push(makeTask('today', 'high', { deadline: futureIso(6) }));
    queue.push(makeTask('normal', 'normal'));
    const summary = queue.getSummary();
    expect(summary.totalTasks).toBe(3);
    expect(summary.overdueTasks).toBe(1);
    expect(summary.tasksDueToday).toBe(1);
    expect(summary.criticalTasks).toBe(1);
    expect(summary.highTasks).toBe(1);
  });
});
src/tests/CronScheduler.test.ts
TypeScript

/**
 * CronScheduler test suite
 * Run with: bun test src/tests/CronScheduler.test.ts
 */

import { describe, it, expect, beforeEach } from 'bun:test';
import { CronScheduler } from '../CronScheduler.js';

let scheduler: CronScheduler;

beforeEach(() => {
  scheduler = new CronScheduler();
});

// ─── parse ────────────────────────────────────────────────────────────────────

describe('CronScheduler.parse — valid expressions', () => {
  it('parses wildcard fields', () => {
    const p = scheduler.parse('* * * * *');
    expect(p.minute.isWildcard).toBe(true);
    expect(p.hour.isWildcard).toBe(true);
    expect(p.minute.values).toHaveLength(60);
    expect(p.hour.values).toHaveLength(24);
  });

  it('parses specific values', () => {
    const p = scheduler.parse('30 9 1 6 0');
    expect(p.minute.values).toEqual([30]);
    expect(p.hour.values).toEqual([9]);
    expect(p.dayOfMonth.values).toEqual([1]);
    expect(p.month.values).toEqual([6]);
    expect(p.dayOfWeek.values).toEqual([0]);
  });

  it('parses comma-separated lists', () => {
    const p = scheduler.parse('0,15,30,45 * * * *');
    expect(p.minute.values).toEqual([0, 15, 30, 45]);
  });

  it('parses range (a-b)', () => {
    const p = scheduler.parse('* 9-17 * * *');
    expect(p.hour.values).toEqual([9, 10, 11, 12, 13, 14, 15, 16, 17]);
  });

  it('parses step (*/n)', () => {
    const p = scheduler.parse('*/15 * * * *');
    expect(p.minute.values).toEqual([0, 15, 30, 45]);
  });

  it('parses step with range (a-b/n)', () => {
    const p = scheduler.parse('0-30/10 * * * *');
    expect(p.minute.values).toEqual([0, 10, 20, 30]);
  });

  it('parses @daily alias', () => {
    const p = scheduler.parse('@daily');
    expect(p.minute.values).toEqual([0]);
    expect(p.hour.values).toEqual([0]);
  });

  it('parses @hourly alias', () => {
    const p = scheduler.parse('@hourly');
    expect(p.minute.values).toEqual([0]);
    expect(p.hour.values).toHaveLength(24);
  });

  it('parses @weekly alias', () => {
    const p = scheduler.parse('@weekly');
    expect(p.dayOfWeek.values).toEqual([0]);
    expect(p.hour.values).toEqual([0]);
  });

  it('parses @yearly alias', () => {
    const p = scheduler.parse('@yearly');
    expect(p.month.values).toEqual([1]);
    expect(p.dayOfMonth.values).toEqual([1]);
  });

  it('parses named day-of-week (mon-fri)', () => {
    const p = scheduler.parse('0 9 * * mon-fri');
    expect(p.dayOfWeek.values).toEqual([1, 2, 3, 4, 5]);
  });

  it('parses named months (jan,dec)', () => {
    const p = scheduler.parse('0 0 1 jan,dec *');
    expect(p.month.values).toEqual([1, 12]);
  });
});

describe('CronScheduler.parse — invalid expressions', () => {
  it('throws on wrong field count', () => {
    expect(() => scheduler.parse('* * * *')).toThrow();
  });

  it('throws on out-of-range minute', () => {
    expect(() => scheduler.parse('60 * * * *')).toThrow();
  });

  it('throws on out-of-range hour', () => {
    expect(() => scheduler.parse('* 24 * * *')).toThrow();
  });

  it('throws on zero step', () => {
    expect(() => scheduler.parse('*/0 * * * *')).toThrow();
  });
});

describe('CronScheduler.validate', () => {
  it('returns null for valid expression', () => {
    expect(scheduler.validate('0 2 * * *')).toBeNull();
  });

  it('returns error string for invalid expression', () => {
    const err = scheduler.validate('bad');
    expect(typeof err).toBe('string');
    expect(err!.length).toBeGreaterThan(0);
  });
});

// ─── matches ─────────────────────────────────────────────────────────────────

describe('CronScheduler.matches', () => {
  it('matches a specific time correctly', () => {
    const p = scheduler.parse('30 14 15 6 *');
    // June 15 at 14:30 UTC
    const match = new Date('2026-06-15T14:30:00Z');
    const noMatch = new Date('2026-06-15T14:31:00Z');
    expect(scheduler.matches(p, match)).toBe(true);
    expect(scheduler.matches(p, noMatch)).toBe(false);
  });

  it('matches wildcard for any value', () => {
    const p = scheduler.parse('* * * * *');
    expect(scheduler.matches(p, new Date())).toBe(true);
  });

  it('matches day-of-week correctly', () => {
    // Sunday = 0
    const p = scheduler.parse('0 12 * * 0');
    const sunday = new Date('2026-05-10T12:00:00Z'); // 2026-05-10 is a Sunday
    const monday = new Date('2026-05-11T12:00:00Z');
    expect(scheduler.matches(p, sunday)).toBe(true);
    expect(scheduler.matches(p, monday)).toBe(false);
  });

  it('matches step expressions', () => {
    const p = scheduler.parse('*/30 * * * *');
    const onHour = new Date('2026-01-01T10:00:00Z');
    const halfHour = new Date('2026-01-01T10:30:00Z');
    const quarter = new Date('2026-01-01T10:15:00Z');
    expect(scheduler.matches(p, onHour)).toBe(true);
    expect(scheduler.matches(p, halfHour)).toBe(true);
    expect(scheduler.matches(p, quarter)).toBe(false);
  });
});

// ─── nextDate ────────────────────────────────────────────────────────────────

describe('CronScheduler.nextDate', () => {
  it('returns a future date', () => {
    const now = new Date();
    const next = scheduler.nextDate('* * * * *', now);
    expect(next).not.toBeNull();
    expect(next!.getTime()).toBeGreaterThan(now.getTime());
  });

  it('returns the correct next run for "0 2 * * *" (2am daily)', () => {
    // Reference: just after 2am
    const after = new Date('2026-05-07T02:01:00Z');
    const next = scheduler.nextDate('0 2 * * *', after);
    expect(next).not.toBeNull();
    // Should be next day at 02:00
    expect(next!.toISOString()).toBe('2026-05-08T02:00:00.000Z');
  });

  it('returns correct next for "*/15 * * * *"', () => {
    const after = new Date('2026-05-07T10:07:00Z');
    const next = scheduler.nextDate('*/15 * * * *', after);
    expect(next!.toISOString()).toBe('2026-05-07T10:15:00.000Z');
  });

  it('returns correct next for "0 9 * * 1" (Mondays at 9am)', () => {
    // 2026-05-07 is a Thursday — next Monday is 2026-05-11
    const after = new Date('2026-05-07T10:00:00Z');
    const next = scheduler.nextDate('0 9 * * 1', after);
    expect(next!.toISOString()).toBe('2026-05-11T09:00:00.000Z');
  });

  it('handles month boundary correctly', () => {
    // "0 0 1 * *" — 1st of each month at midnight
    const after = new Date('2026-05-15T00:00:00Z');
    const next = scheduler.nextDate('0 0 1 * *', after);
    expect(next!.toISOString()).toBe('2026-06-01T00:00:00.000Z');
  });

  it('handles year boundary (Dec 31 → Jan 1 next year)', () => {
    const after = new Date('2026-12-31T23:59:00Z');
    const next = scheduler.nextDate('@yearly', after);
    expect(next!.getUTCFullYear()).toBe(2027);
    expect(next!.getUTCMonth()).toBe(0); // January
    expect(next!.getUTCDate()).toBe(1);
  });
});

describe('CronScheduler.nextNDates', () => {
  it('returns N future dates in ascending order', () => {
    const dates = scheduler.nextNDates('0 * * * *', 5, new Date('2026-05-07T10:00:00Z'));
    expect(dates.length).toBe(5);
    for (let i = 1; i < dates.length; i++) {
      expect(dates[i].getTime()).toBeGreaterThan(dates[i - 1].getTime());
    }
  });

  it('all returned dates match the expression', () => {
    const parsed = scheduler.parse('0 * * * *');
    const dates = scheduler.nextNDates('0 * * * *', 3, new Date());
    for (const d of dates) {
      expect(scheduler.matches(parsed, d)).toBe(true);
    }
  });
});

// ─── getDueSchedules ─────────────────────────────────────────────────────────

describe('CronScheduler.getDueSchedules', () => {
  const now = new Date('2026-05-07T10:00:00Z');

  function makeSchedule(nextRunAt: string | undefined, status = 'active'): any {
    return {
      id: '1',
      name: 'Test Schedule',
      cronExpression: '0 * * * *',
      jobType: 'autodream',
      payload: {},
      status,
      timezone: 'UTC',
      timeoutMinutes: 30,
      nextRunAt,
      createdAt: '2026-01-01T00:00:00Z',
      updatedAt: '2026-01-01T00:00:00Z',
    };
  }

  it('returns schedule whose nextRunAt is in the past', () => {
    const schedule = makeSchedule('2026-05-07T09:00:00Z'); // 1hr ago
    const due = scheduler.getDueSchedules([schedule], now.getTime());
    expect(due.length).toBe(1);
  });

  it('does not return schedule whose nextRunAt is in the future', () => {
    const schedule = makeSchedule('2026-05-07T11:00:00Z'); // 1hr from now
    const due = scheduler.getDueSchedules([schedule], now.getTime());
    expect(due.length).toBe(0);
  });

  it('ignores paused schedules', () => {
    const schedule = makeSchedule('2026-05-07T09:00:00Z', 'paused');
    const due = scheduler.getDueSchedules([schedule], now.getTime());
    expect(due.length).toBe(0);
  });

  it('ignores disabled schedules', () => {
    const schedule = makeSchedule('2026-05-07T09:00:00Z', 'disabled');
    const due = scheduler.getDueSchedules([schedule], now.getTime());
    expect(due.length).toBe(0);
  });
});

// ─── describe ────────────────────────────────────────────────────────────────

describe('CronScheduler.describe', () => {
  it('describes every minute', () => {
    expect(scheduler.describe('* * * * *')).toBe('Every minute');
  });

  it('describes every N minutes', () => {
    expect(scheduler.describe('*/15 * * * *')).toBe('Every 15 minutes');
  });

  it('describes daily at specific time', () => {
    expect(scheduler.describe('0 2 * * *')).toBe('Every day at 02:00 UTC');
  });

  it('describes weekdays', () => {
    expect(scheduler.describe('0 9 * * mon-fri')).toContain('Mon');
    expect(scheduler.describe('0 9 * * mon-fri')).toContain('Fri');
  });

  it('describes @daily alias', () => {
    expect(scheduler.describe('@daily')).toBe('Every day at 00:00 UTC');
  });

  it('returns generic description for complex expressions', () => {
    const desc = scheduler.describe('0 0 1 1,7 *');
    expect(typeof desc).toBe('string');
    expect(desc.length).toBeGreaterThan(0);
  });

  it('returns error message for invalid expression', () => {
    const desc = scheduler.describe('not-valid');
    expect(desc).toContain('Invalid');
  });
});
src/tests/TemporalContext.test.ts
TypeScript

/**
 * TemporalContext test suite
 * Run with: bun test src/tests/TemporalContext.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test';
import { KairosStore } from '../KairosStore.js';
import { TemporalContext } from '../TemporalContext.js';
import type { KairosTask } from '../types/kairos.types.js';

// ─── Fixtures ─────────────────────────────────────────────────────────────────

function makeTask(
  overrides: Partial<Omit<KairosTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'>> = {},
): Omit<KairosTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'> {
  return {
    title: 'Test Task',
    status: 'pending',
    priority: 'normal',
    origin: 'human',
    tags: [],
    metadata: {},
    maxAttempts: 3,
    ...overrides,
  };
}

function pastIso(hoursAgo: number): string {
  return new Date(Date.now() - hoursAgo * 3_600_000).toISOString();
}

function futureIso(hoursFromNow: number): string {
  return new Date(Date.now() + hoursFromNow * 3_600_000).toISOString();
}

// ─── Setup ────────────────────────────────────────────────────────────────────

let store: KairosStore;
let ctx: TemporalContext;

beforeEach(() => {
  store = new KairosStore(':memory:', false);
  ctx = new TemporalContext(store);
});

afterEach(() => {
  store.close();
});

// ─── assembleSnapshot ─────────────────────────────────────────────────────────

describe('TemporalContext.assembleSnapshot', () => {
  it('returns a snapshot with nowUtc and dayName populated', () => {
    const snapshot = ctx.assembleSnapshot();
    expect(snapshot.nowUtc).toMatch(/^\d{4}-\d{2}-\d{2}T/);
    expect(['Sunday','Monday','Tuesday','Wednesday','Thursday','Friday','Saturday'])
      .toContain(snapshot.dayName);
  });

  it('reports zero urgent tasks when queue is empty', () => {
    const snapshot = ctx.assembleSnapshot();
    expect(snapshot.urgentTasks.length).toBe(0);
    expect(snapshot.overdueTasks).toBe(0);
    expect(snapshot.tasksDueToday).toBe(0);
  });

  it('reports overdue task count correctly', () => {
    store.createTask(makeTask({ deadline: pastIso(2), status: 'pending' }));
    store.createTask(makeTask({ deadline: futureIso(2), status: 'pending' }));
    const snapshot = ctx.assembleSnapshot();
    expect(snapshot.overdueTasks).toBe(1);
  });

  it('surfaces urgent tasks up to maxUrgentTasks', () => {
    for (let i = 0; i < 10; i++) {
      store.createTask(makeTask({ priority: 'high', deadline: futureIso(1 + i) }));
    }
    const snapshot = ctx.assembleSnapshot({ maxUrgentTasks: 3 });
    expect(snapshot.urgentTasks.length).toBeLessThanOrEqual(3);
  });

  it('marks overdue tasks correctly in urgentTasks list', () => {
    store.createTask(makeTask({ priority: 'high', deadline: pastIso(1), status: 'pending' }));
    const snapshot = ctx.assembleSnapshot();
    const overdueTasks = snapshot.urgentTasks.filter((t) => t.isOverdue);
    expect(overdueTasks.length).toBeGreaterThanOrEqual(1);
  });

  it('includes next scheduled job when schedules exist', () => {
    // Seed a schedule with a known next run time
    const schedule = store.createSchedule({
      name: 'Test Schedule',
      cronExpression: '0 3 * * *',
      jobType: 'autodream',
      payload: {},
      status: 'active',
      timezone: 'UTC',
      timeoutMinutes: 30,
    });
    store.updateSchedule(schedule.id, { nextRunAt: futureIso(5) });

    const snapshot = ctx.assembleSnapshot({ includeNextJob: true });
    expect(snapshot.nextScheduledJob).toBeDefined();
    expect(snapshot.nextScheduledJob!.jobType).toBe('autodream');
  });

  it('snapshot.estimatedTokens is a positive integer', () => {
    const snapshot = ctx.assembleSnapshot();
    expect(snapshot.estimatedTokens).toBeGreaterThan(0);
    expect(Number.isInteger(snapshot.estimatedTokens)).toBe(true);
  });
});

// ─── format ──────────────────────────────────────────────────────────────────

describe('TemporalContext.format', () => {
  it('returns a string containing <temporal_context> tags', () => {
    const snapshot = ctx.assembleSnapshot();
    const formatted = ctx.format(snapshot);
    expect(formatted).toContain('<temporal_context>');
    expect(formatted).toContain('</temporal_context>');
  });

  it('includes time in the output', () => {
    const snapshot = ctx.assembleSnapshot();
    const formatted = ctx.format(snapshot);
    expect(formatted).toContain('Time:');
  });

  it('includes overdue warning when tasks are overdue', () => {
    store.createTask(makeTask({ deadline: pastIso(2), priority: 'high', status: 'pending' }));
    const snapshot = ctx.assembleSnapshot();
    const formatted = ctx.format(snapshot);
    expect(formatted).toContain('overdue');
  });

  it('returns minimal block when over token budget', () => {
    const snapshot = ctx.assembleSnapshot();
    // Force over budget
    const minimal = ctx.format(snapshot, { maxTokens: 1 });
    expect(minimal).toContain('<temporal_context>');
    expect(minimal.length).toBeLessThan(200);
  });

  it('includes next job when present in snapshot', () => {
    const schedule = store.createSchedule({
      name: 'Nightly AutoDream',
      cronExpression: '0 2 * * *',
      jobType: 'autodream',
      payload: {},
      status: 'active',
      timezone: 'UTC',
      timeoutMinutes: 60,
    });
    store.updateSchedule(schedule.id, { nextRunAt: futureIso(3) });

    const snapshot = ctx.assembleSnapshot({ includeNextJob: true });
    const formatted = ctx.format(snapshot);
    expect(formatted).toContain('Nightly AutoDream');
  });
});

// ─── getContextString ────────────────────────────────────────────────────────

describe('TemporalContext.getContextString', () => {
  it('returns a non-empty string', () => {
    const result = ctx.getContextString();
    expect(typeof result).toBe('string');
    expect(result.length).toBeGreaterThan(0);
  });

  it('never throws even with no data', () => {
    expect(() => ctx.getContextString()).not.toThrow();
  });

  it('respects maxTokens option', () => {
    const result = ctx.getContextString({ maxTokens: 50 });
    expect(result).toContain('<temporal_context>');
    // With maxTokens=50 (~200 chars), should be very short
    expect(result.length).toBeLessThan(500);
  });
});

// ─── extractObservations ─────────────────────────────────────────────────────

describe('TemporalContext.extractObservations', () => {
  it('extracts a basic [OBSERVE: content] marker', () => {
    const msg = 'Some text. [OBSERVE: ToolRegistry is too large and needs splitting]';
    const obs = ctx.extractObservations(msg);
    expect(obs.length).toBe(1);
    expect(obs[0].content).toBe('ToolRegistry is too large and needs splitting');
    expect(obs[0].importance).toBe(3); // default
  });

  it('extracts [OBSERVE:5: content] with custom importance', () => {
    const msg = '[OBSERVE:5: Critical security bug found in auth layer]';
    const obs = ctx.extractObservations(msg);
    expect(obs.length).toBe(1);
    expect(obs[0].importance).toBe(5);
  });

  it('extracts [OBSERVE:1: content] with low importance', () => {
    const msg = '[OBSERVE:1: Minor comment typo fixed]';
    const obs = ctx.extractObservations(msg);
    expect(obs[0].importance).toBe(1);
  });

  it('extracts multiple markers from a single message', () => {
    const msg = [
      '[OBSERVE: First observation]',
      'Some content in between.',
      '[OBSERVE:4: Second observation with high importance]',
      '[OBSERVE:2: Third observation]',
    ].join('\n');
    const obs = ctx.extractObservations(msg);
    expect(obs.length).toBe(3);
    expect(obs[1].importance).toBe(4);
  });

  it('sets the provided source on all extracted observations', () => {
    const obs = ctx.extractObservations('[OBSERVE: test]', 'tool_result');
    expect(obs[0].source).toBe('tool_result');
  });

  it('sets sessionId when provided', () => {
    const obs = ctx.extractObservations('[OBSERVE: test]', 'agent_insight', 'session-123');
    expect(obs[0].sessionId).toBe('session-123');
  });

  it('returns empty array when no markers present', () => {
    const obs = ctx.extractObservations('No markers in this message at all.');
    expect(obs.length).toBe(0);
  });

  it('ignores empty content markers', () => {
    const obs = ctx.extractObservations('[OBSERVE: ]');
    expect(obs.length).toBe(0);
  });

  it('clamps importance to valid range 1-5', () => {
    // Regex only captures 1-5 in the group, so invalid values just won't match
    const obs = ctx.extractObservations('[OBSERVE:3: valid importance]');
    expect(obs[0].importance).toBe(3);
  });

  it('sets dateBucket to today', () => {
    const today = new Date().toISOString().slice(0, 10);
    const obs = ctx.extractObservations('[OBSERVE: test]');
    expect(obs[0].dateBucket).toBe(today);
  });
});

// ─── isWorkingHours / isOvernightWindow ───────────────────────────────────────

describe('TemporalContext — time window helpers', () => {
  it('isWorkingHours returns a boolean', () => {
    const result = ctx.isWorkingHours();
    expect(typeof result).toBe('boolean');
  });

  it('isOvernightWindow returns a boolean', () => {
    const result = ctx.isOvernightWindow();
    expect(typeof result).toBe('boolean');
  });

  it('isWorkingHours uses custom window when provided', () => {
    // Always working hours: 0-24
    const alwaysWorking = ctx.isWorkingHours({ workingHours: { start: 0, end: 23 } });
    expect(alwaysWorking).toBe(true);
  });

  it('isOvernightWindow uses custom window when provided', () => {
    // Never overnight: { start: 25, end: 25 } is impossible, so use a tiny window
    // that definitely does NOT include current hour
    const neverOvernight = ctx.isOvernightWindow({
      overnightWindow: { start: 25, end: 25 },
    });
    // With { start: 25, end: 25 }, start > max hour 23, so never matches
    expect(neverOvernight).toBe(false);
  });
});

// ─── refreshAllScheduleNextRuns ───────────────────────────────────────────────

describe('TemporalContext.refreshAllScheduleNextRuns', () => {
  it('updates nextRunAt for active schedules', () => {
    const schedule = store.createSchedule({
      name: 'Test',
      cronExpression: '0 * * * *',
      jobType: 'memory_compact',
      payload: {},
      status: 'active',
      timezone: 'UTC',
      timeoutMinutes: 10,
    });

    // Initially nextRunAt may be null
    expect(store.getSchedule(schedule.id)?.nextRunAt).toBeUndefined();

    ctx.refreshAllScheduleNextRuns();

    const updated = store.getSchedule(schedule.id);
    expect(updated?.nextRunAt).toBeDefined();
    expect(new Date(updated!.nextRunAt!).getTime()).toBeGreaterThan(Date.now());
  });

  it('does not throw when no schedules exist', () => {
    expect(() => ctx.refreshAllScheduleNextRuns()).not.toThrow();
  });
});
Pass 6 — Part 2 Complete ✅
Full package summary
File	Purpose	Lines (approx)
Part 1		
package.json	Deps: better-sqlite3, zod; peers: core, memory(opt), wiki(opt)	45
tsconfig.json	Compiler config with path aliases for all workspace packages	22
src/types/kairos.types.ts	Full type ABI: tasks, schedules, ticks, observations, Zod schemas, urgency constants, default schedule seeds	~370
src/KairosStore.ts	SQLite persistence — 6-table schema, WAL, full task/schedule/observation/tick CRUD, snooze wakeup, blocker resolution, auto-fail, urgency-sorted queries	~490
src/TaskQueue.ts	Pure in-memory max-heap — computeUrgency(), push/pop/peek/remove/refresh/topN/getOverdueTasks/getSummary, deadline + creation tiebreaking	~280
src/CronScheduler.ts	Zero-dep cron parser — step/range/list/alias syntax, nextDate() with fast-forward, getDueSchedules(), describe(), nextNDates()	~340
src/TemporalContext.ts	Time-awareness injection — snapshot assembly, token-budgeted <temporal_context> block, [OBSERVE:] extraction, getContextString(), schedule refresh	~320
Part 2		
src/TickEngine.ts	Proactive tick loop — KAIROS <tick>HH:MM:SS</tick> pattern, interruptible setTimeout chain, adaptive interval, idle slowdown, overnight sleeping, priority escalation on overdue tasks, mode event emission	~290
src/ObservationLog.ts	Append-only daily log — typed append operations by source, date-bucketed retrieval, DailySummary + WeeklyDigest generation, consolidation management, formatted today summary	~260
src/KairosAgent.ts	Top-level orchestrator — EventBus bridge, HooksRegistry temporal injection, task management API, job handler registry, schedule runner with timeout race, full EventBus event routing	~310
src/KairosReporter.ts	KAIROS_REPORT.md generator — overdue/pending/completed tables, schedule calendar with preview, observation digest, tick audit, suggested actions, compact summary for agent context	~280
src/index.ts	Complete barrel export — all classes, types, schemas, constants, pure functions	~80
src/tests/KairosStore.test.ts	35 tests — init, CRUD, transitions, queries, wakeup/unblock/auto-fail, observations, tick log, stats	~240
src/tests/TaskQueue.test.ts	30 tests — computeUrgency scoring, heap ordering, priority + deadline tiebreaking, overdue queries, populate, topN, refresh, getSummary	~220
src/tests/CronScheduler.test.ts	32 tests — all parse syntaxes, validation, matches, nextDate boundary cases (month/year), nextNDates, getDueSchedules, describe	~230
src/tests/TemporalContext.test.ts	28 tests — assembleSnapshot, format, getContextString, [OBSERVE:] extraction (importance, multi-marker, sessionId, empty), time window helpers, refreshAllScheduleNextRuns	~220
Cross-package consistency summary
Convention	How Pass 6 upholds it
Column naming	tags_csv, session_id, created_at/updated_at, from_id/to_id style (e.g. blocker_task_id) — identical to WikiStore + GraphBuilder
WAL + pragmas	Same journal_mode=WAL, synchronous=NORMAL, foreign_keys=ON, cache_size=-8000, temp_store=MEMORY as all other packages
Fingerprints	SHA-256 hex 64-char on content where applicable — same as WikiParser, IncrementalUpdater
[MARKER: content] pattern	[OBSERVE: ...] parallels [REMEMBER: ...] (memory) and [WIKI: ...] (wiki) for consistent agent control surface
EventBusLike shim	No hard compile-time dep on packages/core — same decoupling pattern as WikiSyncAgent
Append-only audit logs	kairos_observations and kairos_tick_log are never updated/deleted — same philosophy as wiki_history
Seeded default data	Default schedules (AutoDream 2am, Graph Rebuild 2:30am, Wiki Export Sunday, Memory Compact hourly) align exactly with AutoDream scheduling described in Pass 3
Pure computation classes	TaskQueue, CronScheduler, computeUrgency() have zero I/O — same pattern as LeidenClusterer pure logic in Pass 4
KAIROS tick format	<tick>HH:MM:SS</tick> content with priority 'later'/'soon'/'now' — faithful clean-room re-implementation of the Claude Code KAIROS pattern

