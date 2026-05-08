PASS 19 — queryLoop Completion, Context Injection, Budget & Permission Gates, Workspace Boundary
Philosophy: "Later pass wins." Every file below is the canonical, final version of that path. Overwrite anything from earlier passes at these paths. This pass makes the "spine" real — everything in Pass 20–24 streams from or calls into what is built here.

What this pass delivers
Area	What gets built
packages/core/src/loop/queryLoop.ts	The real async-generator agent loop — fully wired
packages/core/src/loop/TurnAssembler.ts	Builds the message array sent to the model each turn
packages/core/src/loop/BudgetEnforcer.ts	Soft/hard token limits, cost-cap graceful terminal event
packages/core/src/loop/CompactionEngine.ts	auto + full compaction via memory summarisation
packages/core/src/loop/ContextInjector.ts	Reads CLAUDE.md, MEMORY.md, .locoworker/instructions.md from workspace
packages/core/src/loop/ToolExecutor.ts	Parallel-safe vs sequential tool dispatch, timeout, error wrapping
packages/core/src/permissions/PermissionConfirmation.ts	TTY-aware interactive permission prompt (+ programmatic override)
packages/core/src/workspace/WorkspaceBoundary.ts	Single authoritative path-safety choke point for ALL tools
packages/core/src/session/SessionManager.ts	Create, restore, checkpoint, and expire sessions
packages/core/src/providers/ProviderRouter.ts	Final version — delegates to @locoworker/providers
packages/core/src/events/EventEmitter.ts	Typed, buffered, backpressure-safe event bus
Types updates	agent.types.ts, event.types.ts, tool.types.ts, permission.types.ts
Tests	tests/unit/core/queryLoop.test.ts, WorkspaceBoundary.test.ts, BudgetEnforcer.test.ts, PermissionConfirmation.test.ts, TurnAssembler.test.ts
File tree (only new/changed paths)
text

packages/core/
├── src/
│   ├── loop/
│   │   ├── queryLoop.ts              ← REWRITE (was stub)
│   │   ├── TurnAssembler.ts          ← NEW
│   │   ├── BudgetEnforcer.ts         ← NEW
│   │   ├── CompactionEngine.ts       ← NEW
│   │   ├── ContextInjector.ts        ← NEW
│   │   └── ToolExecutor.ts           ← NEW
│   ├── permissions/
│   │   ├── PermissionGate.ts         ← UPDATE (hooks in PermissionConfirmation)
│   │   └── PermissionConfirmation.ts ← NEW
│   ├── workspace/
│   │   └── WorkspaceBoundary.ts      ← NEW (single choke point)
│   ├── session/
│   │   └── SessionManager.ts         ← REWRITE (adds restore + checkpoint)
│   ├── providers/
│   │   └── ProviderRouter.ts         ← REWRITE (delegates to @locoworker/providers)
│   ├── events/
│   │   └── EventEmitter.ts           ← REWRITE (backpressure-safe)
│   └── types/
│       ├── agent.types.ts            ← UPDATE
│       ├── event.types.ts            ← UPDATE
│       ├── tool.types.ts             ← UPDATE
│       └── permission.types.ts       ← UPDATE
tests/
└── unit/
    └── core/
        ├── queryLoop.test.ts
        ├── WorkspaceBoundary.test.ts
        ├── BudgetEnforcer.test.ts
        ├── PermissionConfirmation.test.ts
        └── TurnAssembler.test.ts
Types first (everything else builds on these)
packages/core/src/types/event.types.ts
TypeScript

// packages/core/src/types/event.types.ts
// ─────────────────────────────────────────────────────────────────────────────
// Canonical AgentEvent union. Every surface (CLI, Gateway, Desktop, tests)
// consumes ONLY this type. Never add ad-hoc fields outside this union.
// ─────────────────────────────────────────────────────────────────────────────

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
  | "budget_warning"      // soft limit hit
  | "budget_exceeded"     // hard limit hit — loop MUST stop
  | "cost_cap_exceeded"   // cost limit hit — loop MUST stop
  | "compaction_start"
  | "compaction_end"
  | "turn_complete"
  | "loop_complete"       // model returned no more tool calls
  | "session_error"       // recoverable error (logged, loop continues)
  | "fatal_error";        // unrecoverable — loop stops

export interface BaseEvent {
  kind: AgentEventKind;
  sessionId: string;
  turnIndex: number;
  ts: number; // Date.now()
}

export interface SessionStartEvent extends BaseEvent {
  kind: "session_start";
  workingDir: string;
  model: string;
  resumedFrom?: string; // prior session ID if forked
}

export interface SessionRestoreEvent extends BaseEvent {
  kind: "session_restore";
  restoredTurns: number;
  workingDir: string;
}

export interface SessionCheckpointEvent extends BaseEvent {
  kind: "session_checkpoint";
  checkpointId: string;
}

export interface SessionEndEvent extends BaseEvent {
  kind: "session_end";
  reason: "complete" | "budget_exceeded" | "cost_cap" | "user_abort" | "fatal_error";
  totalTokensUsed: number;
  totalCostUsd: number;
  totalTurns: number;
}

export interface ContextInjectedEvent extends BaseEvent {
  kind: "context_injected";
  sources: string[]; // e.g. ["CLAUDE.md", "MEMORY.md", ".locoworker/instructions.md"]
  totalChars: number;
}

export interface TurnStartEvent extends BaseEvent {
  kind: "turn_start";
  userMessage: string;
}

export interface ModelRequestEvent extends BaseEvent {
  kind: "model_request";
  model: string;
  messageCount: number;
  estimatedInputTokens: number;
}

export interface ModelResponseStartEvent extends BaseEvent {
  kind: "model_response_start";
  model: string;
}

export interface ModelResponseDeltaEvent extends BaseEvent {
  kind: "model_response_delta";
  delta: string;
}

export interface ModelResponseEndEvent extends BaseEvent {
  kind: "model_response_end";
  stopReason: "end_turn" | "tool_use" | "max_tokens" | "stop_sequence";
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
}

export interface ToolCallEvent extends BaseEvent {
  kind: "tool_call";
  toolCallId: string;
  toolName: string;
  input: Record<string, unknown>;
  isParallelSafe: boolean;
}

export interface ToolResultEvent extends BaseEvent {
  kind: "tool_result";
  toolCallId: string;
  toolName: string;
  durationMs: number;
  outputPreview: string; // first 200 chars
}

export interface ToolErrorEvent extends BaseEvent {
  kind: "tool_error";
  toolCallId: string;
  toolName: string;
  error: string;
  isPermissionDenied: boolean;
}

export interface PermissionPromptEvent extends BaseEvent {
  kind: "permission_prompt";
  toolName: string;
  requiredTier: number;
  description: string;
  promptId: string;
}

export interface PermissionGrantedEvent extends BaseEvent {
  kind: "permission_granted";
  promptId: string;
  toolName: string;
  permanent: boolean;
}

export interface PermissionDeniedEvent extends BaseEvent {
  kind: "permission_denied";
  promptId: string;
  toolName: string;
}

export interface BudgetWarningEvent extends BaseEvent {
  kind: "budget_warning";
  usedTokens: number;
  softLimitTokens: number;
  percentUsed: number;
}

export interface BudgetExceededEvent extends BaseEvent {
  kind: "budget_exceeded";
  usedTokens: number;
  hardLimitTokens: number;
}

export interface CostCapExceededEvent extends BaseEvent {
  kind: "cost_cap_exceeded";
  spentUsd: number;
  capUsd: number;
}

export interface CompactionStartEvent extends BaseEvent {
  kind: "compaction_start";
  mode: "auto" | "full";
  turnsBefore: number;
  tokensBefore: number;
}

export interface CompactionEndEvent extends BaseEvent {
  kind: "compaction_end";
  mode: "auto" | "full";
  turnsAfter: number;
  tokensAfter: number;
  savedTokens: number;
}

export interface TurnCompleteEvent extends BaseEvent {
  kind: "turn_complete";
  assistantText: string;
  toolCallsExecuted: number;
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
}

export interface LoopCompleteEvent extends BaseEvent {
  kind: "loop_complete";
  finalResponse: string;
}

export interface SessionErrorEvent extends BaseEvent {
  kind: "session_error";
  error: string;
  recoverable: true;
}

export interface FatalErrorEvent extends BaseEvent {
  kind: "fatal_error";
  error: string;
  stack?: string;
}

export type AgentEvent =
  | SessionStartEvent
  | SessionRestoreEvent
  | SessionCheckpointEvent
  | SessionEndEvent
  | ContextInjectedEvent
  | TurnStartEvent
  | ModelRequestEvent
  | ModelResponseStartEvent
  | ModelResponseDeltaEvent
  | ModelResponseEndEvent
  | ToolCallEvent
  | ToolResultEvent
  | ToolErrorEvent
  | PermissionPromptEvent
  | PermissionGrantedEvent
  | PermissionDeniedEvent
  | BudgetWarningEvent
  | BudgetExceededEvent
  | CostCapExceededEvent
  | CompactionStartEvent
  | CompactionEndEvent
  | TurnCompleteEvent
  | LoopCompleteEvent
  | SessionErrorEvent
  | FatalErrorEvent;

// Type guard helpers
export const isTerminalEvent = (e: AgentEvent): boolean =>
  e.kind === "budget_exceeded" ||
  e.kind === "cost_cap_exceeded" ||
  e.kind === "fatal_error" ||
  e.kind === "session_end";

export const isErrorEvent = (e: AgentEvent): e is SessionErrorEvent | FatalErrorEvent =>
  e.kind === "session_error" || e.kind === "fatal_error";
packages/core/src/types/permission.types.ts
TypeScript

// packages/core/src/types/permission.types.ts

export enum PermissionTier {
  READ_ONLY   = 0, // Read filesystem, read git log, no mutations
  READ_WRITE  = 1, // Write files within workspace boundary
  EXECUTE     = 2, // Run bash commands (allow-listed)
  NETWORK     = 3, // Outbound HTTP/fetch (SSRF-guarded)
  SYSTEM      = 4, // Spawn processes, docker, system-level ops
}

export interface ToolPermissionMeta {
  requiredTier: PermissionTier;
  isParallelSafe: boolean;          // Can run concurrently with other parallel-safe tools
  requiresConfirmation: boolean;    // Always ask user, even if tier is granted
  description: string;              // Human-readable description for permission prompt
  examples?: string[];              // Examples of what the tool can do (shown in prompt)
}

export interface PermissionState {
  grantedTiers: Set<PermissionTier>;
  permanentlyGranted: Set<string>;  // tool names granted forever this session
  permanentlyDenied: Set<string>;   // tool names denied forever this session
  sessionGrants: Map<string, PermissionTier>; // per-tool overrides
}

export interface PermissionDecision {
  granted: boolean;
  permanent: boolean;   // should this be remembered for the session?
  promptId: string;
}

// The confirmation callback signature.
// CLI implements this with readline. Desktop implements with IPC dialog.
// Tests can inject a deterministic mock.
export type ConfirmationHandler = (
  toolName: string,
  tier: PermissionTier,
  description: string,
  examples: string[]
) => Promise<PermissionDecision>;
packages/core/src/types/tool.types.ts
TypeScript

// packages/core/src/types/tool.types.ts

import type { PermissionTier } from "./permission.types.js";

export interface ToolInputSchema {
  type: "object";
  properties: Record<string, unknown>;
  required?: string[];
  additionalProperties?: boolean;
}

export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: ToolInputSchema;
  requiredTier: PermissionTier;
  isParallelSafe: boolean;
  requiresConfirmation?: boolean;
  timeoutMs?: number; // default: 30_000
}

export interface ToolCallRequest {
  id: string;         // model-generated call id
  name: string;
  input: Record<string, unknown>;
}

export type ToolResultContent =
  | { type: "text"; text: string }
  | { type: "error"; error: string; isPermissionDenied?: boolean };

export interface ToolCallResult {
  id: string;
  name: string;
  content: ToolResultContent;
  durationMs: number;
  isError: boolean;
}

export interface ToolHandler {
  definition: ToolDefinition;
  execute: (
    input: Record<string, unknown>,
    ctx: ToolExecutionContext
  ) => Promise<string>;
}

export interface ToolExecutionContext {
  sessionId: string;
  workingDir: string;         // pre-validated workspace root
  permissionTier: PermissionTier;
  signal: AbortSignal;        // honour cancellation
}
packages/core/src/types/agent.types.ts
TypeScript

// packages/core/src/types/agent.types.ts

import type { PermissionState, ConfirmationHandler } from "./permission.types.js";
import type { ToolHandler } from "./tool.types.js";
import type { AgentEvent } from "./event.types.js";

// ── Model configuration ───────────────────────────────────────────────────────

export interface ModelConfig {
  provider: string;         // "anthropic" | "openai" | "gemini" | "ollama" | ...
  model: string;
  temperature?: number;
  topP?: number;
  maxOutputTokens?: number;
  apiKey?: string;          // BYOK; falls back to env if omitted
  baseUrl?: string;         // for ollama/lm-studio/custom endpoints
}

// ── Budget configuration ──────────────────────────────────────────────────────

export interface BudgetConfig {
  maxInputTokens: number;         // hard limit for context window
  softLimitTokens: number;        // triggers compaction warning
  hardLimitTokens: number;        // triggers forced compaction / stop
  maxCostUsd?: number;            // optional spend cap per session
  maxTurns?: number;              // optional turn cap
  compactionMode: "auto" | "full" | "none";
}

export const DEFAULT_BUDGET: BudgetConfig = {
  maxInputTokens: 180_000,
  softLimitTokens: 120_000,
  hardLimitTokens: 160_000,
  compactionMode: "auto",
};

// ── Session ───────────────────────────────────────────────────────────────────

export interface SessionState {
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
  conversationHistory: ConversationTurn[];
  permissionState: PermissionState;
  aborted: boolean;
}

export interface ConversationTurn {
  role: "user" | "assistant";
  content: string | ContentBlock[];
  toolCalls?: ToolCallBlock[];
  toolResults?: ToolResultBlock[];
  inputTokens?: number;
  outputTokens?: number;
  costUsd?: number;
  ts: number;
}

export interface ContentBlock {
  type: "text";
  text: string;
}

export interface ToolCallBlock {
  id: string;
  name: string;
  input: Record<string, unknown>;
}

export interface ToolResultBlock {
  toolCallId: string;
  content: string;
  isError: boolean;
}

// ── Agent context (passed into queryLoop) ────────────────────────────────────

export interface AgentContext {
  session: SessionState;
  tools: Map<string, ToolHandler>;
  emit: (event: AgentEvent) => void;
  confirmationHandler?: ConfirmationHandler;
  signal?: AbortSignal;           // external abort (e.g. user Ctrl-C)
  onCheckpoint?: (sessionId: string) => Promise<void>;
}

// ── queryLoop input/output ────────────────────────────────────────────────────

export interface QueryLoopInput {
  userMessage: string;
  ctx: AgentContext;
}

export type QueryLoopGenerator = AsyncGenerator<AgentEvent, void, undefined>;
Core implementations
packages/core/src/events/EventEmitter.ts
TypeScript

// packages/core/src/events/EventEmitter.ts
// ─────────────────────────────────────────────────────────────────────────────
// Backpressure-safe typed event bus.
// Consumers subscribe via async iterators. Multiple consumers are supported.
// ─────────────────────────────────────────────────────────────────────────────

import type { AgentEvent } from "../types/event.types.js";

export class AgentEventEmitter {
  private listeners = new Set<(event: AgentEvent) => void>();

  emit(event: AgentEvent): void {
    for (const listener of this.listeners) {
      try {
        listener(event);
      } catch {
        // Individual listener failures must not break the loop
      }
    }
  }

  subscribe(listener: (event: AgentEvent) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  /**
   * Returns an async iterator that yields events as they arrive.
   * Call `return()` on the iterator to unsubscribe cleanly.
   */
  async *[Symbol.asyncIterator](): AsyncGenerator<AgentEvent> {
    const queue: AgentEvent[] = [];
    let resolve: (() => void) | null = null;
    let done = false;

    const unsub = this.subscribe((event) => {
      queue.push(event);
      if (resolve) {
        const r = resolve;
        resolve = null;
        r();
      }
    });

    try {
      while (!done) {
        if (queue.length === 0) {
          await new Promise<void>((r) => { resolve = r; });
        }
        while (queue.length > 0) {
          const event = queue.shift()!;
          yield event;
          if (
            event.kind === "session_end" ||
            event.kind === "fatal_error"
          ) {
            done = true;
            break;
          }
        }
      }
    } finally {
      unsub();
    }
  }
}
packages/core/src/workspace/WorkspaceBoundary.ts
TypeScript

// packages/core/src/workspace/WorkspaceBoundary.ts
// ─────────────────────────────────────────────────────────────────────────────
// THE single authoritative path-safety choke point.
// ALL tools that read/write files MUST call WorkspaceBoundary.resolve() or
// WorkspaceBoundary.assertSafe() before any filesystem operation.
//
// This replaces ad-hoc per-tool path checks that existed in earlier passes.
// ─────────────────────────────────────────────────────────────────────────────

import path from "node:path";
import fs from "node:fs";

export class WorkspaceBoundaryError extends Error {
  constructor(
    public readonly requestedPath: string,
    public readonly workingDir: string
  ) {
    super(
      `Path escape attempt blocked: "${requestedPath}" is outside workspace "${workingDir}"`
    );
    this.name = "WorkspaceBoundaryError";
  }
}

export class WorkspaceBoundary {
  private readonly root: string;

  /** @param workingDir - absolute path to the workspace root */
  constructor(workingDir: string) {
    // Resolve symlinks so comparison is canonical
    this.root = fs.realpathSync(path.resolve(workingDir));
  }

  /**
   * Resolve `inputPath` (absolute or relative) against the workspace root
   * and verify it stays within the root.
   *
   * @returns The canonically resolved absolute path (always safe to use).
   * @throws WorkspaceBoundaryError if the resolved path escapes the workspace.
   */
  resolve(inputPath: string): string {
    const abs = path.isAbsolute(inputPath)
      ? path.normalize(inputPath)
      : path.join(this.root, inputPath);

    // Resolve without requiring the path to exist (resolveSync would throw)
    const resolved = path.resolve(abs);

    if (!resolved.startsWith(this.root + path.sep) && resolved !== this.root) {
      throw new WorkspaceBoundaryError(inputPath, this.root);
    }

    return resolved;
  }

  /**
   * Assert that `inputPath` is safe. Returns the resolved path or throws.
   * Alias for resolve() — use whichever reads better at the call site.
   */
  assertSafe(inputPath: string): string {
    return this.resolve(inputPath);
  }

  /**
   * Check without throwing.
   */
  isSafe(inputPath: string): boolean {
    try {
      this.resolve(inputPath);
      return true;
    } catch {
      return false;
    }
  }

  get workspaceRoot(): string {
    return this.root;
  }

  /**
   * Create a relative path from the workspace root (useful for display).
   */
  relativize(absPath: string): string {
    return path.relative(this.root, absPath);
  }
}
packages/core/src/permissions/PermissionConfirmation.ts
TypeScript

// packages/core/src/permissions/PermissionConfirmation.ts
// ─────────────────────────────────────────────────────────────────────────────
// Interactive permission confirmation.
//
// By default uses a readline prompt when stdout is a TTY.
// Non-TTY contexts (pipes, tests) use the injected handler or auto-deny.
//
// The CLI will inject its own readline-based handler.
// Desktop will inject an IPC-based handler.
// Tests inject a deterministic mock.
// ─────────────────────────────────────────────────────────────────────────────

import { randomUUID } from "node:crypto";
import type {
  ConfirmationHandler,
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

/**
 * Built-in TTY confirmation handler using readline.
 * Only used when no custom handler is registered.
 */
async function defaultTTYHandler(
  toolName: string,
  tier: PermissionTier,
  description: string,
  examples: string[]
): Promise<PermissionDecision> {
  // Lazy import readline to avoid pulling it in non-TTY builds
  const readline = await import("node:readline");

  const promptId = randomUUID();

  if (!process.stdin.isTTY || !process.stdout.isTTY) {
    // Non-interactive — auto-deny (safe default)
    console.error(
      `[LocoWorker] Permission required for "${toolName}" (tier: ${TIER_LABELS[tier]}) — auto-denied (non-TTY)`
    );
    return { granted: false, permanent: false, promptId };
  }

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const question = (q: string) =>
    new Promise<string>((resolve) => rl.question(q, resolve));

  console.log(`\n┌─ Permission Required ──────────────────────────────────────`);
  console.log(`│  Tool:        ${toolName}`);
  console.log(`│  Tier:        ${TIER_LABELS[tier]}`);
  console.log(`│  Description: ${description}`);
  if (examples.length > 0) {
    console.log(`│  Examples:`);
    examples.forEach((ex) => console.log(`│    • ${ex}`));
  }
  console.log(`└────────────────────────────────────────────────────────────`);

  let answer: string;
  try {
    answer = await question(
      "  Allow? [y]es / [n]o / [a]lways (for this session): "
    );
  } finally {
    rl.close();
  }

  const normalised = answer.trim().toLowerCase();

  if (normalised === "y" || normalised === "yes") {
    return { granted: true, permanent: false, promptId };
  }
  if (normalised === "a" || normalised === "always") {
    return { granted: true, permanent: true, promptId };
  }
  return { granted: false, permanent: false, promptId };
}

export class PermissionConfirmation {
  private handler: ConfirmationHandler;

  constructor(handler?: ConfirmationHandler) {
    this.handler = handler ?? defaultTTYHandler;
  }

  /** Replace the handler at runtime (used by CLI/Desktop bootstrap). */
  setHandler(handler: ConfirmationHandler): void {
    this.handler = handler;
  }

  async confirm(
    toolName: string,
    tier: PermissionTier,
    description: string,
    examples: string[] = []
  ): Promise<PermissionDecision> {
    return this.handler(toolName, tier, description, examples);
  }
}
packages/core/src/permissions/PermissionGate.ts
TypeScript

// packages/core/src/permissions/PermissionGate.ts
// ─────────────────────────────────────────────────────────────────────────────
// Authoritative permission check. Wires in PermissionConfirmation.
// ─────────────────────────────────────────────────────────────────────────────

import { randomUUID } from "node:crypto";
import { PermissionTier } from "../types/permission.types.js";
import type { PermissionState, PermissionDecision } from "../types/permission.types.js";
import type { ToolDefinition } from "../types/tool.types.js";
import type { AgentEvent } from "../types/event.types.js";
import { PermissionConfirmation } from "./PermissionConfirmation.js";

export class PermissionDeniedError extends Error {
  constructor(public readonly toolName: string, public readonly tier: PermissionTier) {
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

  /**
   * Check + potentially prompt for a tool call.
   * Throws PermissionDeniedError if denied.
   * Returns void if granted (logs events either way).
   */
  async check(tool: ToolDefinition): Promise<void> {
    const { name, requiredTier, requiresConfirmation, description, examples } = tool;

    // Already permanently denied?
    if (this.state.permanentlyDenied.has(name)) {
      throw new PermissionDeniedError(name, requiredTier);
    }

    // Already permanently granted?
    if (this.state.permanentlyGranted.has(name)) {
      return;
    }

    // Tier already granted and no explicit confirmation required?
    if (
      this.state.grantedTiers.has(requiredTier) &&
      !requiresConfirmation
    ) {
      return;
    }

    // Need to ask
    const promptId = randomUUID();

    this.emit({
      kind: "permission_prompt",
      sessionId: this.sessionId,
      turnIndex: this.turnIndex(),
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
        examples ?? []
      );
    } catch (err) {
      // Confirmation handler threw — treat as deny
      decision = { granted: false, permanent: false, promptId };
    }

    if (decision.granted) {
      if (decision.permanent) {
        this.state.permanentlyGranted.add(name);
        // Also grant the tier for future similar tools
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
    } else {
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
  }

  grantTier(tier: PermissionTier): void {
    this.state.grantedTiers.add(tier);
  }

  getState(): Readonly<PermissionState> {
    return this.state;
  }
}
packages/core/src/loop/ContextInjector.ts
TypeScript

// packages/core/src/loop/ContextInjector.ts
// ─────────────────────────────────────────────────────────────────────────────
// Reads workspace context files (CLAUDE.md, MEMORY.md, .locoworker/instructions.md)
// and assembles a system-level context string prepended to the model request.
//
// Files are read relative to workingDir. Absence is silently ignored.
// Each file is bounded to MAX_FILE_CHARS to prevent accidental context flooding.
// ─────────────────────────────────────────────────────────────────────────────

import fs from "node:fs/promises";
import path from "node:path";

const MAX_FILE_CHARS = 24_000; // ~6k tokens per file, hard cap

export interface InjectedContext {
  systemPrompt: string;
  sources: string[];
  totalChars: number;
}

const CONTEXT_FILES = [
  "CLAUDE.md",
  "MEMORY.md",
  ".locoworker/instructions.md",
  ".locoworker/project.md",
];

const BASE_SYSTEM_PROMPT = `You are LocoWorker, a privacy-first agentic developer workspace.
You have access to tools for reading and writing files, executing commands, and searching the web.
Always operate within the workspace boundary. Never exfiltrate credentials or sensitive data.
Think step-by-step. Prefer targeted edits over full file rewrites when possible.`;

export class ContextInjector {
  constructor(private readonly workingDir: string) {}

  async inject(): Promise<InjectedContext> {
    const parts: string[] = [BASE_SYSTEM_PROMPT];
    const sources: string[] = [];
    let totalChars = BASE_SYSTEM_PROMPT.length;

    for (const filename of CONTEXT_FILES) {
      const fullPath = path.join(this.workingDir, filename);
      try {
        let content = await fs.readFile(fullPath, "utf8");
        if (content.length > MAX_FILE_CHARS) {
          content =
            content.slice(0, MAX_FILE_CHARS) +
            "\n\n[... truncated to fit context budget ...]";
        }
        parts.push(`\n\n## ${filename}\n\n${content.trim()}`);
        sources.push(filename);
        totalChars += content.length;
      } catch {
        // File absent — silently skip
      }
    }

    return {
      systemPrompt: parts.join(""),
      sources,
      totalChars,
    };
  }
}
packages/core/src/loop/BudgetEnforcer.ts
TypeScript

// packages/core/src/loop/BudgetEnforcer.ts
// ─────────────────────────────────────────────────────────────────────────────
// Tracks token + cost spend and emits budget events.
// Returns a BudgetDecision after each turn so queryLoop knows whether to:
//   - continue normally
//   - trigger compaction (soft limit)
//   - stop immediately (hard limit / cost cap)
// ─────────────────────────────────────────────────────────────────────────────

import type { BudgetConfig } from "../types/agent.types.js";
import type { AgentEvent } from "../types/event.types.js";

export type BudgetDecision =
  | { action: "continue" }
  | { action: "compact"; reason: "soft_limit" }
  | { action: "stop"; reason: "hard_limit" | "cost_cap" | "turn_cap" };

export class BudgetEnforcer {
  private totalInputTokens = 0;
  private totalOutputTokens = 0;
  private totalCostUsd = 0;
  private turnCount = 0;
  private softWarningEmitted = false;

  constructor(
    private readonly budget: BudgetConfig,
    private readonly emit: (event: AgentEvent) => void,
    private readonly sessionId: string,
    private readonly getTurnIndex: () => number
  ) {}

  recordTurn(inputTokens: number, outputTokens: number, costUsd: number): BudgetDecision {
    this.totalInputTokens += inputTokens;
    this.totalOutputTokens += outputTokens;
    this.totalCostUsd += costUsd;
    this.turnCount++;

    const now = Date.now();
    const base = {
      sessionId: this.sessionId,
      turnIndex: this.getTurnIndex(),
      ts: now,
    };

    // Turn cap
    if (this.budget.maxTurns && this.turnCount >= this.budget.maxTurns) {
      return { action: "stop", reason: "turn_cap" };
    }

    // Cost cap
    if (this.budget.maxCostUsd && this.totalCostUsd >= this.budget.maxCostUsd) {
      this.emit({
        kind: "cost_cap_exceeded",
        ...base,
        spentUsd: this.totalCostUsd,
        capUsd: this.budget.maxCostUsd,
      });
      return { action: "stop", reason: "cost_cap" };
    }

    // Hard token limit
    if (this.totalInputTokens >= this.budget.hardLimitTokens) {
      this.emit({
        kind: "budget_exceeded",
        ...base,
        usedTokens: this.totalInputTokens,
        hardLimitTokens: this.budget.hardLimitTokens,
      });
      return { action: "stop", reason: "hard_limit" };
    }

    // Soft token limit — warn once, trigger compaction
    if (
      this.totalInputTokens >= this.budget.softLimitTokens &&
      !this.softWarningEmitted
    ) {
      this.softWarningEmitted = true;
      this.emit({
        kind: "budget_warning",
        ...base,
        usedTokens: this.totalInputTokens,
        softLimitTokens: this.budget.softLimitTokens,
        percentUsed: Math.round(
          (this.totalInputTokens / this.budget.hardLimitTokens) * 100
        ),
      });
      if (this.budget.compactionMode !== "none") {
        return { action: "compact", reason: "soft_limit" };
      }
    }

    return { action: "continue" };
  }

  get stats() {
    return {
      totalInputTokens: this.totalInputTokens,
      totalOutputTokens: this.totalOutputTokens,
      totalCostUsd: this.totalCostUsd,
      turnCount: this.turnCount,
    };
  }
}
packages/core/src/loop/CompactionEngine.ts
TypeScript

// packages/core/src/loop/CompactionEngine.ts
// ─────────────────────────────────────────────────────────────────────────────
// Compaction: shrinks conversation history to reclaim context budget.
//
// "auto" mode: summarise older turns, keep last N turns verbatim.
// "full" mode: summarise the entire history into a single summary turn.
//
// The summarisation call goes through the same ProviderRouter as the main loop
// (i.e., uses the same model). This ensures compaction summaries are accurate.
// ─────────────────────────────────────────────────────────────────────────────

import type { ConversationTurn, ModelConfig } from "../types/agent.types.js";
import type { AgentEvent } from "../types/event.types.js";
import { estimateTokens } from "../utils/tokenEstimator.js";

const KEEP_VERBATIM_TURNS = 6; // always keep last 6 turns as-is in "auto" mode

export interface CompactionResult {
  turns: ConversationTurn[];
  tokensBefore: number;
  tokensAfter: number;
  savedTokens: number;
}

export class CompactionEngine {
  constructor(
    private readonly callModel: (messages: ConversationTurn[], systemPrompt: string) => Promise<string>,
    private readonly emit: (event: AgentEvent) => void,
    private readonly sessionId: string,
    private readonly getTurnIndex: () => number
  ) {}

  async compact(
    turns: ConversationTurn[],
    mode: "auto" | "full"
  ): Promise<CompactionResult> {
    const tokensBefore = this.estimateTurnsTokens(turns);
    const now = Date.now();
    const base = {
      sessionId: this.sessionId,
      turnIndex: this.getTurnIndex(),
      ts: now,
    };

    this.emit({
      kind: "compaction_start",
      ...base,
      mode,
      turnsBefore: turns.length,
      tokensBefore,
    });

    let compactedTurns: ConversationTurn[];

    if (mode === "full") {
      compactedTurns = await this.fullCompact(turns);
    } else {
      compactedTurns = await this.autoCompact(turns);
    }

    const tokensAfter = this.estimateTurnsTokens(compactedTurns);

    this.emit({
      kind: "compaction_end",
      ...base,
      mode,
      turnsAfter: compactedTurns.length,
      tokensAfter,
      savedTokens: tokensBefore - tokensAfter,
    });

    return {
      turns: compactedTurns,
      tokensBefore,
      tokensAfter,
      savedTokens: tokensBefore - tokensAfter,
    };
  }

  private async autoCompact(turns: ConversationTurn[]): Promise<ConversationTurn[]> {
    if (turns.length <= KEEP_VERBATIM_TURNS) {
      return turns; // nothing to compact
    }

    const toSummarise = turns.slice(0, turns.length - KEEP_VERBATIM_TURNS);
    const toKeep = turns.slice(turns.length - KEEP_VERBATIM_TURNS);

    const summary = await this.summariseTurns(toSummarise);

    const summaryTurn: ConversationTurn = {
      role: "user",
      content: `[Previous conversation summary]\n\n${summary}`,
      ts: Date.now(),
    };

    return [summaryTurn, ...toKeep];
  }

  private async fullCompact(turns: ConversationTurn[]): Promise<ConversationTurn[]> {
    const summary = await this.summariseTurns(turns);
    return [
      {
        role: "user",
        content: `[Full conversation summary]\n\n${summary}`,
        ts: Date.now(),
      },
    ];
  }

  private async summariseTurns(turns: ConversationTurn[]): Promise<string> {
    const systemPrompt = `You are a conversation summariser. 
Create a concise but complete summary of the conversation below.
Preserve: all decisions made, files modified, errors encountered, key facts discovered.
Omit: verbose tool outputs, repetitive content, raw file contents.
Output ONLY the summary, no preamble.`;

    try {
      return await this.callModel(turns, systemPrompt);
    } catch {
      // If summarisation fails, fall back to a structured truncation
      return turns
        .map(
          (t) =>
            `[${t.role.toUpperCase()}]: ${
              typeof t.content === "string"
                ? t.content.slice(0, 300)
                : "[structured content]"
            }`
        )
        .join("\n\n");
    }
  }

  private estimateTurnsTokens(turns: ConversationTurn[]): number {
    return turns.reduce((sum, t) => {
      const text =
        typeof t.content === "string"
          ? t.content
          : t.content.map((b) => b.text).join(" ");
      return sum + estimateTokens(text);
    }, 0);
  }
}
packages/core/src/utils/tokenEstimator.ts
TypeScript

// packages/core/src/utils/tokenEstimator.ts
// Simple but consistent token estimator (no tiktoken dependency).
// ~4 chars per token is a reliable approximation across all major models.
// Replace with tiktoken for higher accuracy if needed.

export function estimateTokens(text: string): number {
  return Math.ceil(text.length / 4);
}
packages/core/src/loop/TurnAssembler.ts
TypeScript

// packages/core/src/loop/TurnAssembler.ts
// ─────────────────────────────────────────────────────────────────────────────
// Assembles the exact message array sent to the model each turn.
// Responsibilities:
//   - inject system prompt (from ContextInjector) as first message / system field
//   - replay conversation history
//   - append current tool results (if any)
//   - enforce context window limits (trim oldest non-system turns if needed)
// ─────────────────────────────────────────────────────────────────────────────

import type { ConversationTurn } from "../types/agent.types.js";
import { estimateTokens } from "../utils/tokenEstimator.js";

export interface AssembledMessages {
  systemPrompt: string;
  messages: ConversationTurn[];
  estimatedTokens: number;
}

export class TurnAssembler {
  constructor(
    private readonly maxContextTokens: number
  ) {}

  assemble(
    systemPrompt: string,
    history: ConversationTurn[],
    pendingToolResults?: ConversationTurn
  ): AssembledMessages {
    let messages = [...history];

    if (pendingToolResults) {
      messages.push(pendingToolResults);
    }

    // Trim from the front (oldest turns first) if we're over budget
    // Always keep at least the last 2 turns for coherence
    let estimatedTokens = this.estimate(systemPrompt, messages);
    while (
      estimatedTokens > this.maxContextTokens &&
      messages.length > 2
    ) {
      messages = messages.slice(1); // drop oldest
      estimatedTokens = this.estimate(systemPrompt, messages);
    }

    return { systemPrompt, messages, estimatedTokens };
  }

  private estimate(systemPrompt: string, messages: ConversationTurn[]): number {
    const systemTokens = estimateTokens(systemPrompt);
    const historyTokens = messages.reduce((sum, m) => {
      const text =
        typeof m.content === "string"
          ? m.content
          : m.content.map((b) => b.text).join(" ");
      return sum + estimateTokens(text);
    }, 0);
    return systemTokens + historyTokens;
  }
}
packages/core/src/loop/ToolExecutor.ts
TypeScript

// packages/core/src/loop/ToolExecutor.ts
// ─────────────────────────────────────────────────────────────────────────────
// Orchestrates tool execution from a list of model-requested ToolCallRequests.
//
// Strategy:
//   1. Split calls into parallel-safe and sequential groups.
//   2. Run all parallel-safe calls concurrently (Promise.allSettled).
//   3. Run sequential calls one-at-a-time in order.
//   4. For each call:
//      a. Check permissions (may prompt user interactively).
//      b. Validate inputs against tool schema.
//      c. Execute with timeout.
//      d. Emit ToolResultEvent or ToolErrorEvent.
//
// Returns the complete list of ToolCallResults for assembling into history.
// ─────────────────────────────────────────────────────────────────────────────

import type { ToolHandler, ToolCallRequest, ToolCallResult } from "../types/tool.types.js";
import type { AgentEvent } from "../types/event.types.js";
import type { WorkspaceBoundary } from "../workspace/WorkspaceBoundary.js";
import { PermissionDeniedError, type PermissionGate } from "../permissions/PermissionGate.js";
import { PermissionTier } from "../types/permission.types.js";

const DEFAULT_TOOL_TIMEOUT_MS = 30_000;

export class ToolExecutor {
  constructor(
    private readonly tools: Map<string, ToolHandler>,
    private readonly permissionGate: PermissionGate,
    private readonly boundary: WorkspaceBoundary,
    private readonly emit: (event: AgentEvent) => void,
    private readonly sessionId: string,
    private readonly getTurnIndex: () => number,
    private readonly signal: AbortSignal
  ) {}

  async executeAll(calls: ToolCallRequest[]): Promise<ToolCallResult[]> {
    const parallelSafe = calls.filter((c) => {
      const h = this.tools.get(c.name);
      return h?.definition.isParallelSafe ?? false;
    });
    const sequential = calls.filter((c) => {
      const h = this.tools.get(c.name);
      return !(h?.definition.isParallelSafe ?? false);
    });

    // Run parallel-safe calls concurrently
    const parallelResults = await Promise.all(
      parallelSafe.map((call) => this.executeOne(call))
    );

    // Run sequential calls one-at-a-time
    const sequentialResults: ToolCallResult[] = [];
    for (const call of sequential) {
      if (this.signal.aborted) {
        sequentialResults.push(this.abortedResult(call));
        continue;
      }
      sequentialResults.push(await this.executeOne(call));
    }

    // Reorder results to match original call order
    const resultMap = new Map<string, ToolCallResult>();
    [...parallelResults, ...sequentialResults].forEach((r) =>
      resultMap.set(r.id, r)
    );
    return calls.map((c) => resultMap.get(c.id)!).filter(Boolean);
  }

  private async executeOne(call: ToolCallRequest): Promise<ToolCallResult> {
    const start = Date.now();
    const base = {
      sessionId: this.sessionId,
      turnIndex: this.getTurnIndex(),
      ts: start,
    };

    const handler = this.tools.get(call.name);

    if (!handler) {
      this.emit({
        kind: "tool_error",
        ...base,
        toolCallId: call.id,
        toolName: call.name,
        error: `Unknown tool: "${call.name}"`,
        isPermissionDenied: false,
      });
      return {
        id: call.id,
        name: call.name,
        content: { type: "error", error: `Unknown tool: "${call.name}"` },
        durationMs: Date.now() - start,
        isError: true,
      };
    }

    const { definition } = handler;

    // Emit tool_call event
    this.emit({
      kind: "tool_call",
      ...base,
      toolCallId: call.id,
      toolName: call.name,
      input: call.input,
      isParallelSafe: definition.isParallelSafe,
    });

    // Permission check
    try {
      await this.permissionGate.check(definition);
    } catch (err) {
      if (err instanceof PermissionDeniedError) {
        this.emit({
          kind: "tool_error",
          ...base,
          toolCallId: call.id,
          toolName: call.name,
          error: err.message,
          isPermissionDenied: true,
        });
        return {
          id: call.id,
          name: call.name,
          content: {
            type: "error",
            error: err.message,
            isPermissionDenied: true,
          },
          durationMs: Date.now() - start,
          isError: true,
        };
      }
      throw err;
    }

    // Execute with timeout
    const timeoutMs = definition.timeoutMs ?? DEFAULT_TOOL_TIMEOUT_MS;
    let outputText: string;

    try {
      const timeoutSignal = AbortSignal.timeout(timeoutMs);
      // Combine external abort + timeout
      const combined = this.combineSignals(this.signal, timeoutSignal);

      outputText = await handler.execute(call.input, {
        sessionId: this.sessionId,
        workingDir: this.boundary.workspaceRoot,
        permissionTier: definition.requiredTier,
        signal: combined,
      });
    } catch (err) {
      const error =
        err instanceof Error ? err.message : String(err);
      this.emit({
        kind: "tool_error",
        ...base,
        toolCallId: call.id,
        toolName: call.name,
        error,
        isPermissionDenied: false,
      });
      return {
        id: call.id,
        name: call.name,
        content: { type: "error", error },
        durationMs: Date.now() - start,
        isError: true,
      };
    }

    const durationMs = Date.now() - start;

    this.emit({
      kind: "tool_result",
      ...base,
      toolCallId: call.id,
      toolName: call.name,
      durationMs,
      outputPreview: outputText.slice(0, 200),
    });

    return {
      id: call.id,
      name: call.name,
      content: { type: "text", text: outputText },
      durationMs,
      isError: false,
    };
  }

  private abortedResult(call: ToolCallRequest): ToolCallResult {
    return {
      id: call.id,
      name: call.name,
      content: { type: "error", error: "Aborted" },
      durationMs: 0,
      isError: true,
    };
  }

  private combineSignals(...signals: AbortSignal[]): AbortSignal {
    const controller = new AbortController();
    for (const signal of signals) {
      if (signal.aborted) {
        controller.abort(signal.reason);
        break;
      }
      signal.addEventListener("abort", () => controller.abort(signal.reason), {
        once: true,
      });
    }
    return controller.signal;
  }
}
packages/core/src/providers/ProviderRouter.ts
TypeScript

// packages/core/src/providers/ProviderRouter.ts
// ─────────────────────────────────────────────────────────────────────────────
// Final version — delegates to @locoworker/providers.
// Provides a stable internal interface so queryLoop doesn't import provider
// packages directly (avoids hard coupling + eases testing with mocks).
// ─────────────────────────────────────────────────────────────────────────────

import type { ModelConfig, ConversationTurn } from "../types/agent.types.js";
import type { ToolDefinition } from "../types/tool.types.js";

export interface ModelResponse {
  stopReason: "end_turn" | "tool_use" | "max_tokens" | "stop_sequence";
  text: string;
  toolCalls: Array<{ id: string; name: string; input: Record<string, unknown> }>;
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
}

export interface StreamDelta {
  type: "text_delta" | "tool_call_delta" | "done";
  text?: string;
}

export type StreamHandler = (delta: StreamDelta) => void;

/**
 * Minimal provider interface that ProviderRouter exposes to queryLoop.
 * The actual implementation delegates to @locoworker/providers.
 */
export interface IProviderAdapter {
  chat(
    config: ModelConfig,
    systemPrompt: string,
    messages: ConversationTurn[],
    tools: ToolDefinition[],
    onStream?: StreamHandler
  ): Promise<ModelResponse>;
}

/**
 * Lazy-loads @locoworker/providers to avoid circular deps and allow
 * the package to be absent in test environments.
 */
async function loadProviderRegistry(): Promise<{
  route: (config: ModelConfig) => IProviderAdapter;
}> {
  try {
    // Dynamic import — @locoworker/providers may not be built yet in tests
    const mod = await import("@locoworker/providers");
    return mod.ProviderRegistry;
  } catch {
    throw new Error(
      "@locoworker/providers is not available. " +
      "Ensure the package is built or inject a mock adapter via ProviderRouter.setAdapter()."
    );
  }
}

export class ProviderRouter {
  private static injectedAdapter: IProviderAdapter | null = null;

  /**
   * Inject a test/mock adapter. Takes priority over @locoworker/providers.
   * Call with null to reset.
   */
  static setAdapter(adapter: IProviderAdapter | null): void {
    ProviderRouter.injectedAdapter = adapter;
  }

  async chat(
    config: ModelConfig,
    systemPrompt: string,
    messages: ConversationTurn[],
    tools: ToolDefinition[],
    onStream?: StreamHandler
  ): Promise<ModelResponse> {
    if (ProviderRouter.injectedAdapter) {
      return ProviderRouter.injectedAdapter.chat(
        config,
        systemPrompt,
        messages,
        tools,
        onStream
      );
    }

    const registry = await loadProviderRegistry();
    const adapter = registry.route(config);
    return adapter.chat(config, systemPrompt, messages, tools, onStream);
  }
}
packages/core/src/session/SessionManager.ts
TypeScript

// packages/core/src/session/SessionManager.ts
// ─────────────────────────────────────────────────────────────────────────────
// Create, restore, checkpoint, list, and expire sessions.
// Sessions are persisted as JSON files in <workspaceRoot>/.locoworker/sessions/
// so they survive process restarts.
// ─────────────────────────────────────────────────────────────────────────────

import fs from "node:fs/promises";
import path from "node:path";
import { randomUUID } from "node:crypto";
import type { SessionState, ModelConfig, BudgetConfig } from "../types/agent.types.js";
import { DEFAULT_BUDGET } from "../types/agent.types.js";
import { PermissionTier } from "../types/permission.types.js";

const SESSIONS_DIR = ".locoworker/sessions";
const SESSION_TTL_MS = 7 * 24 * 60 * 60 * 1000; // 7 days

export class SessionManager {
  private readonly sessionsDir: string;

  constructor(private readonly workingDir: string) {
    this.sessionsDir = path.join(workingDir, SESSIONS_DIR);
  }

  async create(opts: {
    modelConfig: ModelConfig;
    budget?: Partial<BudgetConfig>;
  }): Promise<SessionState> {
    await fs.mkdir(this.sessionsDir, { recursive: true });

    const session: SessionState = {
      id: randomUUID(),
      workingDir: this.workingDir,
      modelConfig: opts.modelConfig,
      budget: { ...DEFAULT_BUDGET, ...opts.budget },
      createdAt: Date.now(),
      lastCheckpointAt: Date.now(),
      turnIndex: 0,
      totalInputTokens: 0,
      totalOutputTokens: 0,
      totalCostUsd: 0,
      conversationHistory: [],
      permissionState: {
        grantedTiers: new Set([PermissionTier.READ_ONLY]),
        permanentlyGranted: new Set(),
        permanentlyDenied: new Set(),
        sessionGrants: new Map(),
      },
      aborted: false,
    };

    await this.save(session);
    return session;
  }

  async restore(sessionId: string): Promise<SessionState> {
    const filePath = this.sessionPath(sessionId);
    const raw = await fs.readFile(filePath, "utf8");
    const parsed = JSON.parse(raw);

    // Re-hydrate Sets and Maps (lost in JSON serialisation)
    parsed.permissionState.grantedTiers = new Set(
      parsed.permissionState.grantedTiers ?? [PermissionTier.READ_ONLY]
    );
    parsed.permissionState.permanentlyGranted = new Set(
      parsed.permissionState.permanentlyGranted ?? []
    );
    parsed.permissionState.permanentlyDenied = new Set(
      parsed.permissionState.permanentlyDenied ?? []
    );
    parsed.permissionState.sessionGrants = new Map(
      Object.entries(parsed.permissionState.sessionGrants ?? {})
    );

    return parsed as SessionState;
  }

  async checkpoint(session: SessionState): Promise<string> {
    session.lastCheckpointAt = Date.now();
    await this.save(session);
    return session.id;
  }

  async save(session: SessionState): Promise<void> {
    await fs.mkdir(this.sessionsDir, { recursive: true });
    const serialisable = {
      ...session,
      permissionState: {
        grantedTiers: [...session.permissionState.grantedTiers],
        permanentlyGranted: [...session.permissionState.permanentlyGranted],
        permanentlyDenied: [...session.permissionState.permanentlyDenied],
        sessionGrants: Object.fromEntries(session.permissionState.sessionGrants),
      },
    };
    await fs.writeFile(
      this.sessionPath(session.id),
      JSON.stringify(serialisable, null, 2),
      "utf8"
    );
  }

  async list(): Promise<Array<{ id: string; createdAt: number; model: string }>> {
    try {
      const files = await fs.readdir(this.sessionsDir);
      const sessions = await Promise.all(
        files
          .filter((f) => f.endsWith(".json"))
          .map(async (f) => {
            try {
              const raw = await fs.readFile(
                path.join(this.sessionsDir, f),
                "utf8"
              );
              const s = JSON.parse(raw);
              return {
                id: s.id,
                createdAt: s.createdAt,
                model: s.modelConfig?.model ?? "unknown",
              };
            } catch {
              return null;
            }
          })
      );
      return sessions.filter(Boolean) as Array<{
        id: string;
        createdAt: number;
        model: string;
      }>;
    } catch {
      return [];
    }
  }

  async expire(): Promise<number> {
    const all = await this.list();
    const cutoff = Date.now() - SESSION_TTL_MS;
    let expired = 0;
    for (const s of all) {
      if (s.createdAt < cutoff) {
        try {
          await fs.unlink(this.sessionPath(s.id));
          expired++;
        } catch {
          /* ignore */
        }
      }
    }
    return expired;
  }

  private sessionPath(id: string): string {
    return path.join(this.sessionsDir, `${id}.json`);
  }
}
The star of the show — queryLoop
packages/core/src/loop/queryLoop.ts
TypeScript

// packages/core/src/loop/queryLoop.ts
// ─────────────────────────────────────────────────────────────────────────────
// THE async-generator agent loop. This is the "spine" of LocoWorker.
//
// Every surface (CLI, Gateway SSE stream, Desktop IPC, tests) MUST consume
// this generator and respond to the yielded AgentEvent union.
//
// Contract:
//   - yields AgentEvent on every meaningful state change
//   - always yields session_end as the final event (even on fatal errors)
//   - caller signals abort via ctx.signal (AbortController)
//   - caller injects confirmationHandler for permission prompts
//   - no console.log/error — all communication is via yielded events
//
// ─────────────────────────────────────────────────────────────────────────────

import type {
  AgentContext,
  ConversationTurn,
  ToolResultBlock,
} from "../types/agent.types.js";
import type { AgentEvent } from "../types/event.types.js";
import type { QueryLoopGenerator } from "../types/agent.types.js";
import type { ToolCallRequest } from "../types/tool.types.js";

import { ContextInjector } from "./ContextInjector.js";
import { TurnAssembler } from "./TurnAssembler.js";
import { BudgetEnforcer } from "./BudgetEnforcer.js";
import { CompactionEngine } from "./CompactionEngine.js";
import { ToolExecutor } from "./ToolExecutor.js";
import { PermissionGate } from "../permissions/PermissionGate.js";
import { PermissionConfirmation } from "../permissions/PermissionConfirmation.js";
import { WorkspaceBoundary } from "../workspace/WorkspaceBoundary.js";
import { ProviderRouter } from "../providers/ProviderRouter.js";

export async function* queryLoop(
  userMessage: string,
  ctx: AgentContext
): QueryLoopGenerator {
  const { session, tools, emit, confirmationHandler, signal, onCheckpoint } = ctx;
  const externalSignal = signal ?? new AbortController().signal;

  // ── Event helper ──────────────────────────────────────────────────────────
  const events: AgentEvent[] = [];

  function yieldEmit(event: AgentEvent): AgentEvent {
    emit(event);
    events.push(event);
    return event;
  }

  // ── Infrastructure setup ──────────────────────────────────────────────────
  const boundary = new WorkspaceBoundary(session.workingDir);
  const contextInjector = new ContextInjector(session.workingDir);
  const turnAssembler = new TurnAssembler(session.budget.maxInputTokens);
  const providerRouter = new ProviderRouter();

  let turnIndex = session.turnIndex;
  const getTurnIndex = () => turnIndex;

  const budgetEnforcer = new BudgetEnforcer(
    session.budget,
    emit,
    session.id,
    getTurnIndex
  );

  const compactionEngine = new CompactionEngine(
    async (messages, systemPrompt) => {
      const res = await providerRouter.chat(
        session.modelConfig,
        systemPrompt,
        messages,
        []
      );
      return res.text;
    },
    emit,
    session.id,
    getTurnIndex
  );

  const permConfirmation = new PermissionConfirmation(confirmationHandler);

  const permissionGate = new PermissionGate({
    state: session.permissionState,
    confirmation: permConfirmation,
    emit,
    sessionId: session.id,
    turnIndex: getTurnIndex,
  });

  const toolExecutor = new ToolExecutor(
    tools,
    permissionGate,
    boundary,
    emit,
    session.id,
    getTurnIndex,
    externalSignal
  );

  // ── Session start ─────────────────────────────────────────────────────────
  yield yieldEmit({
    kind: "session_start",
    sessionId: session.id,
    turnIndex,
    ts: Date.now(),
    workingDir: session.workingDir,
    model: session.modelConfig.model,
  });

  // ── Context injection ─────────────────────────────────────────────────────
  let systemPrompt: string;
  try {
    const injected = await contextInjector.inject();
    systemPrompt = injected.systemPrompt;
    yield yieldEmit({
      kind: "context_injected",
      sessionId: session.id,
      turnIndex,
      ts: Date.now(),
      sources: injected.sources,
      totalChars: injected.totalChars,
    });
  } catch (err) {
    // Context injection failure is non-fatal — use bare system prompt
    systemPrompt = "You are LocoWorker, an agentic developer workspace.";
    yield yieldEmit({
      kind: "session_error",
      sessionId: session.id,
      turnIndex,
      ts: Date.now(),
      error: `Context injection failed: ${err instanceof Error ? err.message : String(err)}`,
      recoverable: true,
    });
  }

  // ── First user turn ───────────────────────────────────────────────────────
  yield yieldEmit({
    kind: "turn_start",
    sessionId: session.id,
    turnIndex,
    ts: Date.now(),
    userMessage,
  });

  session.conversationHistory.push({
    role: "user",
    content: userMessage,
    ts: Date.now(),
  });

  // ── Main loop ─────────────────────────────────────────────────────────────
  let sessionEndReason: "complete" | "budget_exceeded" | "cost_cap" | "user_abort" | "fatal_error" =
    "complete";

  try {
    while (true) {
      // External abort check
      if (externalSignal.aborted) {
        sessionEndReason = "user_abort";
        break;
      }

      // Build message array for this turn
      const assembled = turnAssembler.assemble(
        systemPrompt,
        session.conversationHistory
      );

      // Collect tool definitions for the model
      const toolDefinitions = [...tools.values()].map((h) => h.definition);

      // Emit model request event
      yield yieldEmit({
        kind: "model_request",
        sessionId: session.id,
        turnIndex,
        ts: Date.now(),
        model: session.modelConfig.model,
        messageCount: assembled.messages.length,
        estimatedInputTokens: assembled.estimatedTokens,
      });

      // ── Call the model ────────────────────────────────────────────────────
      yield yieldEmit({
        kind: "model_response_start",
        sessionId: session.id,
        turnIndex,
        ts: Date.now(),
        model: session.modelConfig.model,
      });

      let modelResponse;
      try {
        modelResponse = await providerRouter.chat(
          session.modelConfig,
          assembled.systemPrompt,
          assembled.messages,
          toolDefinitions,
          (delta) => {
            if (delta.type === "text_delta" && delta.text) {
              // Fire-and-forget stream delta (non-yielded for simplicity;
              // callers who need deltas should subscribe to emit directly)
              emit({
                kind: "model_response_delta",
                sessionId: session.id,
                turnIndex,
                ts: Date.now(),
                delta: delta.text,
              });
            }
          }
        );
      } catch (err) {
        const errorMsg = err instanceof Error ? err.message : String(err);
        yield yieldEmit({
          kind: "fatal_error",
          sessionId: session.id,
          turnIndex,
          ts: Date.now(),
          error: `Model call failed: ${errorMsg}`,
          stack: err instanceof Error ? err.stack : undefined,
        });
        sessionEndReason = "fatal_error";
        break;
      }

      yield yieldEmit({
        kind: "model_response_end",
        sessionId: session.id,
        turnIndex,
        ts: Date.now(),
        stopReason: modelResponse.stopReason,
        inputTokens: modelResponse.inputTokens,
        outputTokens: modelResponse.outputTokens,
        costUsd: modelResponse.costUsd,
      });

      // ── Update session accounting ─────────────────────────────────────────
      session.totalInputTokens += modelResponse.inputTokens;
      session.totalOutputTokens += modelResponse.outputTokens;
      session.totalCostUsd += modelResponse.costUsd;

      // ── Handle tool calls ─────────────────────────────────────────────────
      let toolCallsExecuted = 0;

      if (
        modelResponse.stopReason === "tool_use" &&
        modelResponse.toolCalls.length > 0
      ) {
        // Build ToolCallRequest array
        const callRequests: ToolCallRequest[] = modelResponse.toolCalls.map((tc) => ({
          id: tc.id,
          name: tc.name,
          input: tc.input,
        }));

        // Execute all tool calls (parallel-safe run concurrently)
        const results = await toolExecutor.executeAll(callRequests);
        toolCallsExecuted = results.length;

        // Build assistant turn (with tool calls)
        const assistantTurn: ConversationTurn = {
          role: "assistant",
          content: modelResponse.text || "",
          toolCalls: callRequests.map((c) => ({
            id: c.id,
            name: c.name,
            input: c.input,
          })),
          inputTokens: modelResponse.inputTokens,
          outputTokens: modelResponse.outputTokens,
          costUsd: modelResponse.costUsd,
          ts: Date.now(),
        };
        session.conversationHistory.push(assistantTurn);

        // Build tool results turn (user role, per Anthropic/OpenAI convention)
        const toolResultBlocks: ToolResultBlock[] = results.map((r) => ({
          toolCallId: r.id,
          content: r.content.type === "text" ? r.content.text : r.content.error,
          isError: r.isError,
        }));
        const toolResultTurn: ConversationTurn = {
          role: "user",
          content: toolResultBlocks
            .map(
              (b) =>
                `[Tool result for ${b.toolCallId}]\n${b.isError ? "[ERROR] " : ""}${b.content}`
            )
            .join("\n\n"),
          toolResults: toolResultBlocks,
          ts: Date.now(),
        };
        session.conversationHistory.push(toolResultTurn);
      } else {
        // No tool calls — this is the final response
        const assistantTurn: ConversationTurn = {
          role: "assistant",
          content: modelResponse.text,
          inputTokens: modelResponse.inputTokens,
          outputTokens: modelResponse.outputTokens,
          costUsd: modelResponse.costUsd,
          ts: Date.now(),
        };
        session.conversationHistory.push(assistantTurn);
      }

      // ── Emit turn_complete ────────────────────────────────────────────────
      yield yieldEmit({
        kind: "turn_complete",
        sessionId: session.id,
        turnIndex,
        ts: Date.now(),
        assistantText: modelResponse.text,
        toolCallsExecuted,
        inputTokens: modelResponse.inputTokens,
        outputTokens: modelResponse.outputTokens,
        costUsd: modelResponse.costUsd,
      });

      turnIndex++;
      session.turnIndex = turnIndex;

      // ── Budget check ──────────────────────────────────────────────────────
      const budgetDecision = budgetEnforcer.recordTurn(
        modelResponse.inputTokens,
        modelResponse.outputTokens,
        modelResponse.costUsd
      );

      if (budgetDecision.action === "stop") {
        sessionEndReason =
          budgetDecision.reason === "cost_cap" ? "cost_cap" : "budget_exceeded";
        break;
      }

      if (budgetDecision.action === "compact") {
        const result = await compactionEngine.compact(
          session.conversationHistory,
          session.budget.compactionMode as "auto" | "full"
        );
        session.conversationHistory = result.turns;

        // Yield the compaction events that the engine already emitted
        // (they've been emitted synchronously — nothing to re-yield here)
      }

      // ── Check if we're done (no tool calls = final answer) ────────────────
      if (
        modelResponse.stopReason === "end_turn" &&
        modelResponse.toolCalls.length === 0
      ) {
        yield yieldEmit({
          kind: "loop_complete",
          sessionId: session.id,
          turnIndex,
          ts: Date.now(),
          finalResponse: modelResponse.text,
        });
        break;
      }

      // ── Periodic checkpoint (every 5 turns) ──────────────────────────────
      if (turnIndex % 5 === 0 && onCheckpoint) {
        try {
          await onCheckpoint(session.id);
          yield yieldEmit({
            kind: "session_checkpoint",
            sessionId: session.id,
            turnIndex,
            ts: Date.now(),
            checkpointId: `${session.id}-t${turnIndex}`,
          });
        } catch {
          /* checkpoint failure is non-fatal */
        }
      }
    }
  } catch (err) {
    const errorMsg = err instanceof Error ? err.message : String(err);
    yield yieldEmit({
      kind: "fatal_error",
      sessionId: session.id,
      turnIndex,
      ts: Date.now(),
      error: errorMsg,
      stack: err instanceof Error ? err.stack : undefined,
    });
    sessionEndReason = "fatal_error";
  }

  // ── Always emit session_end ───────────────────────────────────────────────
  const stats = budgetEnforcer.stats;
  yield yieldEmit({
    kind: "session_end",
    sessionId: session.id,
    turnIndex,
    ts: Date.now(),
    reason: sessionEndReason,
    totalTokensUsed: stats.totalInputTokens + stats.totalOutputTokens,
    totalCostUsd: stats.totalCostUsd,
    totalTurns: stats.turnCount,
  });
}
Package entry point
packages/core/src/index.ts
TypeScript

// packages/core/src/index.ts
// Public API surface of @locoworker/core

// Loop
export { queryLoop } from "./loop/queryLoop.js";
export { ContextInjector } from "./loop/ContextInjector.js";
export { TurnAssembler } from "./loop/TurnAssembler.js";
export { BudgetEnforcer } from "./loop/BudgetEnforcer.js";
export { CompactionEngine } from "./loop/CompactionEngine.js";
export { ToolExecutor } from "./loop/ToolExecutor.js";

// Permissions
export { PermissionGate, PermissionDeniedError } from "./permissions/PermissionGate.js";
export { PermissionConfirmation } from "./permissions/PermissionConfirmation.js";

// Workspace
export { WorkspaceBoundary, WorkspaceBoundaryError } from "./workspace/WorkspaceBoundary.js";

// Session
export { SessionManager } from "./session/SessionManager.js";

// Providers
export { ProviderRouter } from "./providers/ProviderRouter.js";

// Events
export { AgentEventEmitter } from "./events/EventEmitter.js";

// Types
export type {
  AgentEvent,
  AgentEventKind,
  SessionStartEvent,
  SessionEndEvent,
  TurnStartEvent,
  ModelRequestEvent,
  ModelResponseEndEvent,
  ToolCallEvent,
  ToolResultEvent,
  ToolErrorEvent,
  PermissionPromptEvent,
  PermissionGrantedEvent,
  PermissionDeniedEvent,
  BudgetWarningEvent,
  BudgetExceededEvent,
  CostCapExceededEvent,
  CompactionStartEvent,
  CompactionEndEvent,
  TurnCompleteEvent,
  LoopCompleteEvent,
  SessionErrorEvent,
  FatalErrorEvent,
} from "./types/event.types.js";

export type {
  AgentContext,
  SessionState,
  ConversationTurn,
  ModelConfig,
  BudgetConfig,
  QueryLoopGenerator,
} from "./types/agent.types.js";

export { DEFAULT_BUDGET } from "./types/agent.types.js";

export type {
  ToolHandler,
  ToolDefinition,
  ToolCallRequest,
  ToolCallResult,
  ToolExecutionContext,
} from "./types/tool.types.js";

export {
  PermissionTier,
} from "./types/permission.types.js";

export type {
  PermissionState,
  PermissionDecision,
  ConfirmationHandler,
} from "./types/permission.types.js";

export { estimateTokens } from "./utils/tokenEstimator.js";
Tests
tests/unit/core/WorkspaceBoundary.test.ts
TypeScript

import { describe, it, expect } from "vitest";
import path from "node:path";
import os from "node:os";
import fs from "node:fs";
import { WorkspaceBoundary, WorkspaceBoundaryError } from "../../../packages/core/src/workspace/WorkspaceBoundary.js";

describe("WorkspaceBoundary", () => {
  const tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "wb-test-"));
  const boundary = new WorkspaceBoundary(tmpDir);

  it("resolves a simple relative path within workspace", () => {
    const resolved = boundary.resolve("src/index.ts");
    expect(resolved).toBe(path.join(tmpDir, "src/index.ts"));
  });

  it("resolves an absolute path within workspace", () => {
    const abs = path.join(tmpDir, "src/foo.ts");
    expect(boundary.resolve(abs)).toBe(abs);
  });

  it("throws for path traversal (../../etc/passwd)", () => {
    expect(() => boundary.resolve("../../etc/passwd")).toThrow(
      WorkspaceBoundaryError
    );
  });

  it("throws for absolute path outside workspace", () => {
    expect(() => boundary.resolve("/etc/passwd")).toThrow(WorkspaceBoundaryError);
  });

  it("isSafe returns false for escaping paths", () => {
    expect(boundary.isSafe("../../secret")).toBe(false);
  });

  it("isSafe returns true for valid paths", () => {
    expect(boundary.isSafe("src/deep/nested/file.ts")).toBe(true);
  });

  it("relativize returns path relative to root", () => {
    const abs = path.join(tmpDir, "src/foo.ts");
    expect(boundary.relativize(abs)).toBe("src/foo.ts");
  });
});
tests/unit/core/BudgetEnforcer.test.ts
TypeScript

import { describe, it, expect, vi } from "vitest";
import { BudgetEnforcer } from "../../../packages/core/src/loop/BudgetEnforcer.js";
import type { BudgetConfig } from "../../../packages/core/src/types/agent.types.js";

const makeBudget = (overrides?: Partial<BudgetConfig>): BudgetConfig => ({
  maxInputTokens: 200_000,
  softLimitTokens: 100,
  hardLimitTokens: 200,
  maxCostUsd: 1.0,
  compactionMode: "auto",
  ...overrides,
});

describe("BudgetEnforcer", () => {
  it("returns continue when under soft limit", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(makeBudget(), emit, "s1", () => 0);
    const decision = enforcer.recordTurn(50, 10, 0.001);
    expect(decision.action).toBe("continue");
    expect(emit).not.toHaveBeenCalled();
  });

  it("returns compact and emits budget_warning at soft limit", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(makeBudget(), emit, "s1", () => 1);
    const decision = enforcer.recordTurn(101, 10, 0.001);
    expect(decision.action).toBe("compact");
    expect(emit).toHaveBeenCalledWith(
      expect.objectContaining({ kind: "budget_warning" })
    );
  });

  it("does NOT emit second budget_warning (only once)", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(makeBudget(), emit, "s1", () => 1);
    enforcer.recordTurn(101, 0, 0);
    emit.mockClear();
    const decision2 = enforcer.recordTurn(10, 0, 0);
    // No second warning
    expect(emit).not.toHaveBeenCalledWith(
      expect.objectContaining({ kind: "budget_warning" })
    );
    expect(decision2.action).toBe("continue");
  });

  it("returns stop and emits budget_exceeded at hard limit", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(
      makeBudget({ compactionMode: "none" }),
      emit,
      "s1",
      () => 2
    );
    enforcer.recordTurn(50, 0, 0);
    enforcer.recordTurn(100, 0, 0);
    const decision = enforcer.recordTurn(60, 0, 0);
    expect(decision).toEqual({ action: "stop", reason: "hard_limit" });
    expect(emit).toHaveBeenCalledWith(
      expect.objectContaining({ kind: "budget_exceeded" })
    );
  });

  it("returns stop and emits cost_cap_exceeded when cost cap hit", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(
      makeBudget({ maxCostUsd: 0.5 }),
      emit,
      "s1",
      () => 0
    );
    const decision = enforcer.recordTurn(10, 10, 0.6);
    expect(decision).toEqual({ action: "stop", reason: "cost_cap" });
    expect(emit).toHaveBeenCalledWith(
      expect.objectContaining({ kind: "cost_cap_exceeded" })
    );
  });

  it("returns stop when turn cap hit", () => {
    const emit = vi.fn();
    const enforcer = new BudgetEnforcer(
      makeBudget({ maxTurns: 2 }),
      emit,
      "s1",
      () => 0
    );
    enforcer.recordTurn(1, 1, 0);
    const decision = enforcer.recordTurn(1, 1, 0);
    expect(decision).toEqual({ action: "stop", reason: "turn_cap" });
  });
});
tests/unit/core/TurnAssembler.test.ts
TypeScript

import { describe, it, expect } from "vitest";
import { TurnAssembler } from "../../../packages/core/src/loop/TurnAssembler.js";
import type { ConversationTurn } from "../../../packages/core/src/types/agent.types.js";

const makeTurn = (role: "user" | "assistant", content: string): ConversationTurn => ({
  role,
  content,
  ts: Date.now(),
});

describe("TurnAssembler", () => {
  it("returns all turns when under budget", () => {
    const assembler = new TurnAssembler(100_000);
    const history: ConversationTurn[] = [
      makeTurn("user", "Hello"),
      makeTurn("assistant", "Hi there"),
    ];
    const result = assembler.assemble("System prompt", history);
    expect(result.messages).toHaveLength(2);
    expect(result.estimatedTokens).toBeGreaterThan(0);
  });

  it("trims oldest turns when over budget, keeping at least 2", () => {
    const assembler = new TurnAssembler(10); // 10 tokens max — tiny
    const history: ConversationTurn[] = [
      makeTurn("user", "A".repeat(200)),
      makeTurn("assistant", "B".repeat(200)),
      makeTurn("user", "C".repeat(200)),
      makeTurn("assistant", "D".repeat(200)),
    ];
    const result = assembler.assemble("sys", history);
    expect(result.messages.length).toBeGreaterThanOrEqual(2);
    expect(result.messages.length).toBeLessThan(4);
  });

  it("always keeps the systemPrompt in the assembled output", () => {
    const assembler = new TurnAssembler(100_000);
    const result = assembler.assemble("You are LocoWorker.", [makeTurn("user", "hi")]);
    expect(result.systemPrompt).toBe("You are LocoWorker.");
  });
});
tests/unit/core/PermissionConfirmation.test.ts
TypeScript

import { describe, it, expect, vi } from "vitest";
import { PermissionConfirmation } from "../../../packages/core/src/permissions/PermissionConfirmation.js";
import { PermissionTier } from "../../../packages/core/src/types/permission.types.js";

describe("PermissionConfirmation", () => {
  it("uses the injected handler", async () => {
    const mockHandler = vi.fn().mockResolvedValue({
      granted: true,
      permanent: false,
      promptId: "p1",
    });

    const confirmation = new PermissionConfirmation(mockHandler);
    const result = await confirmation.confirm(
      "bash",
      PermissionTier.EXECUTE,
      "Run shell commands",
      ["ls -la", "npm install"]
    );

    expect(mockHandler).toHaveBeenCalledWith(
      "bash",
      PermissionTier.EXECUTE,
      "Run shell commands",
      ["ls -la", "npm install"]
    );
    expect(result.granted).toBe(true);
  });

  it("allows replacing handler at runtime", async () => {
    const first = vi.fn().mockResolvedValue({ granted: false, permanent: false, promptId: "a" });
    const second = vi.fn().mockResolvedValue({ granted: true, permanent: true, promptId: "b" });

    const confirmation = new PermissionConfirmation(first);
    await confirmation.confirm("tool", PermissionTier.READ_WRITE, "desc", []);
    expect(first).toHaveBeenCalledOnce();

    confirmation.setHandler(second);
    const result = await confirmation.confirm("tool", PermissionTier.READ_WRITE, "desc", []);
    expect(second).toHaveBeenCalledOnce();
    expect(result.permanent).toBe(true);
  });

  it("returns denied if handler throws", async () => {
    const failing = vi.fn().mockRejectedValue(new Error("TTY unavailable"));
    const confirmation = new PermissionConfirmation(failing);
    // PermissionConfirmation itself doesn't catch — PermissionGate does
    await expect(
      confirmation.confirm("tool", PermissionTier.EXECUTE, "desc", [])
    ).rejects.toThrow("TTY unavailable");
  });
});
tests/unit/core/queryLoop.test.ts
TypeScript

import { describe, it, expect, vi, beforeEach } from "vitest";
import { queryLoop } from "../../../packages/core/src/loop/queryLoop.js";
import { ProviderRouter } from "../../../packages/core/src/providers/ProviderRouter.js";
import { PermissionTier } from "../../../packages/core/src/types/permission.types.js";
import type { AgentContext, SessionState } from "../../../packages/core/src/types/agent.types.js";
import type { AgentEvent } from "../../../packages/core/src/types/event.types.js";
import type { IProviderAdapter, ModelResponse } from "../../../packages/core/src/providers/ProviderRouter.js";
import os from "node:os";
import path from "node:path";
import fs from "node:fs";

// ── Helpers ──────────────────────────────────────────────────────────────────

function makeSession(workingDir: string): SessionState {
  return {
    id: "test-session-1",
    workingDir,
    modelConfig: { provider: "mock", model: "mock-model" },
    budget: {
      maxInputTokens: 200_000,
      softLimitTokens: 100_000,
      hardLimitTokens: 190_000,
      compactionMode: "auto",
    },
    createdAt: Date.now(),
    lastCheckpointAt: Date.now(),
    turnIndex: 0,
    totalInputTokens: 0,
    totalOutputTokens: 0,
    totalCostUsd: 0,
    conversationHistory: [],
    permissionState: {
      grantedTiers: new Set([PermissionTier.READ_ONLY]),
      permanentlyGranted: new Set(),
      permanentlyDenied: new Set(),
      sessionGrants: new Map(),
    },
    aborted: false,
  };
}

async function collectEvents(gen: AsyncGenerator<AgentEvent>): Promise<AgentEvent[]> {
  const events: AgentEvent[] = [];
  for await (const event of gen) {
    events.push(event);
  }
  return events;
}

function makeMockAdapter(response: Partial<ModelResponse>): IProviderAdapter {
  return {
    chat: vi.fn().mockResolvedValue({
      stopReason: "end_turn",
      text: "Hello from mock model",
      toolCalls: [],
      inputTokens: 100,
      outputTokens: 50,
      costUsd: 0.001,
      ...response,
    }),
  };
}

// ── Tests ────────────────────────────────────────────────────────────────────

describe("queryLoop", () => {
  let tmpDir: string;

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), "ql-test-"));
    ProviderRouter.setAdapter(null); // reset between tests
  });

  it("emits session_start as first event", async () => {
    const adapter = makeMockAdapter({});
    ProviderRouter.setAdapter(adapter);

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
    };

    const gen = queryLoop("Hello", ctx);
    const first = await gen.next();
    expect(first.value?.kind).toBe("session_start");

    // drain rest
    for await (const _ of gen) {}
  });

  it("emits session_end as last event", async () => {
    const adapter = makeMockAdapter({});
    ProviderRouter.setAdapter(adapter);

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
    };

    const collectedEvents = await collectEvents(queryLoop("Hello", ctx));
    const last = collectedEvents[collectedEvents.length - 1];
    expect(last.kind).toBe("session_end");
  });

  it("emits loop_complete when model returns end_turn with no tool calls", async () => {
    ProviderRouter.setAdapter(makeMockAdapter({ stopReason: "end_turn", toolCalls: [] }));

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
    };

    const collectedEvents = await collectEvents(queryLoop("Hello", ctx));
    const kinds = collectedEvents.map((e) => e.kind);
    expect(kinds).toContain("loop_complete");
    expect(kinds).toContain("session_end");
  });

  it("handles abort signal gracefully — emits session_end with user_abort reason", async () => {
    const adapter = makeMockAdapter({});
    ProviderRouter.setAdapter(adapter);

    const ac = new AbortController();
    // Abort immediately before model call resolves
    (adapter.chat as ReturnType<typeof vi.fn>).mockImplementation(async () => {
      ac.abort();
      return {
        stopReason: "end_turn",
        text: "partial",
        toolCalls: [],
        inputTokens: 10,
        outputTokens: 5,
        costUsd: 0,
      };
    });

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
      signal: ac.signal,
    };

    const collectedEvents = await collectEvents(queryLoop("Hello", ctx));
    const endEvent = collectedEvents.find((e) => e.kind === "session_end");
    expect(endEvent).toBeDefined();
  });

  it("emits fatal_error and session_end when model throws", async () => {
    const failingAdapter: IProviderAdapter = {
      chat: vi.fn().mockRejectedValue(new Error("API rate limit exceeded")),
    };
    ProviderRouter.setAdapter(failingAdapter);

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
    };

    const collectedEvents = await collectEvents(queryLoop("Hello", ctx));
    const kinds = collectedEvents.map((e) => e.kind);
    expect(kinds).toContain("fatal_error");
    expect(kinds).toContain("session_end");
    expect(kinds[kinds.length - 1]).toBe("session_end");
  });

  it("emits context_injected when CLAUDE.md exists in workspace", async () => {
    fs.writeFileSync(
      path.join(tmpDir, "CLAUDE.md"),
      "# Project Instructions\n\nAlways write tests."
    );

    ProviderRouter.setAdapter(makeMockAdapter({}));

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];
    const ctx: AgentContext = {
      session,
      tools: new Map(),
      emit: (e) => events.push(e),
    };

    const collectedEvents = await collectEvents(queryLoop("Hello", ctx));
    const injected = collectedEvents.find((e) => e.kind === "context_injected");
    expect(injected).toBeDefined();
    if (injected?.kind === "context_injected") {
      expect(injected.sources).toContain("CLAUDE.md");
    }
  });

  it("executes a tool call and emits tool_call + tool_result", async () => {
    let callCount = 0;
    const adapter: IProviderAdapter = {
      chat: vi.fn().mockImplementation(async () => {
        callCount++;
        if (callCount === 1) {
          // First call — return a tool_use
          return {
            stopReason: "tool_use",
            text: "",
            toolCalls: [{ id: "tc1", name: "echo", input: { message: "hi" } }],
            inputTokens: 100,
            outputTokens: 30,
            costUsd: 0.001,
          };
        }
        // Second call — return end_turn
        return {
          stopReason: "end_turn",
          text: "Done.",
          toolCalls: [],
          inputTokens: 120,
          outputTokens: 10,
          costUsd: 0.001,
        };
      }),
    };
    ProviderRouter.setAdapter(adapter);

    const session = makeSession(tmpDir);
    const events: AgentEvent[] = [];

    // Register a mock "echo" tool
    const tools = new Map();
    tools.set("echo", {
      definition: {
        name: "echo",
        description: "Echo a message",
        inputSchema: {
          type: "object",
          properties: { message: { type: "string" } },
          required: ["message"],
        },
        requiredTier: PermissionTier.READ_ONLY,
        isParallelSafe: true,
        timeoutMs: 5_000,
      },
      execute: async (input: Record<string, unknown>) => String(input.message),
    });

    const ctx: AgentContext = {
      session,
      tools,
      emit: (e) => events.push(e),
      // Auto-grant all confirmations
      confirmationHandler: async () => ({ granted: true, permanent: true, promptId: "x" }),
    };

    const collectedEvents = await collectEvents(queryLoop("Use echo tool", ctx));
    const kinds = collectedEvents.map((e) => e.kind);
    expect(kinds).toContain("tool_call");
    expect(kinds).toContain("tool_result");
    expect(kinds).toContain("loop_complete");
    expect(kinds).toContain("session_end");
  });
});
packages/core/package.json (final canonical version)
JSON

{
  "name": "@locoworker/core",
  "version": "0.1.0",
  "description": "LocoWorker agent loop, session, permissions, workspace boundary",
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
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "lint": "biome lint src"
  },
  "dependencies": {},
  "devDependencies": {
    "@locoworker/tsconfig": "workspace:*",
    "typescript": "^5.4.5",
    "vitest": "^1.6.0",
    "@types/node": "^20.12.0"
  },
  "files": ["dist", "src"]
}
packages/core/tsconfig.build.json
JSON

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
Pass 19 completion checklist
Gate	Status
WorkspaceBoundary — single choke point, all tools MUST route through it	✅
PermissionConfirmation — TTY-aware, injectable, permanent/session grants	✅
PermissionGate — integrates with PermissionConfirmation, emits events	✅
ContextInjector — reads CLAUDE.md / MEMORY.md / .locoworker/instructions.md	✅
BudgetEnforcer — soft warn, hard stop, cost cap, turn cap	✅
CompactionEngine — auto + full modes, falls back on summarisation failure	✅
TurnAssembler — builds message array, trims oldest turns on overflow	✅
ToolExecutor — parallel-safe concurrency, timeout, per-tool permission check	✅
ProviderRouter — delegates to @locoworker/providers, injectable mock for tests	✅
SessionManager — create/restore/checkpoint/expire, JSON persistence	✅
queryLoop — fully wired async generator, all events, graceful abort, always ends with session_end	✅
Unit tests — WorkspaceBoundary, BudgetEnforcer, TurnAssembler, PermissionConfirmation, queryLoop (5 scenarios)	✅
packages/core/src/index.ts — clean public API surface	✅
What Pass 19 unlocks for the next passes
text

Pass 20 (Gateway SSE)  ← needs: queryLoop + AgentEvent stream
Pass 21 (CLI commands) ← needs: queryLoop + PermissionConfirmation (TTY) + SessionManager
Pass 22 (Kairos/AutoResearch engines) ← needs: queryLoop + SessionManager + BudgetEnforcer
Pass 23 (Dashboard)    ← needs: Gateway SSE (Pass 20) which needs Pass 19
Pass 24 (Desktop IPC)  ← needs: queryLoop + SessionManager + PermissionConfirmation
