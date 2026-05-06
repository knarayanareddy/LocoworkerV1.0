Pass 7 — packages/orchestrator (Part 1 of 2)
What packages/orchestrator is: The multi-agent coordination layer for LocoWorker. A clean-room re-implementation of Claude Code's TeammateTool / Agent Teams primitives (discovered via the sourcemap leak and community reverse-engineering), extended with LocoWorker's established SQLite persistence, EventBus integration, and Kairos task queue connectivity. It enables a supervisor agent to spawn typed sub-agents, route messages between them, track shared dependency-aware task lists, enforce file-scope locking to prevent conflicts, and aggregate results — all without external process management (pure async TypeScript, single Node.js process).

Pass 7 split plan
Part	Files Generated
Part 1 (this pass)	package.json · tsconfig.json · src/types/orchestrator.types.ts · src/OrchestratorStore.ts · src/AgentRegistry.ts · src/AgentSpawner.ts · src/MessageRouter.ts
Part 2 (next pass)	src/DelegationPlanner.ts · src/FileLockManager.ts · src/ResultAggregator.ts · src/OrchestratorEngine.ts · src/OrchestratorReporter.ts · src/index.ts · All 4 test files
Full package file tree
text

packages/orchestrator/
├── package.json
├── tsconfig.json
└── src/
    ├── types/
    │   └── orchestrator.types.ts    ← Full type ABI: teams, agents, messages, tasks, delegation
    ├── OrchestratorStore.ts          ← SQLite persistence (teams, members, messages, shared tasks, file locks)
    ├── AgentRegistry.ts              ← Capability-based agent role registry + sub-agent definitions
    ├── AgentSpawner.ts               ← Spawns sub-agent contexts from packages/core SessionManager
    ├── MessageRouter.ts              ← Peer-to-peer message passing + broadcast (TeammateTool.write/broadcast)
    ├── DelegationPlanner.ts          ← Decomposes a task into a typed delegation plan (Part 2)
    ├── FileLockManager.ts            ← File-scope locking to prevent agent conflicts (Part 2)
    ├── ResultAggregator.ts           ← Collects + merges sub-agent outputs (Part 2)
    ├── OrchestratorEngine.ts         ← Main coordination loop: spawn, route, gate, aggregate (Part 2)
    ├── OrchestratorReporter.ts       ← ORCHESTRATOR_REPORT.md generator (Part 2)
    └── index.ts                      ← Barrel export (Part 2)
    tests/
    ├── OrchestratorStore.test.ts     (Part 2)
    ├── AgentRegistry.test.ts         (Part 2)
    ├── AgentSpawner.test.ts          (Part 2)
    └── MessageRouter.test.ts         (Part 2)
packages/orchestrator/package.json
JSON

{
  "name": "@locoworker/orchestrator",
  "version": "1.0.0",
  "description": "Multi-agent coordination layer for LocoWorker: supervisor/subagent spawning, peer-to-peer messaging, shared task lists, file-scope locking, and result aggregation",
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
    "@locoworker/kairos": "workspace:*",
    "@locoworker/memory": "workspace:*",
    "@locoworker/wiki": "workspace:*"
  },
  "peerDependenciesMeta": {
    "@locoworker/kairos":  { "optional": true },
    "@locoworker/memory":  { "optional": true },
    "@locoworker/wiki":    { "optional": true }
  }
}
packages/orchestrator/tsconfig.json
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
      "@locoworker/core":         ["../core/src/index.ts"],
      "@locoworker/shared":       ["../shared/src/index.ts"],
      "@locoworker/kairos":       ["../kairos/src/index.ts"],
      "@locoworker/memory":       ["../memory/src/index.ts"],
      "@locoworker/wiki":         ["../wiki/src/index.ts"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "src/tests"]
}
src/types/orchestrator.types.ts
TypeScript

/**
 * @locoworker/orchestrator — type definitions
 *
 * This package is a clean-room re-implementation of Claude Code's
 * Agent Teams / TeammateTool multi-agent coordination primitives,
 * discovered via:
 *  - The March 2026 sourcemap leak (spawnTeam, discoverTeams,
 *    requestJoin, approveJoin, TeammateTool.write, TeammateTool.broadcast)
 *  - Community reverse-engineering (kieranklaassen gist, claudefa.st docs)
 *  - Anthropic's official Agent Teams documentation (May 2026)
 *
 * LocoWorker extensions beyond the original primitives:
 *  - SQLite-persisted team state (survives process restarts)
 *  - Capability-based agent role registry (typed sub-agent definitions)
 *  - Dependency-aware shared task list (blockedBy, parallelSafe)
 *  - File-scope locking (prevents concurrent agent writes to the same file)
 *  - Result aggregation with conflict detection
 *  - Full EventBus integration (same decoupled pattern as WikiSyncAgent, KairosAgent)
 *  - Quality gate hooks (plan approval, output verification)
 *
 * Architecture summary:
 *   OrchestratorEngine  — drives the coordination loop
 *   OrchestratorStore   — SQLite persistence (single writer)
 *   AgentRegistry       — role/capability catalogue
 *   AgentSpawner        — creates sub-agent sessions via packages/core
 *   MessageRouter       — peer-to-peer + broadcast messaging
 *   DelegationPlanner   — decomposes a goal into a typed plan
 *   FileLockManager     — file-scope locking
 *   ResultAggregator    — collects and merges sub-agent outputs
 *
 * DB schema overview:
 *   orch_teams          — team records (one team per orchestration run)
 *   orch_members        — team member records (one per spawned sub-agent)
 *   orch_messages       — append-only peer-to-peer + broadcast message log
 *   orch_shared_tasks   — shared task list with dependency tracking
 *   orch_file_locks     — file-scope locks (acquired/released per task)
 *   orch_results        — sub-agent output records (append-only)
 *   orch_meta           — key-value store (schema version, etc.)
 *
 * Naming conventions consistent with monorepo:
 *   - snake_case columns, tags_csv, session_id, created_at/updated_at
 *   - ISO 8601 UTC timestamps throughout
 *   - UUIDs for all primary keys
 *   - Append-only for messages, results (no UPDATE/DELETE on those tables)
 */

import { z } from 'zod';

// ─── Team enumerations ────────────────────────────────────────────────────────

/**
 * Lifecycle state of an orchestration team.
 * Mirrors the team lifecycle implied by Claude Code's TeammateTool:
 *   planning → spawning → running → gating → completing → done | failed | aborted
 */
export type TeamStatus =
  | 'planning'    // Team being planned (DelegationPlanner running)
  | 'spawning'    // Sub-agents being spawned (AgentSpawner running)
  | 'running'     // All agents spawned, work in progress
  | 'gating'      // Waiting for quality gate approval
  | 'completing'  // Aggregating results, cleanup in progress
  | 'done'        // All tasks complete, results aggregated
  | 'failed'      // One or more critical failures
  | 'aborted';    // Explicitly cancelled by human or supervisor

/**
 * The execution topology for this team.
 * Determines how tasks are distributed and dependencies resolved.
 */
export type TeamTopology =
  | 'sequential'   // Members run one after another (pipeline)
  | 'parallel'     // All members run concurrently (swarm)
  | 'hierarchical' // Supervisor → feature leads → specialists (tree)
  | 'debate'       // Members run in parallel then synthesise (devil's advocate)
  | 'wave';        // Multiple sequential waves of parallel agents

/**
 * Lifecycle state of a team member (sub-agent).
 * Mirrors approveShutdown / requestShutdown from the Claude Code TeammateTool.
 */
export type MemberStatus =
  | 'spawning'      // Being created by AgentSpawner
  | 'idle'          // Spawned, waiting for first task
  | 'working'       // Actively executing a task
  | 'waiting'       // Waiting on a dependency (blocked)
  | 'reviewing'     // In review/gate step
  | 'shutdown_req'  // requestShutdown sent, waiting for approve
  | 'done'          // Completed all assigned tasks
  | 'failed'        // Fatal error, will not recover
  | 'evicted';      // Forcibly removed by supervisor

/**
 * Predefined agent roles with associated capability sets.
 * Each role maps to a sub-agent definition (prompt + tool allowlist).
 */
export type AgentRole =
  | 'supervisor'        // Team lead — plans, delegates, synthesises
  | 'architect'         // System design, API contracts, ADRs
  | 'implementer'       // Feature implementation
  | 'test_runner'       // Test writing and execution
  | 'security_reviewer' // Security audit (SAST, dependency scan, auth review)
  | 'doc_writer'        // Documentation, CHANGELOG, wiki pages
  | 'refactor'          // Code cleanup, technical debt reduction
  | 'migration'         // DB/schema/data migrations
  | 'devops'            // CI/CD, Dockerfile, k8s manifests
  | 'researcher'        // Exploration, spike, devil's advocate
  | 'reviewer'          // Code review, PR feedback
  | 'custom';           // User-defined role (uses customPrompt field)

/**
 * Tool permission profile for a sub-agent.
 * Subset of packages/core PermissionTier — orchestrator enforces
 * role-appropriate limits at spawn time.
 */
export type AgentPermissionProfile =
  | 'read_only'      // Can only read files and run queries
  | 'write_local'    // Can write files within its assigned scope
  | 'standard'       // Default: read + write + shell (no dangerous)
  | 'full'           // All permissions (only for supervisor)
  | 'custom';        // Caller specifies explicit permission set

/**
 * Shared task status — mirrors Claude Code's shared task list states.
 * From the Agent Teams docs: pending, in_progress, completed, blocked.
 * Extended with: failed, skipped, gating.
 */
export type SharedTaskStatus =
  | 'pending'      // Waiting to be picked up
  | 'in_progress'  // Assigned to a member
  | 'completed'    // Done and verified
  | 'blocked'      // Waiting on blockedBy task(s)
  | 'failed'       // Error — may retry
  | 'skipped'      // Deliberately skipped (dependency failed)
  | 'gating';      // Waiting for quality gate approval

/**
 * Quality gate type — determines what check is needed before proceeding.
 */
export type GateType =
  | 'plan_approval'    // Human must approve the delegation plan
  | 'output_review'    // Human reviews member output before merge
  | 'test_pass'        // All tests must pass
  | 'security_clear'   // Security reviewer must approve
  | 'auto';            // No human needed — auto-approved by engine

/**
 * Message channel in the team communication system.
 * Mirrors TeammateTool write (direct) and broadcast (all).
 */
export type MessageChannel =
  | 'direct'      // Point-to-point: from_member_id → to_member_id
  | 'broadcast'   // All members receive
  | 'supervisor'  // Routed directly to the supervisor member
  | 'system';     // Engine-generated messages (not from an agent)

/**
 * File lock state.
 */
export type FileLockStatus =
  | 'held'     // Currently locked by a member
  | 'released' // Lock released (normal completion)
  | 'expired'; // Lock expired (member timed out)

// ─── Core domain models ───────────────────────────────────────────────────────

/**
 * An orchestration team — the top-level coordination unit.
 * One team per orchestration run. Persists across process restarts.
 */
export interface OrchestratorTeam {
  /** Stable UUID for this team. */
  id: string;

  /**
   * Human-readable name for this team.
   * e.g. "auth-feature-2026-05-07" or "deploy-2026-05-07"
   * Mirrors the team name used in Claude Code's spawnTeam primitive.
   */
  name: string;

  /** Natural-language goal that prompted this team's creation. */
  goal: string;

  /** Current lifecycle status. */
  status: TeamStatus;

  /** Execution topology. */
  topology: TeamTopology;

  /** ID of the session that created this team (supervisor's session). */
  supervisorSessionId: string;

  /**
   * ID of the supervisor member (team lead).
   * Set after first member is spawned.
   */
  supervisorMemberId?: string;

  /** Max concurrent members allowed. Default: 8. */
  maxConcurrentMembers: number;

  /** Max total members (across all waves). Default: 20. */
  maxTotalMembers: number;

  /**
   * Quality gates to enforce during execution.
   * Applied in order at the appropriate lifecycle points.
   */
  gates: QualityGate[];

  /** Tags for filtering / reporting. */
  tags: string[];

  /** Arbitrary metadata (external IDs, PR URLs, etc.). */
  metadata: Record<string, unknown>;

  /** ISO 8601 creation timestamp. */
  createdAt: string;

  /** ISO 8601 last-updated timestamp. */
  updatedAt: string;

  /** ISO 8601 completion timestamp (set when status → done/failed/aborted). */
  completedAt?: string;

  /** Overall error message if status === 'failed'. */
  errorMessage?: string;

  /** Estimated token cost across all members (updated as sessions run). */
  totalTokensUsed: number;

  /** Estimated total cost in USD across all members. */
  totalCostUsd: number;
}

/**
 * A team member — a single spawned sub-agent within the team.
 * Each member has its own session (context window), assigned scope,
 * and communication channels.
 */
export interface TeamMember {
  /** Stable UUID for this member. */
  id: string;

  /** The team this member belongs to. */
  teamId: string;

  /**
   * Member name within the team (unique per team).
   * e.g. "security-reviewer", "backend-dev", "test-runner-1"
   * Used in message addressing (TeammateTool.write target).
   */
  name: string;

  /** Agent role determining capabilities and default prompt. */
  role: AgentRole;

  /** Current lifecycle status. */
  status: MemberStatus;

  /** The core session ID for this member's queryLoop. */
  sessionId?: string;

  /** Permission profile applied at spawn time. */
  permissionProfile: AgentPermissionProfile;

  /**
   * File scope for this member — paths/globs this member is
   * authorised to write to. FileLockManager enforces this.
   * e.g. ["src/auth/**", "tests/auth/**"]
   */
  fileScope: string[];

  /** Custom system prompt suffix (for 'custom' role or role overrides). */
  customPrompt?: string;

  /**
   * The model to use for this member.
   * Allows mixing e.g. opus for architect, haiku for test_runner.
   */
  modelId?: string;

  /** Wave number this member belongs to (for wave topology). */
  wave: number;

  /** IDs of members that must complete before this member starts. */
  dependsOn: string[];

  /** Shared task IDs assigned to this member. */
  assignedTaskIds: string[];

  /** Token and cost tracking. */
  tokensUsed: number;
  costUsd: number;

  /** ISO 8601 timestamps. */
  createdAt: string;
  updatedAt: string;
  startedAt?: string;
  completedAt?: string;

  /** Error message if status === 'failed'. */
  errorMessage?: string;
}

/**
 * A shared task in the team's shared task list.
 * Mirrors the task list described in Claude Code Agent Teams docs:
 * ~/.claude/tasks/{team-name}/
 *
 * In LocoWorker: persisted in SQLite (orch_shared_tasks).
 */
export interface SharedTask {
  /** Stable UUID. */
  id: string;

  /** The team this task belongs to. */
  teamId: string;

  /** Short task title. */
  title: string;

  /** Detailed description (Markdown). */
  description?: string;

  /** Current status. */
  status: SharedTaskStatus;

  /** Member ID currently working on this task (if status === 'in_progress'). */
  assignedMemberId?: string;

  /**
   * IDs of tasks that must be completed before this can start.
   * Enables dependency-aware scheduling.
   */
  blockedBy: string[];

  /**
   * Whether this task can run in parallel with other tasks.
   * Tasks that write to disjoint file scopes are safe to parallelise.
   */
  parallelSafe: boolean;

  /**
   * File paths this task will write to.
   * Used by FileLockManager to prevent conflicts.
   */
  filePaths: string[];

  /** Quality gate required before this task is marked complete. */
  gate?: GateType;

  /** Tags for reporting and filtering. */
  tags: string[];

  /** Ordering hint (lower = earlier in display). */
  order: number;

  /** ISO 8601 timestamps. */
  createdAt: string;
  updatedAt: string;
  completedAt?: string;

  /** Error detail if status === 'failed'. */
  errorMessage?: string;

  /** Number of attempts made. */
  attemptCount: number;
}

/**
 * A message in the team communication system.
 * Append-only — never updated or deleted.
 *
 * Mirrors TeammateTool.write (direct) and TeammateTool.broadcast.
 */
export interface TeamMessage {
  /** Stable UUID. */
  id: string;

  /** The team this message was sent in. */
  teamId: string;

  /** Member ID of the sender (null for system messages). */
  fromMemberId?: string;

  /** Member name of the sender (denormalised for display). */
  fromMemberName?: string;

  /**
   * Target for direct messages — member ID.
   * Null for broadcast and system messages.
   */
  toMemberId?: string;

  /** Target member name (denormalised for display). */
  toMemberName?: string;

  /** Message channel. */
  channel: MessageChannel;

  /** Message content (free text / Markdown). */
  content: string;

  /**
   * Optional structured payload (JSON).
   * Used for machine-readable coordination signals:
   *   { type: 'plan_approval_request', planId: '...' }
   *   { type: 'rollback_required', reason: '...' }
   *   { type: 'task_update', taskId: '...', status: '...' }
   */
  payload?: Record<string, unknown>;

  /** Whether this message has been read by the target member. */
  read: boolean;

  /** ISO 8601 timestamp. */
  sentAt: string;
}

/**
 * A quality gate checkpoint.
 * Gates pause the team at defined lifecycle points until conditions are met.
 */
export interface QualityGate {
  /** Gate type. */
  type: GateType;

  /** Human-readable description shown when gate is triggered. */
  description: string;

  /**
   * The lifecycle point at which this gate fires.
   * 'pre_wave' fires before a wave starts.
   * 'post_wave' fires after a wave completes.
   * 'pre_task' fires before a specific task.
   * 'post_team' fires before the team completes.
   */
  trigger: 'pre_wave' | 'post_wave' | 'pre_task' | 'post_team';

  /** Wave number to apply this gate to (for pre_wave/post_wave triggers). */
  wave?: number;

  /** Task ID to apply this gate to (for pre_task trigger). */
  taskId?: string;

  /**
   * Whether a human must explicitly approve (true) or
   * the gate can auto-pass if conditions are met (false).
   * Default: true for plan_approval/output_review, false for test_pass/auto.
   */
  requiresHumanApproval: boolean;

  /**
   * ISO 8601 timestamp when this gate was approved.
   * Null until approved.
   */
  approvedAt?: string;

  /** Who approved this gate (member name or 'human'). */
  approvedBy?: string;
}

/**
 * A sub-agent result record — the output of a team member's work.
 * Append-only (multiple results per member allowed, e.g., intermediate outputs).
 */
export interface AgentResult {
  /** Stable UUID. */
  id: string;

  /** The team this result belongs to. */
  teamId: string;

  /** Member that produced this result. */
  memberId: string;

  /** Member name (denormalised). */
  memberName: string;

  /** Member role (denormalised). */
  memberRole: AgentRole;

  /** Shared task this result corresponds to (if applicable). */
  taskId?: string;

  /**
   * Result content — the output the member produced.
   * May be a summary, a diff description, or structured data.
   */
  content: string;

  /**
   * Files created or modified by this member.
   * Used by ResultAggregator to detect conflicts.
   */
  filesModified: string[];

  /**
   * Whether this result passed its quality gate (if one was applied).
   * Null if no gate was applied.
   */
  gateApproved?: boolean;

  /** ISO 8601 timestamp. */
  producedAt: string;

  /** Token cost of this result. */
  tokensUsed: number;
}

/**
 * A file lock record — tracks which member holds a lock on a file path.
 */
export interface FileLock {
  /** Stable UUID. */
  id: string;

  /** The team this lock belongs to. */
  teamId: string;

  /** Member holding the lock. */
  memberId: string;

  /** Member name (denormalised). */
  memberName: string;

  /** Absolute or relative file path being locked. */
  filePath: string;

  /** Lock status. */
  status: FileLockStatus;

  /** ISO 8601 timestamp when lock was acquired. */
  acquiredAt: string;

  /** ISO 8601 timestamp when lock was released or expired. */
  releasedAt?: string;

  /**
   * Lock expiry — if still held after this time, the engine marks it
   * as expired and allows other members to acquire it.
   * Default: 30 minutes from acquisition.
   */
  expiresAt: string;

  /** Optional task ID this lock was acquired for. */
  taskId?: string;
}

// ─── Sub-agent definition (role manifest) ────────────────────────────────────

/**
 * A reusable sub-agent role definition — the LocoWorker equivalent of
 * Claude Code's subagent definition files (.claude/agents/*.md).
 *
 * Stored in AgentRegistry (in-memory + optional JSON file).
 * Not persisted to SQLite (loaded at startup).
 */
export interface AgentDefinition {
  /** Unique name for this definition. e.g. "security-reviewer" */
  name: string;

  /** The role this definition implements. */
  role: AgentRole;

  /** System prompt suffix appended to the base system prompt. */
  systemPromptSuffix: string;

  /** Tool names explicitly allowed for this role (empty = use permission profile default). */
  allowedTools: string[];

  /** Tool names explicitly denied for this role. */
  deniedTools: string[];

  /** Default permission profile. */
  permissionProfile: AgentPermissionProfile;

  /** Default model for this role (optional — falls back to session default). */
  defaultModelId?: string;

  /**
   * Tags describing capabilities.
   * e.g. ['typescript', 'security', 'owasp']
   * Used by AgentRegistry.findByCapabilities().
   */
  capabilities: string[];

  /** Human-readable description of what this agent does. */
  description: string;

  /**
   * Maximum context budget allocated to this role (tokens).
   * Roles with bounded scope (test_runner) use less context than
   * open-ended roles (architect).
   */
  maxContextTokens?: number;

  /** Whether instances of this role can run in parallel. Default: true. */
  supportsParallel: boolean;
}

// ─── DB row types ─────────────────────────────────────────────────────────────

export interface TeamRow {
  id: string;
  name: string;
  goal: string;
  status: TeamStatus;
  topology: TeamTopology;
  supervisor_session_id: string;
  supervisor_member_id: string | null;
  max_concurrent_members: number;
  max_total_members: number;
  gates_json: string;
  tags_csv: string;
  metadata_json: string;
  created_at: string;
  updated_at: string;
  completed_at: string | null;
  error_message: string | null;
  total_tokens_used: number;
  total_cost_usd: number;
}

export interface MemberRow {
  id: string;
  team_id: string;
  name: string;
  role: AgentRole;
  status: MemberStatus;
  session_id: string | null;
  permission_profile: AgentPermissionProfile;
  file_scope_json: string;
  custom_prompt: string | null;
  model_id: string | null;
  wave: number;
  depends_on_json: string;
  assigned_task_ids_json: string;
  tokens_used: number;
  cost_usd: number;
  created_at: string;
  updated_at: string;
  started_at: string | null;
  completed_at: string | null;
  error_message: string | null;
}

export interface SharedTaskRow {
  id: string;
  team_id: string;
  title: string;
  description: string | null;
  status: SharedTaskStatus;
  assigned_member_id: string | null;
  blocked_by_json: string;
  parallel_safe: 0 | 1;
  file_paths_json: string;
  gate: GateType | null;
  tags_csv: string;
  order_index: number;
  created_at: string;
  updated_at: string;
  completed_at: string | null;
  error_message: string | null;
  attempt_count: number;
}

export interface MessageRow {
  id: string;
  team_id: string;
  from_member_id: string | null;
  from_member_name: string | null;
  to_member_id: string | null;
  to_member_name: string | null;
  channel: MessageChannel;
  content: string;
  payload_json: string | null;
  read: 0 | 1;
  sent_at: string;
}

export interface ResultRow {
  id: string;
  team_id: string;
  member_id: string;
  member_name: string;
  member_role: AgentRole;
  task_id: string | null;
  content: string;
  files_modified_json: string;
  gate_approved: 0 | 1 | null;
  produced_at: string;
  tokens_used: number;
}

export interface FileLockRow {
  id: string;
  team_id: string;
  member_id: string;
  member_name: string;
  file_path: string;
  status: FileLockStatus;
  acquired_at: string;
  released_at: string | null;
  expires_at: string;
  task_id: string | null;
}

// ─── Query & result types ─────────────────────────────────────────────────────

export interface TeamQueryOptions {
  status?: TeamStatus | TeamStatus[];
  topology?: TeamTopology;
  limit?: number;
  offset?: number;
}

export interface MemberQueryOptions {
  teamId: string;
  status?: MemberStatus | MemberStatus[];
  role?: AgentRole;
  wave?: number;
}

export interface SharedTaskQueryOptions {
  teamId: string;
  status?: SharedTaskStatus | SharedTaskStatus[];
  assignedMemberId?: string;
  blockedOnly?: boolean;
  readyOnly?: boolean;    // pending + all blockedBy are completed
  limit?: number;
}

export interface OrchestratorStoreStats {
  totalTeams: number;
  activeTeams: number;
  totalMembers: number;
  activeMembers: number;
  totalMessages: number;
  unreadMessages: number;
  totalSharedTasks: number;
  pendingSharedTasks: number;
  completedSharedTasks: number;
  activeFileLocks: number;
  dbSizeBytes: number;
}

export interface SpawnRequest {
  teamId: string;
  memberName: string;
  role: AgentRole;
  definition?: AgentDefinition;
  permissionProfile?: AgentPermissionProfile;
  fileScope?: string[];
  customPrompt?: string;
  modelId?: string;
  wave?: number;
  dependsOn?: string[];
}

export interface SpawnResult {
  member: TeamMember;
  sessionId: string;
  success: boolean;
  error?: string;
}

export interface DelegationPlan {
  teamId: string;
  topology: TeamTopology;
  waves: DelegationWave[];
  gates: QualityGate[];
  estimatedTokens: number;
  estimatedCostUsd: number;
  reasoning: string;
}

export interface DelegationWave {
  waveNumber: number;
  members: Array<{
    name: string;
    role: AgentRole;
    prompt: string;
    fileScope: string[];
    tasks: Array<{
      title: string;
      description: string;
      filePaths: string[];
      parallelSafe: boolean;
      blockedBy: string[];
    }>;
  }>;
}

// ─── Zod validation schemas ───────────────────────────────────────────────────

export const AgentDefinitionSchema = z.object({
  name: z.string().min(1).max(100).regex(/^[a-z0-9-]+$/, 'Name must be kebab-case'),
  role: z.enum([
    'supervisor','architect','implementer','test_runner','security_reviewer',
    'doc_writer','refactor','migration','devops','researcher','reviewer','custom',
  ]),
  systemPromptSuffix: z.string(),
  allowedTools: z.array(z.string()).default([]),
  deniedTools: z.array(z.string()).default([]),
  permissionProfile: z.enum(['read_only','write_local','standard','full','custom']),
  defaultModelId: z.string().optional(),
  capabilities: z.array(z.string()).default([]),
  description: z.string(),
  maxContextTokens: z.number().int().positive().optional(),
  supportsParallel: z.boolean().default(true),
});

export const OrchestratorTeamSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(200).regex(/^[a-z0-9-]+$/, 'Team name must be kebab-case'),
  goal: z.string().min(1),
  status: z.enum(['planning','spawning','running','gating','completing','done','failed','aborted']),
  topology: z.enum(['sequential','parallel','hierarchical','debate','wave']),
  supervisorSessionId: z.string(),
  maxConcurrentMembers: z.number().int().positive().max(50).default(8),
  maxTotalMembers: z.number().int().positive().max(100).default(20),
  tags: z.array(z.string()).default([]),
  metadata: z.record(z.unknown()).default({}),
  totalTokensUsed: z.number().int().nonneg().default(0),
  totalCostUsd: z.number().nonneg().default(0),
});

// ─── Constants ────────────────────────────────────────────────────────────────

/** SQLite schema version — bump when schema changes. */
export const ORCHESTRATOR_SCHEMA_VERSION = 1;

/** Default SQLite DB path. */
export const DEFAULT_ORCHESTRATOR_DB = '.locoworker/orchestrator/orchestrator.db';

/** Default max concurrent members per team. */
export const DEFAULT_MAX_CONCURRENT_MEMBERS = 8;

/** Default max total members per team (all waves combined). */
export const DEFAULT_MAX_TOTAL_MEMBERS = 20;

/** Default file lock TTL in minutes. */
export const DEFAULT_LOCK_TTL_MINUTES = 30;

/** Max messages fetched per query. */
export const MAX_MESSAGES_PER_QUERY = 500;

/** [DELEGATE: role "task description"] marker in agent messages. */
export const DELEGATE_MARKER_REGEX =
  /\[DELEGATE:\s*([a-z_]+)\s+"([^"]+)"\]/g;

/** [APPROVE_PLAN] marker — human approval of delegation plan. */
export const APPROVE_PLAN_MARKER = '[APPROVE_PLAN]';

/** [REJECT_PLAN: reason] marker — human rejection of delegation plan. */
export const REJECT_PLAN_MARKER_REGEX = /\[REJECT_PLAN:\s*([^\]]+)\]/;

/**
 * Built-in agent definitions — the default role catalogue.
 * These are the clean-room equivalents of Claude Code's
 * built-in subagent definition types.
 */
export const BUILT_IN_AGENT_DEFINITIONS: AgentDefinition[] = [
  {
    name: 'supervisor',
    role: 'supervisor',
    description: 'Team lead — plans, delegates, synthesises results, and manages quality gates.',
    systemPromptSuffix: `You are the supervisor agent for this team. Your responsibilities:
1. Break the goal into concrete tasks and assign them to teammates.
2. Monitor progress via the shared task list.
3. Route messages between teammates using [MESSAGE: member-name] content.
4. Synthesise results from all members into a final coherent output.
5. Enforce quality gates — request human approval when required.
6. Use [OBSERVE: ...] to log important decisions for the observation log.`,
    allowedTools: [],
    deniedTools: [],
    permissionProfile: 'full',
    capabilities: ['planning', 'delegation', 'synthesis', 'coordination'],
    supportsParallel: false,
    maxContextTokens: 100_000,
  },
  {
    name: 'architect',
    role: 'architect',
    description: 'System design — API contracts, ADRs, module boundaries.',
    systemPromptSuffix: `You are the architect agent. Focus on:
- Designing clean interfaces and module boundaries.
- Writing Architecture Decision Records (ADRs).
- Identifying risks and trade-offs before implementation begins.
- Do NOT write implementation code — that is the implementer's job.`,
    allowedTools: ['read_file', 'search_files', 'wiki_write', 'list_directory'],
    deniedTools: ['write_file', 'run_shell'],
    permissionProfile: 'read_only',
    capabilities: ['typescript', 'system-design', 'api-design', 'architecture'],
    supportsParallel: false,
    maxContextTokens: 80_000,
  },
  {
    name: 'implementer',
    role: 'implementer',
    description: 'Feature implementation — writes code within assigned file scope.',
    systemPromptSuffix: `You are an implementer agent. Focus on:
- Writing production-quality code within your assigned file scope ONLY.
- Following the architecture and API contracts established by the architect.
- Writing inline documentation and JSDoc.
- Do NOT write tests — that is the test_runner's job.`,
    allowedTools: [],
    deniedTools: [],
    permissionProfile: 'write_local',
    capabilities: ['typescript', 'implementation', 'code-writing'],
    supportsParallel: true,
    maxContextTokens: 60_000,
  },
  {
    name: 'test-runner',
    role: 'test_runner',
    description: 'Test writing and execution — bun test, coverage analysis.',
    systemPromptSuffix: `You are the test runner agent. Focus on:
- Writing comprehensive unit and integration tests.
- Running the test suite and reporting failures.
- Achieving meaningful coverage — not just line coverage.
- Do NOT modify source files — only test files and test utilities.`,
    allowedTools: ['read_file', 'write_file', 'run_shell'],
    deniedTools: [],
    permissionProfile: 'write_local',
    capabilities: ['testing', 'typescript', 'bun-test', 'coverage'],
    supportsParallel: true,
    maxContextTokens: 40_000,
  },
  {
    name: 'security-reviewer',
    role: 'security_reviewer',
    description: 'Security audit — OWASP checks, dependency scanning, auth review.',
    systemPromptSuffix: `You are the security reviewer agent. Focus on:
- Reviewing code for OWASP Top 10 vulnerabilities.
- Checking authentication, authorisation, and input validation.
- Scanning dependencies for known CVEs.
- Producing a structured security report (not fixing code directly).`,
    allowedTools: ['read_file', 'search_files', 'run_shell', 'wiki_write'],
    deniedTools: ['write_file'],
    permissionProfile: 'read_only',
    capabilities: ['security', 'owasp', 'dependency-scanning', 'auth-review'],
    supportsParallel: false,
    maxContextTokens: 50_000,
  },
  {
    name: 'doc-writer',
    role: 'doc_writer',
    description: 'Documentation — CHANGELOG, wiki pages, API docs, README.',
    systemPromptSuffix: `You are the documentation writer agent. Focus on:
- Writing clear, accurate documentation.
- Updating CHANGELOG.md with a structured entry.
- Creating or updating wiki pages using [WIKI: slug] markers.
- Do NOT modify source code — only documentation files.`,
    allowedTools: ['read_file', 'write_file', 'wiki_write'],
    deniedTools: ['run_shell'],
    permissionProfile: 'write_local',
    capabilities: ['documentation', 'markdown', 'changelog', 'wiki'],
    supportsParallel: true,
    maxContextTokens: 30_000,
  },
  {
    name: 'researcher',
    role: 'researcher',
    description: 'Exploration and devil\'s advocate — challenges assumptions, explores alternatives.',
    systemPromptSuffix: `You are the researcher / devil's advocate agent. Focus on:
- Challenging the proposed approach and surfacing risks.
- Exploring alternative implementations.
- Synthesising your findings into a concise report.
- Use [OBSERVE:4: ...] to log critical findings for AutoDream.`,
    allowedTools: ['read_file', 'search_files', 'wiki_read'],
    deniedTools: ['write_file', 'run_shell'],
    permissionProfile: 'read_only',
    capabilities: ['research', 'analysis', 'risk-assessment'],
    supportsParallel: true,
    maxContextTokens: 40_000,
  },
  {
    name: 'reviewer',
    role: 'reviewer',
    description: 'Code review — structured feedback on diffs and pull requests.',
    systemPromptSuffix: `You are the code reviewer agent. Focus on:
- Providing structured, actionable code review feedback.
- Checking for correctness, readability, and adherence to conventions.
- Identifying bugs, edge cases, and missing tests.
- Output a structured review: LGTM | NEEDS_CHANGES | BLOCKING.`,
    allowedTools: ['read_file', 'search_files'],
    deniedTools: ['write_file', 'run_shell'],
    permissionProfile: 'read_only',
    capabilities: ['code-review', 'typescript', 'quality'],
    supportsParallel: false,
    maxContextTokens: 50_000,
  },
];
src/OrchestratorStore.ts
TypeScript

/**
 * OrchestratorStore — SQLite persistence layer for @locoworker/orchestrator
 *
 * Tables:
 *   orch_teams          — team records
 *   orch_members        — team member records
 *   orch_messages       — append-only message log
 *   orch_shared_tasks   — shared dependency-aware task list
 *   orch_file_locks     — file-scope lock records
 *   orch_results        — append-only sub-agent result records
 *   orch_meta           — key-value store
 *
 * Design principles (identical to other LocoWorker packages):
 *  - OrchestratorStore is the ONLY class that writes to the DB.
 *  - WAL mode + NORMAL sync for performance.
 *  - append-only for messages and results (no UPDATE/DELETE).
 *  - All column names follow established monorepo convention:
 *    snake_case, tags_csv, session_id, created_at/updated_at.
 *  - Foreign keys ON with CASCADE where appropriate.
 */

import Database, { type Database as DB } from 'better-sqlite3';
import { randomUUID } from 'node:crypto';
import { existsSync, mkdirSync } from 'node:fs';
import { dirname } from 'node:path';

import type {
  OrchestratorTeam,
  TeamRow,
  TeamMember,
  MemberRow,
  SharedTask,
  SharedTaskRow,
  TeamMessage,
  MessageRow,
  AgentResult,
  ResultRow,
  FileLock,
  FileLockRow,
  TeamStatus,
  MemberStatus,
  SharedTaskStatus,
  FileLockStatus,
  GateType,
  TeamQueryOptions,
  MemberQueryOptions,
  SharedTaskQueryOptions,
  OrchestratorStoreStats,
  QualityGate,
  MessageChannel,
  AgentRole,
  TeamTopology,
  AgentPermissionProfile,
} from './types/orchestrator.types.js';

import {
  ORCHESTRATOR_SCHEMA_VERSION,
  DEFAULT_LOCK_TTL_MINUTES,
} from './types/orchestrator.types.js';

// ─── Helpers ──────────────────────────────────────────────────────────────────

function nowIso(): string {
  return new Date().toISOString();
}

function ensureDir(filePath: string): void {
  if (filePath === ':memory:') return;
  const dir = dirname(filePath);
  if (!existsSync(dir)) mkdirSync(dir, { recursive: true });
}

// ─── OrchestratorStore ────────────────────────────────────────────────────────

export class OrchestratorStore {
  readonly db: DB;

  constructor(dbPath: string) {
    ensureDir(dbPath);
    this.db = new Database(dbPath);
    this.applyPragmas();
    this.createSchema();
    this.assertSchemaVersion();
  }

  // ─── Lifecycle ──────────────────────────────────────────────────────────────

  close(): void {
    this.db.close();
  }

  // ─── Team CRUD ──────────────────────────────────────────────────────────────

  createTeam(
    input: Omit<OrchestratorTeam,
      'id' | 'createdAt' | 'updatedAt' | 'totalTokensUsed' | 'totalCostUsd'>,
  ): OrchestratorTeam {
    const now = nowIso();
    const team: OrchestratorTeam = {
      ...input,
      id: randomUUID(),
      totalTokensUsed: 0,
      totalCostUsd: 0,
      createdAt: now,
      updatedAt: now,
    };

    this.db.prepare<TeamRow>(/* sql */`
      INSERT INTO orch_teams
        (id, name, goal, status, topology, supervisor_session_id, supervisor_member_id,
         max_concurrent_members, max_total_members, gates_json, tags_csv, metadata_json,
         created_at, updated_at, completed_at, error_message, total_tokens_used, total_cost_usd)
      VALUES
        (@id, @name, @goal, @status, @topology, @supervisor_session_id, @supervisor_member_id,
         @max_concurrent_members, @max_total_members, @gates_json, @tags_csv, @metadata_json,
         @created_at, @updated_at, @completed_at, @error_message, @total_tokens_used, @total_cost_usd)
    `).run(this._teamToRow(team));

    return team;
  }

  getTeam(id: string): OrchestratorTeam | undefined {
    const row = this.db
      .prepare<[string], TeamRow>('SELECT * FROM orch_teams WHERE id = ?')
      .get(id);
    return row ? this._rowToTeam(row) : undefined;
  }

  queryTeams(options: TeamQueryOptions = {}): TeamRow[] {
    const conditions: string[] = [];
    const params: (string | number)[] = [];
    const limit = options.limit ?? 50;
    const offset = options.offset ?? 0;

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      conditions.push(`status IN (${statuses.map(() => '?').join(', ')})`);
      params.push(...statuses);
    }

    if (options.topology) {
      conditions.push('topology = ?');
      params.push(options.topology);
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';
    return this.db
      .prepare<(string | number)[], TeamRow>(
        `SELECT * FROM orch_teams ${where} ORDER BY created_at DESC LIMIT ? OFFSET ?`,
      )
      .all(...params, limit, offset);
  }

  updateTeam(
    id: string,
    updates: Partial<Omit<OrchestratorTeam, 'id' | 'createdAt'>>,
  ): OrchestratorTeam | undefined {
    const existing = this.getTeam(id);
    if (!existing) return undefined;
    const merged: OrchestratorTeam = { ...existing, ...updates, id, updatedAt: nowIso() };
    this.db.prepare(/* sql */`
      UPDATE orch_teams SET
        name = @name, goal = @goal, status = @status, topology = @topology,
        supervisor_session_id = @supervisor_session_id,
        supervisor_member_id = @supervisor_member_id,
        max_concurrent_members = @max_concurrent_members,
        max_total_members = @max_total_members,
        gates_json = @gates_json, tags_csv = @tags_csv, metadata_json = @metadata_json,
        updated_at = @updated_at, completed_at = @completed_at,
        error_message = @error_message,
        total_tokens_used = @total_tokens_used, total_cost_usd = @total_cost_usd
      WHERE id = @id
    `).run(this._teamToRow(merged));
    return merged;
  }

  transitionTeam(
    id: string,
    newStatus: TeamStatus,
    opts: { errorMessage?: string } = {},
  ): OrchestratorTeam | undefined {
    const isTerminal =
      newStatus === 'done' || newStatus === 'failed' || newStatus === 'aborted';
    return this.updateTeam(id, {
      status: newStatus,
      ...(isTerminal ? { completedAt: nowIso() } : {}),
      ...(opts.errorMessage ? { errorMessage: opts.errorMessage } : {}),
    });
  }

  // ─── Member CRUD ────────────────────────────────────────────────────────────

  createMember(
    input: Omit<TeamMember, 'id' | 'createdAt' | 'updatedAt' | 'tokensUsed' | 'costUsd'>,
  ): TeamMember {
    const now = nowIso();
    const member: TeamMember = {
      ...input,
      id: randomUUID(),
      tokensUsed: 0,
      costUsd: 0,
      createdAt: now,
      updatedAt: now,
    };
    this.db.prepare<MemberRow>(/* sql */`
      INSERT INTO orch_members
        (id, team_id, name, role, status, session_id, permission_profile,
         file_scope_json, custom_prompt, model_id, wave, depends_on_json,
         assigned_task_ids_json, tokens_used, cost_usd,
         created_at, updated_at, started_at, completed_at, error_message)
      VALUES
        (@id, @team_id, @name, @role, @status, @session_id, @permission_profile,
         @file_scope_json, @custom_prompt, @model_id, @wave, @depends_on_json,
         @assigned_task_ids_json, @tokens_used, @cost_usd,
         @created_at, @updated_at, @started_at, @completed_at, @error_message)
    `).run(this._memberToRow(member));
    return member;
  }

  getMember(id: string): TeamMember | undefined {
    const row = this.db
      .prepare<[string], MemberRow>('SELECT * FROM orch_members WHERE id = ?')
      .get(id);
    return row ? this._rowToMember(row) : undefined;
  }

  getMemberByName(teamId: string, name: string): TeamMember | undefined {
    const row = this.db
      .prepare<[string, string], MemberRow>(
        'SELECT * FROM orch_members WHERE team_id = ? AND name = ?',
      )
      .get(teamId, name);
    return row ? this._rowToMember(row) : undefined;
  }

  queryMembers(options: MemberQueryOptions): MemberRow[] {
    const conditions: string[] = ['team_id = ?'];
    const params: (string | number)[] = [options.teamId];

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      conditions.push(`status IN (${statuses.map(() => '?').join(', ')})`);
      params.push(...statuses);
    }
    if (options.role) {
      conditions.push('role = ?');
      params.push(options.role);
    }
    if (options.wave !== undefined) {
      conditions.push('wave = ?');
      params.push(options.wave);
    }

    return this.db
      .prepare<(string | number)[], MemberRow>(
        `SELECT * FROM orch_members WHERE ${conditions.join(' AND ')} ORDER BY wave, created_at`,
      )
      .all(...params);
  }

  updateMember(
    id: string,
    updates: Partial<Omit<TeamMember, 'id' | 'createdAt'>>,
  ): TeamMember | undefined {
    const existing = this.getMember(id);
    if (!existing) return undefined;
    const merged: TeamMember = { ...existing, ...updates, id, updatedAt: nowIso() };
    this.db.prepare(/* sql */`
      UPDATE orch_members SET
        name = @name, role = @role, status = @status, session_id = @session_id,
        permission_profile = @permission_profile, file_scope_json = @file_scope_json,
        custom_prompt = @custom_prompt, model_id = @model_id, wave = @wave,
        depends_on_json = @depends_on_json,
        assigned_task_ids_json = @assigned_task_ids_json,
        tokens_used = @tokens_used, cost_usd = @cost_usd,
        updated_at = @updated_at, started_at = @started_at,
        completed_at = @completed_at, error_message = @error_message
      WHERE id = @id
    `).run(this._memberToRow(merged));
    return merged;
  }

  transitionMember(
    id: string,
    newStatus: MemberStatus,
    opts: { errorMessage?: string; sessionId?: string } = {},
  ): TeamMember | undefined {
    const isTerminal = newStatus === 'done' || newStatus === 'failed' || newStatus === 'evicted';
    const isStarting = newStatus === 'working' || newStatus === 'idle';
    return this.updateMember(id, {
      status: newStatus,
      ...(isTerminal ? { completedAt: nowIso() } : {}),
      ...(isStarting ? { startedAt: nowIso() } : {}),
      ...(opts.errorMessage ? { errorMessage: opts.errorMessage } : {}),
      ...(opts.sessionId ? { sessionId: opts.sessionId } : {}),
    });
  }

  // ─── Shared tasks ───────────────────────────────────────────────────────────

  createSharedTask(
    input: Omit<SharedTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'>,
  ): SharedTask {
    const now = nowIso();
    const task: SharedTask = { ...input, id: randomUUID(), attemptCount: 0, createdAt: now, updatedAt: now };
    this.db.prepare<SharedTaskRow>(/* sql */`
      INSERT INTO orch_shared_tasks
        (id, team_id, title, description, status, assigned_member_id, blocked_by_json,
         parallel_safe, file_paths_json, gate, tags_csv, order_index,
         created_at, updated_at, completed_at, error_message, attempt_count)
      VALUES
        (@id, @team_id, @title, @description, @status, @assigned_member_id, @blocked_by_json,
         @parallel_safe, @file_paths_json, @gate, @tags_csv, @order_index,
         @created_at, @updated_at, @completed_at, @error_message, @attempt_count)
    `).run(this._sharedTaskToRow(task));
    return task;
  }

  getSharedTask(id: string): SharedTask | undefined {
    const row = this.db
      .prepare<[string], SharedTaskRow>('SELECT * FROM orch_shared_tasks WHERE id = ?')
      .get(id);
    return row ? this._rowToSharedTask(row) : undefined;
  }

  querySharedTasks(options: SharedTaskQueryOptions): SharedTaskRow[] {
    const conditions: string[] = ['team_id = ?'];
    const params: (string | number)[] = [options.teamId];

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      conditions.push(`status IN (${statuses.map(() => '?').join(', ')})`);
      params.push(...statuses);
    }
    if (options.assignedMemberId) {
      conditions.push('assigned_member_id = ?');
      params.push(options.assignedMemberId);
    }

    const limit = options.limit ?? 200;
    const rows = this.db
      .prepare<(string | number)[], SharedTaskRow>(
        `SELECT * FROM orch_shared_tasks WHERE ${conditions.join(' AND ')}
         ORDER BY order_index, created_at LIMIT ?`,
      )
      .all(...params, limit);

    if (options.readyOnly) {
      // "ready" = pending AND all blockedBy tasks are completed
      const completedIds = new Set(
        this.db
          .prepare<[string], { id: string }>(
            "SELECT id FROM orch_shared_tasks WHERE team_id = ? AND status = 'completed'",
          )
          .all(options.teamId)
          .map((r) => r.id),
      );
      return rows.filter((r) => {
        if (r.status !== 'pending') return false;
        const blockedBy: string[] = JSON.parse(r.blocked_by_json);
        return blockedBy.every((bid) => completedIds.has(bid));
      });
    }

    return rows;
  }

  updateSharedTask(
    id: string,
    updates: Partial<Omit<SharedTask, 'id' | 'createdAt'>>,
  ): SharedTask | undefined {
    const existing = this.getSharedTask(id);
    if (!existing) return undefined;
    const merged: SharedTask = { ...existing, ...updates, id, updatedAt: nowIso() };
    this.db.prepare(/* sql */`
      UPDATE orch_shared_tasks SET
        title = @title, description = @description, status = @status,
        assigned_member_id = @assigned_member_id, blocked_by_json = @blocked_by_json,
        parallel_safe = @parallel_safe, file_paths_json = @file_paths_json,
        gate = @gate, tags_csv = @tags_csv, order_index = @order_index,
        updated_at = @updated_at, completed_at = @completed_at,
        error_message = @error_message, attempt_count = @attempt_count
      WHERE id = @id
    `).run(this._sharedTaskToRow(merged));
    return merged;
  }

  transitionSharedTask(
    id: string,
    newStatus: SharedTaskStatus,
    opts: { assignedMemberId?: string; errorMessage?: string } = {},
  ): SharedTask | undefined {
    const isTerminal = newStatus === 'completed' || newStatus === 'failed' || newStatus === 'skipped';
    const existing = this.getSharedTask(id);
    if (!existing) return undefined;
    return this.updateSharedTask(id, {
      status: newStatus,
      ...(isTerminal ? { completedAt: nowIso() } : {}),
      ...(opts.assignedMemberId !== undefined ? { assignedMemberId: opts.assignedMemberId } : {}),
      ...(opts.errorMessage ? { errorMessage: opts.errorMessage } : {}),
      ...(newStatus === 'in_progress' ? { attemptCount: existing.attemptCount + 1 } : {}),
    });
  }

  /**
   * Auto-skip tasks whose all blockedBy tasks have failed/skipped.
   * Returns the number of tasks auto-skipped.
   */
  autoSkipBlockedTasks(teamId: string): number {
    const blockedRows = this.db
      .prepare<[string], SharedTaskRow>(
        "SELECT * FROM orch_shared_tasks WHERE team_id = ? AND status = 'blocked'",
      )
      .all(teamId);

    const terminalIds = new Set(
      this.db
        .prepare<[string], { id: string }>(
          "SELECT id FROM orch_shared_tasks WHERE team_id = ? AND status IN ('failed','skipped')",
        )
        .all(teamId)
        .map((r) => r.id),
    );

    let skipped = 0;
    for (const row of blockedRows) {
      const blockedBy: string[] = JSON.parse(row.blocked_by_json);
      const allBlockersTerminal = blockedBy.every((bid) => terminalIds.has(bid));
      if (allBlockersTerminal) {
        this.transitionSharedTask(row.id, 'skipped', {
          errorMessage: 'All upstream dependencies failed or were skipped',
        });
        skipped++;
      }
    }
    return skipped;
  }

  // ─── Messages (append-only) ─────────────────────────────────────────────────

  sendMessage(
    input: Omit<TeamMessage, 'id' | 'sentAt' | 'read'>,
  ): TeamMessage {
    const message: TeamMessage = {
      ...input,
      id: randomUUID(),
      sentAt: nowIso(),
      read: false,
    };
    this.db.prepare(/* sql */`
      INSERT INTO orch_messages
        (id, team_id, from_member_id, from_member_name, to_member_id, to_member_name,
         channel, content, payload_json, read, sent_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 0, ?)
    `).run(
      message.id,
      message.teamId,
      message.fromMemberId ?? null,
      message.fromMemberName ?? null,
      message.toMemberId ?? null,
      message.toMemberName ?? null,
      message.channel,
      message.content,
      message.payload ? JSON.stringify(message.payload) : null,
      message.sentAt,
    );
    return message;
  }

  getMessages(
    teamId: string,
    opts: {
      memberId?: string;         // filter to messages for this member (to_member_id = memberId OR broadcast)
      channel?: MessageChannel;
      unreadOnly?: boolean;
      limit?: number;
    } = {},
  ): MessageRow[] {
    const conditions: string[] = ['team_id = ?'];
    const params: (string | number)[] = [teamId];
    const limit = opts.limit ?? 100;

    if (opts.memberId) {
      // Direct messages to this member OR broadcast messages
      conditions.push(
        "(to_member_id = ? OR channel IN ('broadcast', 'system'))",
      );
      params.push(opts.memberId);
    }
    if (opts.channel) {
      conditions.push('channel = ?');
      params.push(opts.channel);
    }
    if (opts.unreadOnly) {
      conditions.push('read = 0');
    }

    return this.db
      .prepare<(string | number)[], MessageRow>(
        `SELECT * FROM orch_messages WHERE ${conditions.join(' AND ')}
         ORDER BY sent_at ASC LIMIT ?`,
      )
      .all(...params, limit);
  }

  markMessagesRead(messageIds: string[]): number {
    if (messageIds.length === 0) return 0;
    const placeholders = messageIds.map(() => '?').join(', ');
    const result = this.db
      .prepare(`UPDATE orch_messages SET read = 1 WHERE id IN (${placeholders})`)
      .run(...messageIds);
    return result.changes;
  }

  // ─── Results (append-only) ──────────────────────────────────────────────────

  appendResult(
    input: Omit<AgentResult, 'id' | 'producedAt'>,
  ): AgentResult {
    const result: AgentResult = {
      ...input,
      id: randomUUID(),
      producedAt: nowIso(),
    };
    this.db.prepare(/* sql */`
      INSERT INTO orch_results
        (id, team_id, member_id, member_name, member_role, task_id,
         content, files_modified_json, gate_approved, produced_at, tokens_used)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      result.id,
      result.teamId,
      result.memberId,
      result.memberName,
      result.memberRole,
      result.taskId ?? null,
      result.content,
      JSON.stringify(result.filesModified),
      result.gateApproved === undefined ? null : result.gateApproved ? 1 : 0,
      result.producedAt,
      result.tokensUsed,
    );
    return result;
  }

  getResults(teamId: string, memberId?: string): ResultRow[] {
    if (memberId) {
      return this.db
        .prepare<[string, string], ResultRow>(
          'SELECT * FROM orch_results WHERE team_id = ? AND member_id = ? ORDER BY produced_at',
        )
        .all(teamId, memberId);
    }
    return this.db
      .prepare<[string], ResultRow>(
        'SELECT * FROM orch_results WHERE team_id = ? ORDER BY produced_at',
      )
      .all(teamId);
  }

  // ─── File locks ─────────────────────────────────────────────────────────────

  acquireFileLock(
    teamId: string,
    memberId: string,
    memberName: string,
    filePath: string,
    taskId?: string,
    ttlMinutes = DEFAULT_LOCK_TTL_MINUTES,
  ): FileLock | { conflict: FileLock } {
    // Check if file is already locked
    const existing = this.db
      .prepare<[string, string], FileLockRow>(
        "SELECT * FROM orch_file_locks WHERE team_id = ? AND file_path = ? AND status = 'held'",
      )
      .get(teamId, filePath);

    if (existing) {
      // Check if lock has expired
      const expiresAt = new Date(existing.expires_at).getTime();
      if (Date.now() > expiresAt) {
        // Expire the old lock
        this.db
          .prepare(
            "UPDATE orch_file_locks SET status = 'expired', released_at = ? WHERE id = ?",
          )
          .run(nowIso(), existing.id);
      } else {
        // Live conflict
        return { conflict: this._rowToFileLock(existing) };
      }
    }

    const now = nowIso();
    const expiresAt = new Date(Date.now() + ttlMinutes * 60_000).toISOString();
    const lock: FileLock = {
      id: randomUUID(),
      teamId,
      memberId,
      memberName,
      filePath,
      status: 'held',
      acquiredAt: now,
      expiresAt,
      taskId,
    };

    this.db.prepare(/* sql */`
      INSERT INTO orch_file_locks
        (id, team_id, member_id, member_name, file_path, status,
         acquired_at, released_at, expires_at, task_id)
      VALUES (?, ?, ?, ?, ?, 'held', ?, NULL, ?, ?)
    `).run(lock.id, teamId, memberId, memberName, filePath, now, expiresAt, taskId ?? null);

    return lock;
  }

  releaseFileLock(lockId: string): boolean {
    const result = this.db
      .prepare(
        "UPDATE orch_file_locks SET status = 'released', released_at = ? WHERE id = ? AND status = 'held'",
      )
      .run(nowIso(), lockId);
    return result.changes > 0;
  }

  releaseAllLocksForMember(teamId: string, memberId: string): number {
    const result = this.db
      .prepare(
        "UPDATE orch_file_locks SET status = 'released', released_at = ? WHERE team_id = ? AND member_id = ? AND status = 'held'",
      )
      .run(nowIso(), teamId, memberId);
    return result.changes;
  }

  expireStaleFileLocks(teamId: string): number {
    const result = this.db
      .prepare(
        "UPDATE orch_file_locks SET status = 'expired', released_at = ? WHERE team_id = ? AND status = 'held' AND expires_at < datetime('now')",
      )
      .run(nowIso(), teamId);
    return result.changes;
  }

  getActiveFileLocks(teamId: string): FileLockRow[] {
    return this.db
      .prepare<[string], FileLockRow>(
        "SELECT * FROM orch_file_locks WHERE team_id = ? AND status = 'held' ORDER BY acquired_at",
      )
      .all(teamId);
  }

  // ─── Stats ─────────────────────────────────────────────────────────────────

  getStats(): OrchestratorStoreStats {
    const totalTeams = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM orch_teams')
      .get()?.count ?? 0);

    const activeTeams = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM orch_teams WHERE status IN ('planning','spawning','running','gating','completing')",
      )
      .get()?.count ?? 0);

    const totalMembers = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM orch_members')
      .get()?.count ?? 0);

    const activeMembers = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM orch_members WHERE status IN ('spawning','idle','working','waiting','reviewing')",
      )
      .get()?.count ?? 0);

    const totalMessages = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM orch_messages')
      .get()?.count ?? 0);

    const unreadMessages = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM orch_messages WHERE read = 0')
      .get()?.count ?? 0);

    const totalSharedTasks = (this.db
      .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM orch_shared_tasks')
      .get()?.count ?? 0);

    const pendingSharedTasks = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM orch_shared_tasks WHERE status IN ('pending','in_progress','blocked','gating')",
      )
      .get()?.count ?? 0);

    const completedSharedTasks = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM orch_shared_tasks WHERE status = 'completed'",
      )
      .get()?.count ?? 0);

    const activeFileLocks = (this.db
      .prepare<[], { count: number }>(
        "SELECT COUNT(*) as count FROM orch_file_locks WHERE status = 'held'",
      )
      .get()?.count ?? 0);

    const pageCount = (this.db.prepare('PRAGMA page_count').get() as { page_count: number })
      .page_count;
    const pageSize = (this.db.prepare('PRAGMA page_size').get() as { page_size: number })
      .page_size;

    return {
      totalTeams,
      activeTeams,
      totalMembers,
      activeMembers,
      totalMessages,
      unreadMessages,
      totalSharedTasks,
      pendingSharedTasks,
      completedSharedTasks,
      activeFileLocks,
      dbSizeBytes: pageCount * pageSize,
    };
  }

  // ─── Meta ───────────────────────────────────────────────────────────────────

  getMeta(key: string): string | null {
    return (
      this.db
        .prepare<[string], { value: string }>('SELECT value FROM orch_meta WHERE key = ?')
        .get(key)?.value ?? null
    );
  }

  setMeta(key: string, value: string): void {
    this.db
      .prepare('INSERT OR REPLACE INTO orch_meta(key, value) VALUES (?, ?)')
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
    this.db.exec(/* sql */`
      CREATE TABLE IF NOT EXISTS orch_teams (
        id                      TEXT PRIMARY KEY,
        name                    TEXT NOT NULL UNIQUE,
        goal                    TEXT NOT NULL,
        status                  TEXT NOT NULL DEFAULT 'planning',
        topology                TEXT NOT NULL DEFAULT 'parallel',
        supervisor_session_id   TEXT NOT NULL,
        supervisor_member_id    TEXT,
        max_concurrent_members  INTEGER NOT NULL DEFAULT 8,
        max_total_members       INTEGER NOT NULL DEFAULT 20,
        gates_json              TEXT NOT NULL DEFAULT '[]',
        tags_csv                TEXT NOT NULL DEFAULT '',
        metadata_json           TEXT NOT NULL DEFAULT '{}',
        created_at              TEXT NOT NULL,
        updated_at              TEXT NOT NULL,
        completed_at            TEXT,
        error_message           TEXT,
        total_tokens_used       INTEGER NOT NULL DEFAULT 0,
        total_cost_usd          REAL NOT NULL DEFAULT 0.0
      );

      CREATE INDEX IF NOT EXISTS idx_orch_teams_status   ON orch_teams(status);
      CREATE INDEX IF NOT EXISTS idx_orch_teams_created  ON orch_teams(created_at DESC);

      CREATE TABLE IF NOT EXISTS orch_members (
        id                    TEXT PRIMARY KEY,
        team_id               TEXT NOT NULL,
        name                  TEXT NOT NULL,
        role                  TEXT NOT NULL,
        status                TEXT NOT NULL DEFAULT 'spawning',
        session_id            TEXT,
        permission_profile    TEXT NOT NULL DEFAULT 'standard',
        file_scope_json       TEXT NOT NULL DEFAULT '[]',
        custom_prompt         TEXT,
        model_id              TEXT,
        wave                  INTEGER NOT NULL DEFAULT 0,
        depends_on_json       TEXT NOT NULL DEFAULT '[]',
        assigned_task_ids_json TEXT NOT NULL DEFAULT '[]',
        tokens_used           INTEGER NOT NULL DEFAULT 0,
        cost_usd              REAL NOT NULL DEFAULT 0.0,
        created_at            TEXT NOT NULL,
        updated_at            TEXT NOT NULL,
        started_at            TEXT,
        completed_at          TEXT,
        error_message         TEXT,
        FOREIGN KEY (team_id) REFERENCES orch_teams(id) ON DELETE CASCADE,
        UNIQUE (team_id, name)
      );

      CREATE INDEX IF NOT EXISTS idx_orch_members_team    ON orch_members(team_id);
      CREATE INDEX IF NOT EXISTS idx_orch_members_status  ON orch_members(status);
      CREATE INDEX IF NOT EXISTS idx_orch_members_wave    ON orch_members(wave);

      CREATE TABLE IF NOT EXISTS orch_messages (
        id               TEXT PRIMARY KEY,
        team_id          TEXT NOT NULL,
        from_member_id   TEXT,
        from_member_name TEXT,
        to_member_id     TEXT,
        to_member_name   TEXT,
        channel          TEXT NOT NULL DEFAULT 'direct',
        content          TEXT NOT NULL,
        payload_json     TEXT,
        read             INTEGER NOT NULL DEFAULT 0,
        sent_at          TEXT NOT NULL,
        FOREIGN KEY (team_id) REFERENCES orch_teams(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_orch_msgs_team     ON orch_messages(team_id);
      CREATE INDEX IF NOT EXISTS idx_orch_msgs_to       ON orch_messages(to_member_id)
        WHERE to_member_id IS NOT NULL;
      CREATE INDEX IF NOT EXISTS idx_orch_msgs_unread   ON orch_messages(read)
        WHERE read = 0;
      CREATE INDEX IF NOT EXISTS idx_orch_msgs_sent     ON orch_messages(sent_at ASC);

      CREATE TABLE IF NOT EXISTS orch_shared_tasks (
        id                 TEXT PRIMARY KEY,
        team_id            TEXT NOT NULL,
        title              TEXT NOT NULL,
        description        TEXT,
        status             TEXT NOT NULL DEFAULT 'pending',
        assigned_member_id TEXT,
        blocked_by_json    TEXT NOT NULL DEFAULT '[]',
        parallel_safe      INTEGER NOT NULL DEFAULT 1,
        file_paths_json    TEXT NOT NULL DEFAULT '[]',
        gate               TEXT,
        tags_csv           TEXT NOT NULL DEFAULT '',
        order_index        INTEGER NOT NULL DEFAULT 0,
        created_at         TEXT NOT NULL,
        updated_at         TEXT NOT NULL,
        completed_at       TEXT,
        error_message      TEXT,
        attempt_count      INTEGER NOT NULL DEFAULT 0,
        FOREIGN KEY (team_id) REFERENCES orch_teams(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_orch_tasks_team    ON orch_shared_tasks(team_id);
      CREATE INDEX IF NOT EXISTS idx_orch_tasks_status  ON orch_shared_tasks(status);
      CREATE INDEX IF NOT EXISTS idx_orch_tasks_member  ON orch_shared_tasks(assigned_member_id)
        WHERE assigned_member_id IS NOT NULL;

      CREATE TABLE IF NOT EXISTS orch_file_locks (
        id           TEXT PRIMARY KEY,
        team_id      TEXT NOT NULL,
        member_id    TEXT NOT NULL,
        member_name  TEXT NOT NULL,
        file_path    TEXT NOT NULL,
        status       TEXT NOT NULL DEFAULT 'held',
        acquired_at  TEXT NOT NULL,
        released_at  TEXT,
        expires_at   TEXT NOT NULL,
        task_id      TEXT,
        FOREIGN KEY (team_id) REFERENCES orch_teams(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_orch_locks_team    ON orch_file_locks(team_id);
      CREATE INDEX IF NOT EXISTS idx_orch_locks_file    ON orch_file_locks(file_path)
        WHERE status = 'held';
      CREATE INDEX IF NOT EXISTS idx_orch_locks_member  ON orch_file_locks(member_id);

      CREATE TABLE IF NOT EXISTS orch_results (
        id                 TEXT PRIMARY KEY,
        team_id            TEXT NOT NULL,
        member_id          TEXT NOT NULL,
        member_name        TEXT NOT NULL,
        member_role        TEXT NOT NULL,
        task_id            TEXT,
        content            TEXT NOT NULL,
        files_modified_json TEXT NOT NULL DEFAULT '[]',
        gate_approved      INTEGER,
        produced_at        TEXT NOT NULL,
        tokens_used        INTEGER NOT NULL DEFAULT 0,
        FOREIGN KEY (team_id) REFERENCES orch_teams(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_orch_results_team   ON orch_results(team_id);
      CREATE INDEX IF NOT EXISTS idx_orch_results_member ON orch_results(member_id);

      CREATE TABLE IF NOT EXISTS orch_meta (
        key   TEXT PRIMARY KEY,
        value TEXT NOT NULL
      );
    `);
  }

  private assertSchemaVersion(): void {
    const stored = this.getMeta('schema_version');
    if (stored === null) {
      this.setMeta('schema_version', String(ORCHESTRATOR_SCHEMA_VERSION));
    } else if (Number(stored) !== ORCHESTRATOR_SCHEMA_VERSION) {
      throw new Error(
        `OrchestratorStore: schema version mismatch. ` +
          `DB has v${stored}, code expects v${ORCHESTRATOR_SCHEMA_VERSION}. ` +
          `Delete the DB to rebuild.`,
      );
    }
  }

  // ─── Row ↔ domain conversions ──────────────────────────────────────────────

  _teamToRow(t: OrchestratorTeam): TeamRow {
    return {
      id: t.id,
      name: t.name,
      goal: t.goal,
      status: t.status,
      topology: t.topology,
      supervisor_session_id: t.supervisorSessionId,
      supervisor_member_id: t.supervisorMemberId ?? null,
      max_concurrent_members: t.maxConcurrentMembers,
      max_total_members: t.maxTotalMembers,
      gates_json: JSON.stringify(t.gates),
      tags_csv: t.tags.join(','),
      metadata_json: JSON.stringify(t.metadata),
      created_at: t.createdAt,
      updated_at: t.updatedAt,
      completed_at: t.completedAt ?? null,
      error_message: t.errorMessage ?? null,
      total_tokens_used: t.totalTokensUsed,
      total_cost_usd: t.totalCostUsd,
    };
  }

  _rowToTeam(row: TeamRow): OrchestratorTeam {
    return {
      id: row.id,
      name: row.name,
      goal: row.goal,
      status: row.status,
      topology: row.topology,
      supervisorSessionId: row.supervisor_session_id,
      supervisorMemberId: row.supervisor_member_id ?? undefined,
      maxConcurrentMembers: row.max_concurrent_members,
      maxTotalMembers: row.max_total_members,
      gates: JSON.parse(row.gates_json) as QualityGate[],
      tags: row.tags_csv ? row.tags_csv.split(',').filter(Boolean) : [],
      metadata: JSON.parse(row.metadata_json) as Record<string, unknown>,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
      completedAt: row.completed_at ?? undefined,
      errorMessage: row.error_message ?? undefined,
      totalTokensUsed: row.total_tokens_used,
      totalCostUsd: row.total_cost_usd,
    };
  }

  _memberToRow(m: TeamMember): MemberRow {
    return {
      id: m.id,
      team_id: m.teamId,
      name: m.name,
      role: m.role,
      status: m.status,
      session_id: m.sessionId ?? null,
      permission_profile: m.permissionProfile,
      file_scope_json: JSON.stringify(m.fileScope),
      custom_prompt: m.customPrompt ?? null,
      model_id: m.modelId ?? null,
      wave: m.wave,
      depends_on_json: JSON.stringify(m.dependsOn),
      assigned_task_ids_json: JSON.stringify(m.assignedTaskIds),
      tokens_used: m.tokensUsed,
      cost_usd: m.costUsd,
      created_at: m.createdAt,
      updated_at: m.updatedAt,
      started_at: m.startedAt ?? null,
      completed_at: m.completedAt ?? null,
      error_message: m.errorMessage ?? null,
    };
  }

  _rowToMember(row: MemberRow): TeamMember {
    return {
      id: row.id,
      teamId: row.team_id,
      name: row.name,
      role: row.role,
      status: row.status,
      sessionId: row.session_id ?? undefined,
      permissionProfile: row.permission_profile,
      fileScope: JSON.parse(row.file_scope_json) as string[],
      customPrompt: row.custom_prompt ?? undefined,
      modelId: row.model_id ?? undefined,
      wave: row.wave,
      dependsOn: JSON.parse(row.depends_on_json) as string[],
      assignedTaskIds: JSON.parse(row.assigned_task_ids_json) as string[],
      tokensUsed: row.tokens_used,
      costUsd: row.cost_usd,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
      startedAt: row.started_at ?? undefined,
      completedAt: row.completed_at ?? undefined,
      errorMessage: row.error_message ?? undefined,
    };
  }

  private _sharedTaskToRow(t: SharedTask): SharedTaskRow {
    return {
      id: t.id,
      team_id: t.teamId,
      title: t.title,
      description: t.description ?? null,
      status: t.status,
      assigned_member_id: t.assignedMemberId ?? null,
      blocked_by_json: JSON.stringify(t.blockedBy),
      parallel_safe: t.parallelSafe ? 1 : 0,
      file_paths_json: JSON.stringify(t.filePaths),
      gate: t.gate ?? null,
      tags_csv: t.tags.join(','),
      order_index: t.order,
      created_at: t.createdAt,
      updated_at: t.updatedAt,
      completed_at: t.completedAt ?? null,
      error_message: t.errorMessage ?? null,
      attempt_count: t.attemptCount,
    };
  }

  _rowToSharedTask(row: SharedTaskRow): SharedTask {
    return {
      id: row.id,
      teamId: row.team_id,
      title: row.title,
      description: row.description ?? undefined,
      status: row.status,
      assignedMemberId: row.assigned_member_id ?? undefined,
      blockedBy: JSON.parse(row.blocked_by_json) as string[],
      parallelSafe: row.parallel_safe === 1,
      filePaths: JSON.parse(row.file_paths_json) as string[],
      gate: row.gate ?? undefined,
      tags: row.tags_csv ? row.tags_csv.split(',').filter(Boolean) : [],
      order: row.order_index,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
      completedAt: row.completed_at ?? undefined,
      errorMessage: row.error_message ?? undefined,
      attemptCount: row.attempt_count,
    };
  }

  _rowToFileLock(row: FileLockRow): FileLock {
    return {
      id: row.id,
      teamId: row.team_id,
      memberId: row.member_id,
      memberName: row.member_name,
      filePath: row.file_path,
      status: row.status,
      acquiredAt: row.acquired_at,
      releasedAt: row.released_at ?? undefined,
      expiresAt: row.expires_at,
      taskId: row.task_id ?? undefined,
    };
  }
}
src/AgentRegistry.ts
TypeScript

/**
 * AgentRegistry — capability-based agent role catalogue for @locoworker/orchestrator
 *
 * AgentRegistry is the LocoWorker equivalent of Claude Code's
 * subagent definition system (.claude/agents/*.md files).
 *
 * Responsibilities:
 *  - Store and retrieve AgentDefinition records by name or role
 *  - Find definitions by required capabilities (tag-based matching)
 *  - Validate definition schemas (Zod)
 *  - Load built-in definitions on first init
 *  - Support user-registered custom definitions (runtime registration)
 *  - Generate the system prompt suffix for a given role at spawn time
 *
 * Design:
 *  - AgentRegistry is pure in-memory (no DB dependency).
 *  - Definitions can optionally be persisted to a JSON file
 *    (.locoworker/orchestrator/agents.json) for cross-process reuse.
 *  - Built-in definitions are always registered first; custom definitions
 *    can override them by registering with the same name.
 *
 * Integration:
 *  - AgentSpawner calls AgentRegistry.get(name) to get the definition
 *    used when building the sub-agent's system prompt.
 *  - DelegationPlanner calls AgentRegistry.findByCapabilities() to select
 *    appropriate roles for each delegation wave.
 *  - OrchestratorEngine calls AgentRegistry.listAll() for reporting.
 */

import { readFile, writeFile, mkdir } from 'node:fs/promises';
import { dirname } from 'node:path';
import { existsSync } from 'node:fs';

import { AgentDefinitionSchema } from './types/orchestrator.types.js';
import type {
  AgentDefinition,
  AgentRole,
  AgentPermissionProfile,
} from './types/orchestrator.types.js';
import { BUILT_IN_AGENT_DEFINITIONS } from './types/orchestrator.types.js';

// ─── Matching helpers ─────────────────────────────────────────────────────────

/**
 * Compute a capability match score between a definition and a required
 * capability set. Higher score = better match.
 * Score = number of required capabilities matched.
 */
function capabilityScore(
  definition: AgentDefinition,
  requiredCapabilities: string[],
): number {
  const defCaps = new Set(definition.capabilities.map((c) => c.toLowerCase()));
  return requiredCapabilities.filter((cap) =>
    defCaps.has(cap.toLowerCase()),
  ).length;
}

// ─── AgentRegistry ────────────────────────────────────────────────────────────

export class AgentRegistry {
  private readonly _definitions = new Map<string, AgentDefinition>();
  private _persistPath: string | null = null;

  constructor() {
    // Always load built-ins first
    for (const def of BUILT_IN_AGENT_DEFINITIONS) {
      this._definitions.set(def.name, def);
    }
  }

  // ─── Registration ──────────────────────────────────────────────────────────

  /**
   * Register a custom agent definition.
   * If a definition with the same name already exists, it is overwritten.
   * Throws if the definition fails Zod schema validation.
   */
  register(definition: AgentDefinition): void {
    const result = AgentDefinitionSchema.safeParse(definition);
    if (!result.success) {
      throw new Error(
        `AgentRegistry: invalid definition "${definition.name}": ` +
          result.error.issues.map((i) => `${i.path.join('.')}: ${i.message}`).join('; '),
      );
    }
    this._definitions.set(definition.name, definition);
  }

  /**
   * Register multiple definitions at once.
   */
  registerAll(definitions: AgentDefinition[]): void {
    for (const def of definitions) {
      this.register(def);
    }
  }

  /**
   * Remove a definition by name.
   * Returns true if found and removed, false otherwise.
   * Built-in definitions can be removed (allowing full override).
   */
  unregister(name: string): boolean {
    return this._definitions.delete(name);
  }

  // ─── Retrieval ─────────────────────────────────────────────────────────────

  /**
   * Get a definition by exact name.
   * Returns undefined if not found.
   */
  get(name: string): AgentDefinition | undefined {
    return this._definitions.get(name);
  }

  /**
   * Get a definition by name, falling back to the first definition
   * matching the given role if the name is not found.
   */
  getOrFallback(name: string, role: AgentRole): AgentDefinition | undefined {
    return this._definitions.get(name) ?? this.getByRole(role);
  }

  /**
   * Get the first definition matching a given role.
   */
  getByRole(role: AgentRole): AgentDefinition | undefined {
    for (const def of this._definitions.values()) {
      if (def.role === role) return def;
    }
    return undefined;
  }

  /**
   * Get all definitions matching a given role (may be multiple).
   */
  getAllByRole(role: AgentRole): AgentDefinition[] {
    return [...this._definitions.values()].filter((d) => d.role === role);
  }

  /**
   * List all registered definitions.
   */
  listAll(): AgentDefinition[] {
    return [...this._definitions.values()];
  }

  /**
   * List all names currently registered.
   */
  listNames(): string[] {
    return [...this._definitions.keys()];
  }

  /**
   * Check if a definition with the given name is registered.
   */
  has(name: string): boolean {
    return this._definitions.has(name);
  }

  // ─── Capability-based discovery ─────────────────────────────────────────────

  /**
   * Find definitions that match ALL of the required capabilities.
   * Returns definitions sorted by match score (best first).
   */
  findByCapabilities(requiredCapabilities: string[]): AgentDefinition[] {
    if (requiredCapabilities.length === 0) return this.listAll();

    return [...this._definitions.values()]
      .map((def) => ({
        def,
        score: capabilityScore(def, requiredCapabilities),
      }))
      .filter(({ score }) => score > 0)
      .sort((a, b) => b.score - a.score)
      .map(({ def }) => def);
  }

  /**
   * Find the single best-matching definition for a set of required capabilities.
   * Returns undefined if nothing matches.
   */
  bestMatch(requiredCapabilities: string[]): AgentDefinition | undefined {
    return this.findByCapabilities(requiredCapabilities)[0];
  }

  /**
   * Find all definitions that support parallel execution.
   */
  getParallelSafe(): AgentDefinition[] {
    return [...this._definitions.values()].filter((d) => d.supportsParallel);
  }

  // ─── Prompt generation ─────────────────────────────────────────────────────

  /**
   * Build the complete system prompt suffix for a member at spawn time.
   * Merges the role definition's suffix with member-specific overrides.
   *
   * The suffix is appended by AgentSpawner to the base system prompt
   * from packages/core TurnAssembler.
   */
  buildSystemPromptSuffix(
    definitionName: string,
    opts: {
      teamName: string;
      memberName: string;
      fileScope: string[];
      customPrompt?: string;
      teamGoal?: string;
    },
  ): string {
    const def = this._definitions.get(definitionName);

    const sections: string[] = [
      `## Team Context`,
      `You are **${opts.memberName}** in team **${opts.teamName}**.`,
    ];

    if (opts.teamGoal) {
      sections.push(`**Team goal:** ${opts.teamGoal}`);
    }

    if (opts.fileScope.length > 0) {
      sections.push(
        '',
        '## Your File Scope',
        'You may ONLY create or modify files matching these paths:',
        ...opts.fileScope.map((p) => `- \`${p}\``),
        '',
        '⚠ Writing outside your file scope will be blocked by the file lock system.',
      );
    }

    if (def?.systemPromptSuffix) {
      sections.push('', '## Role Instructions', def.systemPromptSuffix);
    }

    if (opts.customPrompt) {
      sections.push('', '## Additional Instructions', opts.customPrompt);
    }

    sections.push(
      '',
      '## Communication',
      'To send a message to another teammate: `[MESSAGE: teammate-name] Your message here`',
      'To broadcast to all teammates: `[BROADCAST] Your message here`',
      'To log an observation: `[OBSERVE:N: Your observation]` (N = 1-5 importance)',
    );

    return sections.join('\n');
  }

  // ─── Tool allowlist resolution ─────────────────────────────────────────────

  /**
   * Compute the effective tool list for a definition.
   * Returns null if the definition has no restrictions (all tools allowed).
   * Returns a filtered list if allowedTools or deniedTools are set.
   */
  resolveToolList(
    definitionName: string,
    availableTools: string[],
  ): string[] | null {
    const def = this._definitions.get(definitionName);
    if (!def) return null;

    if (def.allowedTools.length === 0 && def.deniedTools.length === 0) {
      return null; // No restrictions
    }

    let tools = availableTools;

    if (def.allowedTools.length > 0) {
      const allowed = new Set(def.allowedTools);
      tools = tools.filter((t) => allowed.has(t));
    }

    if (def.deniedTools.length > 0) {
      const denied = new Set(def.deniedTools);
      tools = tools.filter((t) => !denied.has(t));
    }

    return tools;
  }

  // ─── Persistence (optional) ────────────────────────────────────────────────

  /**
   * Load custom definitions from a JSON file.
   * Built-in definitions are NOT stored in the file (they're always in code).
   * Custom definitions in the file override built-ins with the same name.
   */
  async loadFromFile(filePath: string): Promise<number> {
    this._persistPath = filePath;

    if (!existsSync(filePath)) return 0;

    try {
      const raw = await readFile(filePath, 'utf8');
      const parsed = JSON.parse(raw) as AgentDefinition[];

      if (!Array.isArray(parsed)) {
        throw new Error('Agent definitions file must contain a JSON array.');
      }

      let loaded = 0;
      for (const def of parsed) {
        try {
          this.register(def);
          loaded++;
        } catch (err) {
          console.warn(
            `[AgentRegistry] Skipping invalid definition "${def?.name}": ` +
              `${err instanceof Error ? err.message : String(err)}`,
          );
        }
      }
      return loaded;
    } catch (err) {
      throw new Error(
        `AgentRegistry: failed to load definitions from "${filePath}": ` +
          `${err instanceof Error ? err.message : String(err)}`,
      );
    }
  }

  /**
   * Persist only CUSTOM definitions (non-built-in) to the configured file.
   * Does nothing if no persist path is set.
   */
  async saveToFile(filePath?: string): Promise<void> {
    const path = filePath ?? this._persistPath;
    if (!path) return;

    // Only persist non-built-in definitions
    const builtInNames = new Set(BUILT_IN_AGENT_DEFINITIONS.map((d) => d.name));
    const customDefs = [...this._definitions.values()].filter(
      (d) => !builtInNames.has(d.name),
    );

    await mkdir(dirname(path), { recursive: true });
    await writeFile(path, JSON.stringify(customDefs, null, 2), 'utf8');
  }

  // ─── Reporting helpers ─────────────────────────────────────────────────────

  /**
   * Produce a Markdown summary of all registered definitions.
   * Used by OrchestratorReporter.
   */
  toMarkdown(): string {
    const lines: string[] = [
      '## Agent Registry',
      '',
      `${this._definitions.size} definitions registered`,
      '',
      '| Name | Role | Capabilities | Parallel | Context |',
      '|------|------|-------------|----------|---------|',
    ];

    for (const def of this._definitions.values()) {
      const caps = def.capabilities.slice(0, 4).join(', ') +
        (def.capabilities.length > 4 ? '…' : '');
      const ctx = def.maxContextTokens
        ? `${Math.round(def.maxContextTokens / 1000)}k`
        : '—';
      lines.push(
        `| **${def.name}** | ${def.role} | ${caps} | ${def.supportsParallel ? '✅' : '❌'} | ${ctx} |`,
      );
    }

    return lines.join('\n');
  }
}
src/AgentSpawner.ts
TypeScript

/**
 * AgentSpawner — creates sub-agent sessions for @locoworker/orchestrator
 *
 * AgentSpawner is the LocoWorker implementation of the Claude Code
 * spawnTeam / spawn primitive — but rather than launching separate
 * OS processes or tmux panes, it creates new sessions in the
 * packages/core SessionManager, each with its own isolated context
 * window, permission set, and tool allowlist.
 *
 * This "in-process" spawning model is deliberately chosen:
 *  - No external process management required
 *  - Clean integration with the existing EventBus and HooksRegistry
 *  - File-scope enforcement is done by FileLockManager (DB-based),
 *    not OS-level process isolation
 *  - Context windows are isolated (separate session → separate history)
 *  - Sessions can be torn down cleanly on team completion
 *
 * For workloads requiring true OS-level isolation (different git worktrees,
 * separate file system namespaces), the AgentSpawner can optionally
 * resolve a worktree path per member — this is a future extension point
 * noted in the architecture but not implemented in this pass.
 *
 * Responsibilities:
 *  - Build member session config from AgentDefinition + SpawnRequest
 *  - Create a TeamMember record in OrchestratorStore
 *  - Resolve permission profile → core PermissionTier mapping
 *  - Inject the role-specific system prompt suffix (from AgentRegistry)
 *  - Track spawn latency and session IDs
 *  - Handle spawn failures gracefully (member → 'failed' status)
 *  - Provide a teardown method for graceful member shutdown
 *
 * Core integration:
 *  - Depends on packages/core SessionManager (via interface shim)
 *    for session creation.
 *  - The core SessionManager is injected at construction time
 *    (not imported statically) for the same decoupling pattern
 *    used across all LocoWorker packages.
 */

import type { OrchestratorStore } from './OrchestratorStore.js';
import type { AgentRegistry } from './AgentRegistry.js';
import type {
  TeamMember,
  SpawnRequest,
  SpawnResult,
  AgentPermissionProfile,
  AgentRole,
} from './types/orchestrator.types.js';

// ─── Core interface shims ─────────────────────────────────────────────────────
// Minimal shapes we need from packages/core so orchestrator has no
// hard compile-time dependency on it.

export interface CoreModelConfig {
  providerId: string;
  modelId: string;
  maxTokens?: number;
  temperature?: number;
}

export interface CoreSessionConfig {
  workingDirectory: string;
  modelConfig: CoreModelConfig;
  maxTurns?: number;
  permissionTier?: string;        // maps from AgentPermissionProfile
  customSystemPromptSuffix?: string;
  allowedTools?: string[];
  sessionId?: string;
}

export interface CoreSession {
  id: string;
  modelConfig: CoreModelConfig;
  workingDirectory: string;
  createdAt: string;
}

export interface SessionManagerLike {
  create(config: CoreSessionConfig): Promise<CoreSession>;
  destroy(sessionId: string): Promise<void>;
  get(sessionId: string): CoreSession | undefined;
}

// ─── Permission profile → core tier mapping ───────────────────────────────────

const PERMISSION_PROFILE_TO_TIER: Record<AgentPermissionProfile, string> = {
  read_only:   'READ_ONLY',
  write_local: 'WRITE_LOCAL',
  standard:    'SHELL',        // standard includes read+write+shell
  full:        'DANGEROUS',    // supervisor gets full access
  custom:      'WRITE_LOCAL',  // custom falls back to write_local; caller adjusts
};

// ─── Option types ─────────────────────────────────────────────────────────────

export interface AgentSpawnerOptions {
  /** Default working directory for spawned sessions. */
  workingDirectory: string;

  /** Default model config used when member doesn't specify a model. */
  defaultModelConfig: CoreModelConfig;

  /** If true, log spawn events to console. Default: false. */
  verbose?: boolean;
}

export interface TeardownResult {
  memberId: string;
  memberName: string;
  sessionId: string;
  success: boolean;
  error?: string;
}

// ─── AgentSpawner ─────────────────────────────────────────────────────────────

export class AgentSpawner {
  private readonly _verbose: boolean;

  constructor(
    private readonly store: OrchestratorStore,
    private readonly registry: AgentRegistry,
    private readonly sessionManager: SessionManagerLike,
    private readonly options: AgentSpawnerOptions,
  ) {
    this._verbose = options.verbose ?? false;
  }

  // ─── Spawn ─────────────────────────────────────────────────────────────────

  /**
   * Spawn a single team member — creates a session and persists the
   * member record in OrchestratorStore.
   *
   * Returns a SpawnResult with the created member and session ID.
   * On failure, transitions the member to 'failed' status and returns
   * a SpawnResult with success=false.
   */
  async spawn(request: SpawnRequest): Promise<SpawnResult> {
    const start = Date.now();

    // 1) Create the member record (status: 'spawning')
    const member = this.store.createMember({
      teamId: request.teamId,
      name: request.memberName,
      role: request.role,
      status: 'spawning',
      permissionProfile: request.permissionProfile ?? 'standard',
      fileScope: request.fileScope ?? [],
      customPrompt: request.customPrompt,
      modelId: request.modelId,
      wave: request.wave ?? 0,
      dependsOn: request.dependsOn ?? [],
      assignedTaskIds: [],
    });

    this._log(`Spawning member "${member.name}" [${member.role}] wave=${member.wave}`);

    try {
      // 2) Look up the agent definition for this role
      const defName = request.definition?.name
        ?? this.registry.getByRole(request.role)?.name
        ?? request.role;

      // 3) Build the team context for the system prompt suffix
      const team = this.store.getTeam(request.teamId);
      const promptSuffix = this.registry.buildSystemPromptSuffix(defName, {
        teamName: team?.name ?? request.teamId,
        memberName: member.name,
        fileScope: member.fileScope,
        customPrompt: request.customPrompt,
        teamGoal: team?.goal,
      });

      // 4) Resolve tool list
      const def = request.definition ?? this.registry.getOrFallback(defName, request.role);
      const allowedTools = def
        ? (this.registry.resolveToolList(defName, this._getDefaultTools()) ?? undefined)
        : undefined;

      // 5) Build core session config
      const permissionTier = PERMISSION_PROFILE_TO_TIER[
        request.permissionProfile ?? 'standard'
      ];

      const sessionConfig: CoreSessionConfig = {
        workingDirectory: this.options.workingDirectory,
        modelConfig: {
          ...this.options.defaultModelConfig,
          ...(request.modelId ? { modelId: request.modelId } : {}),
          ...(def?.maxContextTokens ? { maxTokens: def.maxContextTokens } : {}),
        },
        permissionTier,
        customSystemPromptSuffix: promptSuffix,
        allowedTools,
      };

      // 6) Create the session via SessionManager
      const session = await this.sessionManager.create(sessionConfig);

      // 7) Transition member to 'idle' with the session ID
      const updatedMember = this.store.transitionMember(member.id, 'idle', {
        sessionId: session.id,
      });

      const duration = Date.now() - start;
      this._log(
        `Spawned "${member.name}" → session ${session.id} (${duration}ms)`,
      );

      return {
        member: updatedMember ?? member,
        sessionId: session.id,
        success: true,
      };
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : String(err);
      this._log(`Spawn failed for "${member.name}": ${errorMessage}`);

      // Transition member to failed
      const failedMember = this.store.transitionMember(member.id, 'failed', {
        errorMessage,
      });

      return {
        member: failedMember ?? member,
        sessionId: '',
        success: false,
        error: errorMessage,
      };
    }
  }

  /**
   * Spawn multiple members for a wave, optionally in parallel.
   * Returns results for all members (including failures).
   */
  async spawnWave(
    requests: SpawnRequest[],
    parallel = true,
  ): Promise<SpawnResult[]> {
    if (requests.length === 0) return [];

    this._log(
      `Spawning wave of ${requests.length} members (${parallel ? 'parallel' : 'sequential'})`,
    );

    if (parallel) {
      const settled = await Promise.allSettled(
        requests.map((req) => this.spawn(req)),
      );
      return settled.map((r, i) => {
        if (r.status === 'fulfilled') return r.value;
        return {
          member: {
            id: '',
            teamId: requests[i].teamId,
            name: requests[i].memberName,
            role: requests[i].role,
            status: 'failed' as const,
            permissionProfile: requests[i].permissionProfile ?? 'standard',
            fileScope: requests[i].fileScope ?? [],
            wave: requests[i].wave ?? 0,
            dependsOn: requests[i].dependsOn ?? [],
            assignedTaskIds: [],
            tokensUsed: 0,
            costUsd: 0,
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString(),
          } as TeamMember,
          sessionId: '',
          success: false,
          error: r.reason instanceof Error ? r.reason.message : String(r.reason),
        };
      });
    } else {
      const results: SpawnResult[] = [];
      for (const req of requests) {
        results.push(await this.spawn(req));
      }
      return results;
    }
  }

  // ─── Teardown ──────────────────────────────────────────────────────────────

  /**
   * Gracefully tear down a member — destroys its session and transitions
   * it to 'done' (or 'failed' if teardown errors).
   */
  async teardown(
    memberId: string,
    reason: 'done' | 'failed' | 'evicted' = 'done',
  ): Promise<TeardownResult> {
    const member = this.store.getMember(memberId);
    if (!member) {
      return {
        memberId,
        memberName: memberId,
        sessionId: '',
        success: false,
        error: 'Member not found',
      };
    }

    // Release all file locks held by this member
    this.store.releaseAllLocksForMember(member.teamId, memberId);

    // Destroy the session
    if (member.sessionId) {
      try {
        await this.sessionManager.destroy(member.sessionId);
      } catch (err) {
        // Log but don't fail teardown
        this._log(
          `Session destroy error for "${member.name}": ` +
            `${err instanceof Error ? err.message : String(err)}`,
        );
      }
    }

    // Transition member
    this.store.transitionMember(memberId, reason);

    this._log(`Torn down member "${member.name}" (reason: ${reason})`);

    return {
      memberId,
      memberName: member.name,
      sessionId: member.sessionId ?? '',
      success: true,
    };
  }

  /**
   * Tear down all members of a team (for team completion/abort).
   */
  async teardownTeam(teamId: string): Promise<TeardownResult[]> {
    const memberRows = this.store.queryMembers({
      teamId,
      status: ['spawning', 'idle', 'working', 'waiting', 'reviewing'],
    });

    const results = await Promise.allSettled(
      memberRows.map((row) =>
        this.teardown(this.store._rowToMember(row).id, 'done'),
      ),
    );

    return results.map((r) => {
      if (r.status === 'fulfilled') return r.value;
      return {
        memberId: '',
        memberName: 'unknown',
        sessionId: '',
        success: false,
        error: r.reason instanceof Error ? r.reason.message : String(r.reason),
      };
    });
  }

  // ─── Status helpers ────────────────────────────────────────────────────────

  /**
   * Check if a member's underlying session is still alive.
   */
  isSessionAlive(memberId: string): boolean {
    const member = this.store.getMember(memberId);
    if (!member?.sessionId) return false;
    return this.sessionManager.get(member.sessionId) !== undefined;
  }

  /**
   * Get the live session for a member (or undefined).
   */
  getSession(memberId: string): CoreSession | undefined {
    const member = this.store.getMember(memberId);
    if (!member?.sessionId) return undefined;
    return this.sessionManager.get(member.sessionId);
  }

  // ─── Internal helpers ──────────────────────────────────────────────────────

  /**
   * Default tool list — all core tools by name.
   * AgentRegistry.resolveToolList() filters this per definition.
   *
   * In production this would be populated from packages/core ToolRegistry.
   * Here it lists the canonical tool names established across the passes.
   */
  private _getDefaultTools(): string[] {
    return [
      'read_file',
      'write_file',
      'list_directory',
      'search_files',
      'run_shell',
      'wiki_read',
      'wiki_write',
      'graph_query',
      'memory_read',
      'memory_write',
      'task_create',
      'task_complete',
      'observe',
    ];
  }

  private _log(msg: string): void {
    if (this._verbose) {
      console.log(`[AgentSpawner] ${msg}`);
    }
  }
}
src/MessageRouter.ts
TypeScript

/**
 * MessageRouter — peer-to-peer and broadcast messaging for @locoworker/orchestrator
 *
 * The LocoWorker re-implementation of Claude Code's TeammateTool:
 *   TeammateTool.write(target, content)    → MessageRouter.send(direct)
 *   TeammateTool.broadcast(content)        → MessageRouter.broadcast()
 *
 * Additionally implements:
 *   - Supervisor-directed messages (always routed to the team lead)
 *   - System messages (engine-generated notifications)
 *   - [MESSAGE: name] and [BROADCAST] marker extraction from agent text
 *   - Message delivery via EventBus (for live UI updates)
 *   - Unread inbox queries for each member
 *   - Structured payload routing (plan_approval_request, rollback_required, etc.)
 *
 * Design:
 *  - MessageRouter is a thin coordination layer over OrchestratorStore.
 *  - It never processes message content — that is the agent's job.
 *  - Message routing is synchronous (DB write) but EventBus emission is async.
 *  - MessageRouter is safe to use from any class (read and write operations).
 *
 * Marker syntax (agent messages):
 *   [MESSAGE: teammate-name] Your message content here
 *   [BROADCAST] Content to send to all teammates
 *
 * These markers are extracted by MessageRouter.extractMarkers() and
 * processed by OrchestratorEngine's beforeTurn hook.
 */

import type { OrchestratorStore } from './OrchestratorStore.js';
import type {
  TeamMessage,
  MessageRow,
  MessageChannel,
  TeamMember,
} from './types/orchestrator.types.js';

// ─── Marker regexes ───────────────────────────────────────────────────────────

/** [MESSAGE: member-name] content */
const MESSAGE_MARKER_REGEX =
  /\[MESSAGE:\s*([a-z0-9_-]+)\]\s+([\s\S]*?)(?=\[MESSAGE:|$)/gi;

/** [BROADCAST] content */
const BROADCAST_MARKER_REGEX =
  /\[BROADCAST\]\s+([\s\S]*?)(?=\[MESSAGE:|\[BROADCAST\]|$)/gi;

// ─── Core interface shims ─────────────────────────────────────────────────────

interface EventBusLike {
  emit(event: { type: string; [key: string]: unknown }): Promise<void>;
}

// ─── Option types ─────────────────────────────────────────────────────────────

export interface SendOptions {
  /** Arbitrary structured payload for machine-readable coordination. */
  payload?: Record<string, unknown>;

  /** Mark as read immediately (e.g., system self-notifications). */
  markReadImmediately?: boolean;
}

export interface ExtractedMessage {
  channel: 'direct' | 'broadcast';
  targetName?: string;   // for direct messages
  content: string;
}

export interface RouterStats {
  totalMessages: number;
  unreadMessages: number;
  broadcastMessages: number;
  directMessages: number;
  systemMessages: number;
}

// ─── MessageRouter ────────────────────────────────────────────────────────────

export class MessageRouter {
  private _eventBus: EventBusLike | null = null;

  constructor(private readonly store: OrchestratorStore) {}

  // ─── EventBus attachment ───────────────────────────────────────────────────

  /**
   * Attach an EventBus instance so message sends trigger live events.
   * Optional — the router works without EventBus (just DB writes).
   */
  attachEventBus(eventBus: EventBusLike): this {
    this._eventBus = eventBus;
    return this;
  }

  // ─── Sending ───────────────────────────────────────────────────────────────

  /**
   * Send a direct message from one member to another.
   * Equivalent to TeammateTool.write(targetName, content).
   */
  send(
    teamId: string,
    from: TeamMember,
    toMember: TeamMember,
    content: string,
    options: SendOptions = {},
  ): TeamMessage {
    return this._doSend({
      teamId,
      fromMemberId: from.id,
      fromMemberName: from.name,
      toMemberId: toMember.id,
      toMemberName: toMember.name,
      channel: 'direct',
      content: content.trim(),
      payload: options.payload,
    }, options.markReadImmediately ?? false);
  }

  /**
   * Send a direct message from a member to the supervisor.
   * Equivalent to the supervisor-escalation pattern.
   */
  sendToSupervisor(
    teamId: string,
    from: TeamMember,
    supervisorMember: TeamMember,
    content: string,
    options: SendOptions = {},
  ): TeamMessage {
    return this._doSend({
      teamId,
      fromMemberId: from.id,
      fromMemberName: from.name,
      toMemberId: supervisorMember.id,
      toMemberName: supervisorMember.name,
      channel: 'supervisor',
      content: content.trim(),
      payload: options.payload,
    }, options.markReadImmediately ?? false);
  }

  /**
   * Broadcast a message to all team members.
   * Equivalent to TeammateTool.broadcast(content).
   */
  broadcast(
    teamId: string,
    from: TeamMember | null,
    content: string,
    options: SendOptions = {},
  ): TeamMessage {
    return this._doSend({
      teamId,
      fromMemberId: from?.id,
      fromMemberName: from?.name,
      toMemberId: undefined,
      toMemberName: undefined,
      channel: 'broadcast',
      content: content.trim(),
      payload: options.payload,
    }, options.markReadImmediately ?? false);
  }

  /**
   * Send a system message (engine-generated, no sender member).
   */
  sendSystem(
    teamId: string,
    content: string,
    options: SendOptions = {},
  ): TeamMessage {
    return this._doSend({
      teamId,
      fromMemberId: undefined,
      fromMemberName: 'system',
      toMemberId: undefined,
      toMemberName: undefined,
      channel: 'system',
      content: content.trim(),
      payload: options.payload,
    }, options.markReadImmediately ?? true);
  }

  /**
   * Send a typed coordination signal as a structured payload message.
   * Used for machine-readable signals like plan_approval_request.
   */
  sendSignal(
    teamId: string,
    from: TeamMember | null,
    target: TeamMember | null,
    signalType: string,
    payload: Record<string, unknown>,
    humanReadableContent: string,
  ): TeamMessage {
    return this._doSend({
      teamId,
      fromMemberId: from?.id,
      fromMemberName: from?.name ?? 'system',
      toMemberId: target?.id,
      toMemberName: target?.name,
      channel: target ? 'direct' : 'broadcast',
      content: humanReadableContent,
      payload: { type: signalType, ...payload },
    }, false);
  }

  // ─── Inbox ─────────────────────────────────────────────────────────────────

  /**
   * Get unread messages for a member.
   * Includes: direct messages to this member + broadcast + system messages.
   */
  getInbox(teamId: string, memberId: string, limit = 50): MessageRow[] {
    return this.store.getMessages(teamId, {
      memberId,
      unreadOnly: true,
      limit,
    });
  }

  /**
   * Get all messages for a team (full log).
   */
  getFullLog(teamId: string, limit = 500): MessageRow[] {
    return this.store.getMessages(teamId, { limit });
  }

  /**
   * Mark a set of messages as read.
   */
  markRead(messageIds: string[]): number {
    return this.store.markMessagesRead(messageIds);
  }

  /**
   * Mark all unread inbox messages for a member as read.
   */
  drainInbox(teamId: string, memberId: string): number {
    const inbox = this.getInbox(teamId, memberId, 1000);
    if (inbox.length === 0) return 0;
    return this.markRead(inbox.map((m) => m.id));
  }

  // ─── Marker extraction ─────────────────────────────────────────────────────

  /**
   * Extract [MESSAGE: name] and [BROADCAST] markers from agent output text.
   * Called by OrchestratorEngine's beforeTurn hook to process outgoing
   * messages embedded in agent responses.
   *
   * Returns an array of extracted messages (not yet persisted — caller
   * must resolve member names and call send/broadcast).
   */
  extractMarkers(text: string): ExtractedMessage[] {
    const messages: ExtractedMessage[] = [];

    // Extract [MESSAGE: name] markers
    MESSAGE_MARKER_REGEX.lastIndex = 0;
    let match: RegExpExecArray | null;
    while ((match = MESSAGE_MARKER_REGEX.exec(text)) !== null) {
      const targetName = match[1].trim().toLowerCase();
      const content = match[2].trim();
      if (targetName && content) {
        messages.push({ channel: 'direct', targetName, content });
      }
    }

    // Extract [BROADCAST] markers
    BROADCAST_MARKER_REGEX.lastIndex = 0;
    while ((match = BROADCAST_MARKER_REGEX.exec(text)) !== null) {
      const content = match[1].trim();
      if (content) {
        messages.push({ channel: 'broadcast', content });
      }
    }

    return messages;
  }

  /**
   * Process extracted markers against a live member map and persist them.
   * `senderMember` is the member whose output contained the markers.
   * `membersByName` maps lowercase member name → TeamMember.
   */
  processExtractedMarkers(
    teamId: string,
    senderMember: TeamMember,
    extracted: ExtractedMessage[],
    membersByName: Map<string, TeamMember>,
    supervisorMember?: TeamMember,
  ): TeamMessage[] {
    const sent: TeamMessage[] = [];

    for (const msg of extracted) {
      if (msg.channel === 'broadcast') {
        sent.push(this.broadcast(teamId, senderMember, msg.content));
      } else if (msg.channel === 'direct' && msg.targetName) {
        const target = membersByName.get(msg.targetName.toLowerCase());
        if (target) {
          sent.push(this.send(teamId, senderMember, target, msg.content));
        } else if (
          msg.targetName === 'supervisor' &&
          supervisorMember
        ) {
          sent.push(
            this.sendToSupervisor(teamId, senderMember, supervisorMember, msg.content),
          );
        } else {
          // Target not found — send as system warning
          this.sendSystem(
            teamId,
            `⚠ Member "${senderMember.name}" tried to message unknown member "${msg.targetName}". Message was dropped.`,
          );
        }
      }
    }

    return sent;
  }

  // ─── Stats ─────────────────────────────────────────────────────────────────

  /**
   * Get message statistics for a team.
   */
  getStats(teamId: string): RouterStats {
    interface CountRow { channel: MessageChannel; count: number; unread: number }
    const rows = this.store.db
      .prepare<[string], CountRow>(/* sql */`
        SELECT channel,
               COUNT(*) as count,
               SUM(CASE WHEN read = 0 THEN 1 ELSE 0 END) as unread
        FROM orch_messages
        WHERE team_id = ?
        GROUP BY channel
      `)
      .all(teamId);

    let totalMessages = 0;
    let unreadMessages = 0;
    let broadcastMessages = 0;
    let directMessages = 0;
    let systemMessages = 0;

    for (const row of rows) {
      totalMessages += row.count;
      unreadMessages += row.unread;
      if (row.channel === 'broadcast') broadcastMessages += row.count;
      else if (row.channel === 'direct' || row.channel === 'supervisor') directMessages += row.count;
      else if (row.channel === 'system') systemMessages += row.count;
    }

    return { totalMessages, unreadMessages, broadcastMessages, directMessages, systemMessages };
  }

  // ─── Internal helpers ──────────────────────────────────────────────────────

  private _doSend(
    input: Omit<TeamMessage, 'id' | 'sentAt' | 'read'>,
    markReadImmediately: boolean,
  ): TeamMessage {
    const message = this.store.sendMessage(input);

    if (markReadImmediately) {
      this.store.markMessagesRead([message.id]);
    }

    // Emit to EventBus (fire-and-forget)
    void this._emitMessageEvent(message);

    return message;
  }

  private async _emitMessageEvent(message: TeamMessage): Promise<void> {
    if (!this._eventBus) return;
    try {
      await this._eventBus.emit({
        type: 'orchestrator_message',
        teamId: message.teamId,
        messageId: message.id,
        channel: message.channel,
        fromMemberName: message.fromMemberName,
        toMemberName: message.toMemberName,
        content: message.content.slice(0, 200), // truncate for event payload
        sentAt: message.sentAt,
      });
    } catch {
      // Never crash on EventBus failure
    }
  }
}
Pass 7 — Part 1 Complete ✅
What was generated
File	Purpose	Lines (approx)
package.json	Package config: better-sqlite3 + zod; peers: core(required), kairos/memory/wiki (optional)	48
tsconfig.json	Compiler config, all monorepo path aliases	24
src/types/orchestrator.types.ts	Full type ABI: teams, members, shared tasks, messages, quality gates, results, file locks, agent definitions, Zod schemas, built-in role catalogue (8 roles), constants, marker regexes	~520
src/OrchestratorStore.ts	SQLite persistence — 7-table schema (teams/members/messages/shared_tasks/file_locks/results/meta), WAL pragmas, full CRUD, dependency-aware task queries, file lock conflict detection + expiry, auto-skip blocked tasks, stats	~580
src/AgentRegistry.ts	In-memory capability registry — registration, validation, role/capability lookup, best-match scoring (Jaccard on tags), tool allowlist resolution, system prompt generation, optional JSON file persistence, Markdown reporting	~310
src/AgentSpawner.ts	Sub-agent session factory — maps permission profiles to core tiers, builds role-scoped system prompts, creates sessions via SessionManagerLike shim, sequential + parallel wave spawning, graceful teardown with file lock release	~280
src/MessageRouter.ts	Peer-to-peer + broadcast messaging — send/broadcast/sendToSupervisor/sendSystem/sendSignal, [MESSAGE: name] + [BROADCAST] marker extraction + processing, inbox queries, EventBus emission, routing stats	~290
Part 2 preview (ready to generate on request)
File	Purpose
src/DelegationPlanner.ts	Decomposes a natural-language goal into a typed DelegationPlan with waves, member assignments, dependency graph, and quality gates
src/FileLockManager.ts	File-scope lock enforcement — scope overlap detection, lock acquisition queue, conflict resolution, TTL expiry, glob matching
src/ResultAggregator.ts	Collects sub-agent AgentResult records, detects file conflicts, merges outputs, produces a structured synthesis report
src/OrchestratorEngine.ts	Main coordination loop — drives all phases (planning → spawning → running → gating → completing), EventBus integration, schedule-wave execution, gate enforcement
src/OrchestratorReporter.ts	ORCHESTRATOR_REPORT.md generator — team timeline, member status table, shared task list, message log excerpt, conflict report, cost summary
src/index.ts	Complete barrel export
src/tests/OrchestratorStore.test.ts	Full bun test suite
src/tests/AgentRegistry.test.ts	Full bun test suite
src/tests/AgentSpawner.test.ts	Full bun test suite
src/tests/MessageRouter.test.ts	Full bun test suite
Key design decisions & cross-package consistency
Decision	Rationale
In-process spawning (no tmux/worktrees)	Keeps the implementation pure TypeScript with no OS-level dependencies; file-scope isolation is enforced by FileLockManager (DB-based), not process isolation
8-role built-in catalogue maps to Claude Code's subagent types	supervisor, architect, implementer, test_runner, security_reviewer, doc_writer, researcher, reviewer — matches real-world Agent Teams patterns from the docs
DB column naming follows monorepo convention	tags_csv, session_id, created_at/updated_at, depends_on_json, file_scope_json — identical to all prior packages
7-table schema (messages + results are append-only)	Mirrors wiki_history and kairos_observations — never update/delete audit-critical tables
SessionManagerLike interface shim	No hard compile-time dep on packages/core — same decoupling pattern as WikiSyncAgent, KairosAgent
[MESSAGE: name]/[BROADCAST] marker syntax	Consistent with [REMEMBER:], [OBSERVE:], [WIKI:] — agents use a unified [VERB: ...] control surface across all packages
Capability scoring (Jaccard-style tag overlap)	Same pattern as WikiLinker's slug candidate suggestion — consistent "similarity = set intersection" approach
Quality gates model	Matches Claude Code's approveShutdown/requestShutdown/plan_approval_request signals, generalised to support test_pass, security_clear, and auto gate types
DELEGATE_MARKER_REGEX in types	[DELEGATE: role "task"] gives the agent a direct way to spawn sub-agents from a message — same control-surface philosophy as all other marker types






Pass 7 — packages/orchestrator (Part 2 of 2)
Continuing from Part 1. This pass completes @locoworker/orchestrator with the delegation planner, file lock manager, result aggregator, the main orchestration engine, markdown reporter, barrel export, and all four fully runnable bun test test suites.

src/DelegationPlanner.ts
TypeScript

/**
 * DelegationPlanner — goal decomposition into typed delegation plan
 *                     for @locoworker/orchestrator
 *
 * DelegationPlanner is the LocoWorker implementation of the supervisor's
 * "planning phase" — the moment when a high-level goal is decomposed into:
 *  - A set of concrete shared tasks
 *  - A topology (sequential | parallel | hierarchical | wave)
 *  - Member role assignments per wave
 *  - A dependency graph (which tasks block which other tasks)
 *  - Quality gates (plan approval, output review, test pass, security clear)
 *
 * Two planning strategies are supported:
 *  1) LLM-based planning (supervisor agent generates the plan)
 *  2) Template-based planning (pre-defined plan templates for common workflows)
 *
 * This pass implements the template-based planner as a reference implementation.
 * LLM-based planning is available as an extension point (parse structured plan
 * output from the supervisor agent's first turn).
 *
 * Responsibilities:
 *  - Generate a DelegationPlan from a goal string + topology hint
 *  - Assign appropriate agent roles based on goal keywords
 *  - Build a dependency graph (e.g., architect → implementer → test_runner)
 *  - Estimate token cost and USD cost per wave
 *  - Insert quality gates at appropriate lifecycle points
 *  - Validate the plan (no circular dependencies, file scope overlaps)
 *  - Serialize the plan to JSON for approval/storage
 *
 * Design:
 *  - DelegationPlanner is stateless (no DB dependency).
 *  - It uses AgentRegistry to look up role definitions and capabilities.
 *  - Plans are validated before being returned.
 *  - The engine (OrchestratorEngine) consumes the plan and executes it.
 */

import type { AgentRegistry } from './AgentRegistry.js';
import type {
  DelegationPlan,
  DelegationWave,
  QualityGate,
  TeamTopology,
  AgentRole,
  AgentDefinition,
} from './types/orchestrator.types.js';

// ─── Plan templates ───────────────────────────────────────────────────────────

interface PlanTemplate {
  name: string;
  keywords: string[];
  topology: TeamTopology;
  waves: Array<{
    roles: AgentRole[];
    fileScopes: Record<AgentRole, string[]>;
    tasks: Array<{
      title: string;
      description: string;
      role: AgentRole;
      filePaths: string[];
      parallelSafe: boolean;
      blockedBy?: string[];
    }>;
  }>;
  gates: Array<{
    type: QualityGate['type'];
    trigger: QualityGate['trigger'];
    wave?: number;
    requiresHumanApproval: boolean;
    description: string;
  }>;
}

const PLAN_TEMPLATES: PlanTemplate[] = [
  {
    name: 'feature-implementation',
    keywords: ['implement', 'feature', 'add', 'build', 'create'],
    topology: 'wave',
    waves: [
      {
        roles: ['architect'],
        fileScopes: { architect: ['**/*.md', 'docs/**'] },
        tasks: [
          {
            title: 'Design system architecture',
            description: 'Design API contracts, module boundaries, and write ADR',
            role: 'architect',
            filePaths: ['docs/architecture/**/*.md'],
            parallelSafe: false,
          },
        ],
      },
      {
        roles: ['implementer', 'implementer'],
        fileScopes: {
          implementer: ['src/**/*.ts', '!src/**/*.test.ts'],
        },
        tasks: [
          {
            title: 'Implement core logic',
            description: 'Write production code following the architecture',
            role: 'implementer',
            filePaths: ['src/**/*.ts'],
            parallelSafe: true,
            blockedBy: ['Design system architecture'],
          },
        ],
      },
      {
        roles: ['test_runner', 'doc_writer'],
        fileScopes: {
          test_runner: ['src/**/*.test.ts', 'tests/**'],
          doc_writer: ['*.md', 'docs/**', 'CHANGELOG.md'],
        },
        tasks: [
          {
            title: 'Write comprehensive tests',
            description: 'Unit and integration tests with coverage analysis',
            role: 'test_runner',
            filePaths: ['src/**/*.test.ts'],
            parallelSafe: true,
            blockedBy: ['Implement core logic'],
          },
          {
            title: 'Update documentation',
            description: 'README, CHANGELOG, and wiki pages',
            role: 'doc_writer',
            filePaths: ['README.md', 'CHANGELOG.md', 'docs/**'],
            parallelSafe: true,
            blockedBy: ['Implement core logic'],
          },
        ],
      },
      {
        roles: ['security_reviewer'],
        fileScopes: { security_reviewer: [] },
        tasks: [
          {
            title: 'Security audit',
            description: 'Review for OWASP Top 10 and dependency vulnerabilities',
            role: 'security_reviewer',
            filePaths: [],
            parallelSafe: false,
            blockedBy: ['Write comprehensive tests'],
          },
        ],
      },
    ],
    gates: [
      {
        type: 'plan_approval',
        trigger: 'pre_wave',
        wave: 0,
        requiresHumanApproval: true,
        description: 'Human must approve the architecture design before implementation begins',
      },
      {
        type: 'test_pass',
        trigger: 'post_wave',
        wave: 2,
        requiresHumanApproval: false,
        description: 'All tests must pass before security review',
      },
      {
        type: 'security_clear',
        trigger: 'post_wave',
        wave: 3,
        requiresHumanApproval: true,
        description: 'Security reviewer must approve before merge',
      },
    ],
  },
  {
    name: 'refactor',
    keywords: ['refactor', 'cleanup', 'debt', 'improve'],
    topology: 'sequential',
    waves: [
      {
        roles: ['refactor'],
        fileScopes: { refactor: ['src/**/*.ts'] },
        tasks: [
          {
            title: 'Refactor codebase',
            description: 'Clean up technical debt and improve code quality',
            role: 'refactor',
            filePaths: ['src/**/*.ts'],
            parallelSafe: false,
          },
        ],
      },
      {
        roles: ['test_runner'],
        fileScopes: { test_runner: ['src/**/*.test.ts'] },
        tasks: [
          {
            title: 'Verify refactor with tests',
            description: 'Ensure all tests still pass after refactor',
            role: 'test_runner',
            filePaths: ['src/**/*.test.ts'],
            parallelSafe: false,
            blockedBy: ['Refactor codebase'],
          },
        ],
      },
    ],
    gates: [
      {
        type: 'test_pass',
        trigger: 'post_wave',
        wave: 1,
        requiresHumanApproval: false,
        description: 'All tests must pass after refactor',
      },
    ],
  },
  {
    name: 'bug-fix',
    keywords: ['fix', 'bug', 'error', 'issue'],
    topology: 'sequential',
    waves: [
      {
        roles: ['implementer'],
        fileScopes: { implementer: ['src/**/*.ts'] },
        tasks: [
          {
            title: 'Fix bug',
            description: 'Identify root cause and implement fix',
            role: 'implementer',
            filePaths: ['src/**/*.ts'],
            parallelSafe: false,
          },
        ],
      },
      {
        roles: ['test_runner'],
        fileScopes: { test_runner: ['src/**/*.test.ts'] },
        tasks: [
          {
            title: 'Add regression test',
            description: 'Write test that would have caught this bug',
            role: 'test_runner',
            filePaths: ['src/**/*.test.ts'],
            parallelSafe: false,
            blockedBy: ['Fix bug'],
          },
        ],
      },
    ],
    gates: [
      {
        type: 'test_pass',
        trigger: 'post_wave',
        wave: 1,
        requiresHumanApproval: false,
        description: 'New regression test must pass',
      },
    ],
  },
  {
    name: 'research',
    keywords: ['research', 'explore', 'investigate', 'spike'],
    topology: 'debate',
    waves: [
      {
        roles: ['researcher', 'researcher'],
        fileScopes: { researcher: [] },
        tasks: [
          {
            title: 'Research approach A',
            description: 'Explore first potential solution',
            role: 'researcher',
            filePaths: [],
            parallelSafe: true,
          },
          {
            title: 'Research approach B',
            description: 'Explore alternative solution',
            role: 'researcher',
            filePaths: [],
            parallelSafe: true,
          },
        ],
      },
      {
        roles: ['architect'],
        fileScopes: { architect: ['docs/**'] },
        tasks: [
          {
            title: 'Synthesize research findings',
            description: 'Compare approaches and recommend best path forward',
            role: 'architect',
            filePaths: ['docs/research/**/*.md'],
            parallelSafe: false,
            blockedBy: ['Research approach A', 'Research approach B'],
          },
        ],
      },
    ],
    gates: [
      {
        type: 'output_review',
        trigger: 'post_wave',
        wave: 1,
        requiresHumanApproval: true,
        description: 'Human must approve the recommended approach',
      },
    ],
  },
];

// ─── Cost estimation constants ────────────────────────────────────────────────

const AVG_TOKENS_PER_TASK = {
  architect: 40_000,
  implementer: 30_000,
  test_runner: 20_000,
  security_reviewer: 25_000,
  doc_writer: 15_000,
  refactor: 30_000,
  researcher: 20_000,
  reviewer: 15_000,
  supervisor: 50_000,
  migration: 25_000,
  devops: 20_000,
  custom: 20_000,
};

const AVG_COST_PER_1K_TOKENS_USD = 0.015; // Rough average across models

// ─── DelegationPlanner ────────────────────────────────────────────────────────

export class DelegationPlanner {
  constructor(private readonly registry: AgentRegistry) {}

  /**
   * Generate a delegation plan from a natural-language goal.
   *
   * Planning strategy:
   *  1. Match goal keywords against plan templates
   *  2. If match found → customize the template for this goal
   *  3. If no match → use a simple parallel topology with role inference
   *  4. Validate the plan (circular deps, file scope conflicts, etc.)
   *  5. Return the structured plan
   */
  async plan(
    teamId: string,
    goal: string,
    topologyHint?: TeamTopology,
  ): Promise<DelegationPlan> {
    const goalLower = goal.toLowerCase();

    // 1) Try to match a template
    const template = this._matchTemplate(goalLower, topologyHint);

    if (template) {
      return this._buildFromTemplate(teamId, goal, template);
    }

    // 2) Fallback: infer a simple plan from goal keywords
    return this._buildInferredPlan(teamId, goal, topologyHint ?? 'parallel');
  }

  /**
   * Validate a plan for common issues.
   * Throws on validation failure.
   */
  validate(plan: DelegationPlan): void {
    // Check for circular dependencies
    this._checkCircularDependencies(plan);

    // Check for file scope overlaps in parallel waves
    this._checkFileScopeConflicts(plan);

    // Ensure all referenced roles are registered
    this._checkRoleRegistration(plan);
  }

  /**
   * Estimate the token cost and USD cost of executing a plan.
   * Updates plan.estimatedTokens and plan.estimatedCostUsd in place.
   */
  estimateCost(plan: DelegationPlan): void {
    let totalTokens = 0;

    for (const wave of plan.waves) {
      for (const member of wave.members) {
        const def = this.registry.get(member.name) ?? this.registry.getByRole(member.role);
        const tokensPerTask = def?.maxContextTokens ?? AVG_TOKENS_PER_TASK[member.role];
        totalTokens += tokensPerTask * member.tasks.length;
      }
    }

    plan.estimatedTokens = totalTokens;
    plan.estimatedCostUsd = (totalTokens / 1000) * AVG_COST_PER_1K_TOKENS_USD;
  }

  // ─── Template matching ─────────────────────────────────────────────────────

  private _matchTemplate(
    goalLower: string,
    topologyHint?: TeamTopology,
  ): PlanTemplate | null {
    const candidates = PLAN_TEMPLATES.filter((t) =>
      t.keywords.some((kw) => goalLower.includes(kw)),
    );

    if (candidates.length === 0) return null;

    // Prefer template matching both keywords AND topology hint
    if (topologyHint) {
      const exact = candidates.find((t) => t.topology === topologyHint);
      if (exact) return exact;
    }

    // Fall back to first keyword match
    return candidates[0];
  }

  private _buildFromTemplate(
    teamId: string,
    goal: string,
    template: PlanTemplate,
  ): DelegationPlan {
    const waves: DelegationWave[] = [];

    for (let i = 0; i < template.waves.length; i++) {
      const waveTemplate = template.waves[i];
      const members: DelegationWave['members'] = [];

      // Build members from template roles
      const roleCounts = new Map<AgentRole, number>();
      for (const role of waveTemplate.roles) {
        const count = roleCounts.get(role) ?? 0;
        roleCounts.set(role, count + 1);
        const suffix = count > 0 ? `-${count + 1}` : '';
        const memberName = `${role}${suffix}`;

        const tasksForThisMember = waveTemplate.tasks.filter(
          (t) => t.role === role,
        );

        members.push({
          name: memberName,
          role,
          prompt: `You are ${memberName} in wave ${i + 1}`,
          fileScope: waveTemplate.fileScopes[role] ?? [],
          tasks: tasksForThisMember.map((t) => ({
            title: t.title,
            description: t.description,
            filePaths: t.filePaths,
            parallelSafe: t.parallelSafe,
            blockedBy: t.blockedBy ?? [],
          })),
        });
      }

      waves.push({ waveNumber: i, members });
    }

    const gates: QualityGate[] = template.gates.map((g) => ({
      type: g.type,
      description: g.description,
      trigger: g.trigger,
      wave: g.wave,
      requiresHumanApproval: g.requiresHumanApproval,
    }));

    const plan: DelegationPlan = {
      teamId,
      topology: template.topology,
      waves,
      gates,
      estimatedTokens: 0,
      estimatedCostUsd: 0,
      reasoning: `Matched template: ${template.name}`,
    };

    this.estimateCost(plan);
    return plan;
  }

  // ─── Inferred planning (simple heuristic) ─────────────────────────────────

  private _buildInferredPlan(
    teamId: string,
    goal: string,
    topology: TeamTopology,
  ): DelegationPlan {
    // Simple heuristic: one implementer + one test_runner in parallel
    const waves: DelegationWave[] = [
      {
        waveNumber: 0,
        members: [
          {
            name: 'implementer',
            role: 'implementer',
            prompt: 'Implement the requested functionality',
            fileScope: ['src/**/*.ts', '!src/**/*.test.ts'],
            tasks: [
              {
                title: 'Implement goal',
                description: goal,
                filePaths: ['src/**/*.ts'],
                parallelSafe: false,
                blockedBy: [],
              },
            ],
          },
          {
            name: 'test-runner',
            role: 'test_runner',
            prompt: 'Write tests for the implementation',
            fileScope: ['src/**/*.test.ts'],
            tasks: [
              {
                title: 'Write tests',
                description: 'Write comprehensive tests',
                filePaths: ['src/**/*.test.ts'],
                parallelSafe: true,
                blockedBy: ['Implement goal'],
              },
            ],
          },
        ],
      },
    ];

    const gates: QualityGate[] = [
      {
        type: 'test_pass',
        trigger: 'post_wave',
        wave: 0,
        requiresHumanApproval: false,
        description: 'All tests must pass',
      },
    ];

    const plan: DelegationPlan = {
      teamId,
      topology,
      waves,
      gates,
      estimatedTokens: 0,
      estimatedCostUsd: 0,
      reasoning: 'Inferred simple plan: implementer + test_runner',
    };

    this.estimateCost(plan);
    return plan;
  }

  // ─── Validation helpers ────────────────────────────────────────────────────

  private _checkCircularDependencies(plan: DelegationPlan): void {
    const taskNames = new Set<string>();
    const deps = new Map<string, string[]>();

    for (const wave of plan.waves) {
      for (const member of wave.members) {
        for (const task of member.tasks) {
          taskNames.add(task.title);
          deps.set(task.title, task.blockedBy ?? []);
        }
      }
    }

    // DFS cycle detection
    const visited = new Set<string>();
    const recStack = new Set<string>();

    const hasCycle = (task: string): boolean => {
      if (!visited.has(task)) {
        visited.add(task);
        recStack.add(task);

        const blockers = deps.get(task) ?? [];
        for (const blocker of blockers) {
          if (!visited.has(blocker) && hasCycle(blocker)) return true;
          if (recStack.has(blocker)) return true;
        }
      }
      recStack.delete(task);
      return false;
    };

    for (const task of taskNames) {
      if (hasCycle(task)) {
        throw new Error(
          `DelegationPlanner: circular dependency detected involving task "${task}"`,
        );
      }
    }
  }

  private _checkFileScopeConflicts(plan: DelegationPlan): void {
    // For each wave, check if members with overlapping file scopes
    // are running parallel tasks
    for (const wave of plan.waves) {
      const parallelMembers = wave.members.filter((m) =>
        m.tasks.some((t) => t.parallelSafe),
      );

      for (let i = 0; i < parallelMembers.length; i++) {
        for (let j = i + 1; j < parallelMembers.length; j++) {
          const a = parallelMembers[i];
          const b = parallelMembers[j];
          if (this._scopesOverlap(a.fileScope, b.fileScope)) {
            throw new Error(
              `DelegationPlanner: file scope conflict between parallel members "${a.name}" and "${b.name}" in wave ${wave.waveNumber}`,
            );
          }
        }
      }
    }
  }

  private _scopesOverlap(a: string[], b: string[]): boolean {
    // Simple heuristic: if any scope pattern from A could match a path in B's scope, they overlap
    // In production this would use a proper glob matcher
    for (const aScope of a) {
      for (const bScope of b) {
        // Exact match or prefix match
        if (aScope === bScope) return true;
        if (aScope.startsWith(bScope) || bScope.startsWith(aScope)) return true;
      }
    }
    return false;
  }

  private _checkRoleRegistration(plan: DelegationPlan): void {
    for (const wave of plan.waves) {
      for (const member of wave.members) {
        if (!this.registry.getByRole(member.role)) {
          throw new Error(
            `DelegationPlanner: role "${member.role}" for member "${member.name}" is not registered in AgentRegistry`,
          );
        }
      }
    }
  }
}
src/FileLockManager.ts
TypeScript

/**
 * FileLockManager — file-scope lock enforcement for @locoworker/orchestrator
 *
 * FileLockManager prevents concurrent writes to the same file by multiple
 * team members, avoiding the "two agents editing the same file" race condition
 * that would otherwise require merge conflict resolution.
 *
 * Lock model (inspired by advisory file locks, but DB-based):
 *  - A member declares intent to write to a file before executing a task
 *  - FileLockManager checks if the file is already locked by another member
 *  - If locked → conflict → block the member until lock is released
 *  - If not locked → acquire lock (with TTL)
 *  - On task completion → release lock
 *  - Locks expire after TTL (default 30 min) even if not explicitly released
 *
 * Glob support:
 *  - File scope patterns (e.g., "src/auth/**") are matched using a simple
 *    glob matcher. This allows members to lock entire directories.
 *  - Overlap detection: "src/**" overlaps with "src/auth/foo.ts"
 *
 * Integration:
 *  - AgentSpawner sets member.fileScope at spawn time
 *  - OrchestratorEngine calls FileLockManager.acquireForTask() before
 *    a member starts a task
 *  - OrchestratorEngine calls FileLockManager.releaseForMember() after
 *    a member completes or fails a task
 *  - FileLockManager.expireStaleLocks() is called on each engine tick
 *
 * Design:
 *  - FileLockManager is a thin wrapper over OrchestratorStore
 *  - All lock state is persisted in SQLite (orch_file_locks table)
 *  - Conflict detection is synchronous (DB query)
 *  - FileLockManager is safe to call from any class
 */

import type { OrchestratorStore } from './OrchestratorStore.js';
import type {
  FileLock,
  TeamMember,
  SharedTask,
} from './types/orchestrator.types.js';
import { DEFAULT_LOCK_TTL_MINUTES } from './types/orchestrator.types.js';

// ─── Glob matching helpers ────────────────────────────────────────────────────

/**
 * Simple glob matcher — supports ** and * wildcards.
 * In production this would use a battle-tested glob library (minimatch, picomatch).
 * Here we implement a minimal version for demonstration.
 */
function globMatch(pattern: string, path: string): boolean {
  // Convert glob to regex
  const regexStr = pattern
    .replace(/\./g, '\\.')                  // escape dots
    .replace(/\*\*/g, '§§')                 // temp marker for **
    .replace(/\*/g, '[^/]*')                // * → [^/]*
    .replace(/§§/g, '.*')                   // ** → .*
    .replace(/\?/g, '.');                   // ? → .

  const regex = new RegExp(`^${regexStr}$`);
  return regex.test(path);
}

/**
 * Check if two glob patterns can match overlapping paths.
 * e.g., "src/**" and "src/auth/**" overlap.
 */
function globsOverlap(a: string, b: string): boolean {
  // Simple heuristic: if one is a prefix of the other, they overlap
  // More sophisticated: expand both patterns and check for intersection
  if (a === b) return true;

  // Normalize patterns
  const aNorm = a.replace(/\*\*/g, '').replace(/\*/g, '');
  const bNorm = b.replace(/\*\*/g, '').replace(/\*/g, '');

  if (aNorm.startsWith(bNorm) || bNorm.startsWith(aNorm)) return true;

  // If both contain **, they likely overlap
  if (a.includes('**') && b.includes('**')) {
    const aBase = a.split('**')[0];
    const bBase = b.split('**')[0];
    if (aBase === bBase) return true;
  }

  return false;
}

// ─── Conflict types ───────────────────────────────────────────────────────────

export interface LockConflict {
  filePath: string;
  existingLock: FileLock;
  requestedBy: TeamMember;
}

export interface LockAcquisitionResult {
  success: boolean;
  locks: FileLock[];
  conflicts: LockConflict[];
}

// ─── FileLockManager ──────────────────────────────────────────────────────────

export class FileLockManager {
  constructor(private readonly store: OrchestratorStore) {}

  /**
   * Acquire locks for all file paths in a task.
   * If any path is already locked by a different member → conflict.
   *
   * Returns:
   *  - success=true + locks[] if all locks acquired
   *  - success=false + conflicts[] if any lock conflicts
   */
  acquireForTask(
    teamId: string,
    member: TeamMember,
    task: SharedTask,
    ttlMinutes = DEFAULT_LOCK_TTL_MINUTES,
  ): LockAcquisitionResult {
    const filePaths = task.filePaths;
    if (filePaths.length === 0) {
      return { success: true, locks: [], conflicts: [] };
    }

    // Expire stale locks first
    this.expireStaleLocks(teamId);

    const locks: FileLock[] = [];
    const conflicts: LockConflict[] = [];

    for (const filePath of filePaths) {
      const result = this.store.acquireFileLock(
        teamId,
        member.id,
        member.name,
        filePath,
        task.id,
        ttlMinutes,
      );

      if ('conflict' in result) {
        conflicts.push({
          filePath,
          existingLock: result.conflict,
          requestedBy: member,
        });
      } else {
        locks.push(result);
      }
    }

    if (conflicts.length > 0) {
      // Roll back any locks we just acquired
      for (const lock of locks) {
        this.store.releaseFileLock(lock.id);
      }
      return { success: false, locks: [], conflicts };
    }

    return { success: true, locks, conflicts: [] };
  }

  /**
   * Check if a member's file scope would conflict with any active locks.
   * Used for preemptive conflict detection before spawning a wave.
   *
   * Returns an array of conflicting locks (empty if no conflicts).
   */
  checkScopeConflicts(
    teamId: string,
    memberName: string,
    fileScope: string[],
  ): FileLock[] {
    this.expireStaleLocks(teamId);

    const activeLocks = this.store.getActiveFileLocks(teamId);
    const conflicts: FileLock[] = [];

    for (const lock of activeLocks) {
      const lockRow = this.store._rowToFileLock(lock);
      // Skip locks held by this member
      if (lockRow.memberName === memberName) continue;

      // Check if any scope pattern overlaps with the locked file
      for (const scopePattern of fileScope) {
        if (
          globMatch(scopePattern, lockRow.filePath) ||
          globsOverlap(scopePattern, lockRow.filePath)
        ) {
          conflicts.push(lockRow);
          break;
        }
      }
    }

    return conflicts;
  }

  /**
   * Release all locks held by a member (called on task completion or failure).
   */
  releaseForMember(teamId: string, memberId: string): number {
    return this.store.releaseAllLocksForMember(teamId, memberId);
  }

  /**
   * Release a specific lock by ID.
   */
  releaseLock(lockId: string): boolean {
    return this.store.releaseFileLock(lockId);
  }

  /**
   * Expire all locks whose TTL has passed.
   * Should be called periodically by the engine.
   */
  expireStaleLocks(teamId: string): number {
    return this.store.expireStaleFileLocks(teamId);
  }

  /**
   * Get all active locks for a team (for reporting).
   */
  getActiveLocks(teamId: string): FileLock[] {
    const rows = this.store.getActiveFileLocks(teamId);
    return rows.map((r) => this.store._rowToFileLock(r));
  }

  /**
   * Check if a specific file path is currently locked.
   */
  isLocked(teamId: string, filePath: string): boolean {
    const locks = this.getActiveLocks(teamId);
    return locks.some((lock) => lock.filePath === filePath);
  }

  /**
   * Get the member who holds a lock on a file (or null).
   */
  getLockHolder(teamId: string, filePath: string): string | null {
    const locks = this.getActiveLocks(teamId);
    const lock = locks.find((l) => l.filePath === filePath);
    return lock?.memberName ?? null;
  }

  /**
   * Format conflicts as a human-readable summary.
   */
  formatConflicts(conflicts: LockConflict[]): string {
    if (conflicts.length === 0) return 'No conflicts';

    const lines = [
      `${conflicts.length} file lock conflict${conflicts.length !== 1 ? 's' : ''}:`,
      '',
    ];

    for (const conflict of conflicts) {
      lines.push(
        `- **${conflict.filePath}** is locked by **${conflict.existingLock.memberName}** ` +
          `(expires ${conflict.existingLock.expiresAt.slice(0, 16)})`,
      );
    }

    return lines.join('\n');
  }
}
src/ResultAggregator.ts
TypeScript

/**
 * ResultAggregator — collects and merges sub-agent outputs for @locoworker/orchestrator
 *
 * ResultAggregator implements the "completing" phase of a team's lifecycle:
 *  - Collect all AgentResult records from OrchestratorStore
 *  - Detect file-level conflicts (multiple members modified the same file)
 *  - Merge non-conflicting outputs into a structured synthesis
 *  - Produce a final aggregated report for the supervisor to review
 *
 * Conflict detection rules:
 *  - If two members modified the same file → conflict
 *  - If one member's output failed its quality gate → gate_failed conflict
 *  - Otherwise → merge
 *
 * Merge strategy:
 *  - Non-conflicting file changes are accepted
 *  - Results are ordered by wave number (architect → implementer → test → docs)
 *  - A structured synthesis report lists all outputs + conflicts
 *
 * Design:
 *  - ResultAggregator is stateless (no DB writes)
 *  - It reads from OrchestratorStore
 *  - The engine uses the aggregated report to determine team completion status
 */

import type { OrchestratorStore } from './OrchestratorStore.js';
import type {
  AgentResult,
  TeamMember,
  OrchestratorTeam,
} from './types/orchestrator.types.js';

// ─── Result types ─────────────────────────────────────────────────────────────

export interface AggregatedResult {
  teamId: string;
  teamName: string;
  teamGoal: string;
  totalResults: number;
  successfulResults: number;
  failedGates: number;
  conflicts: ResultConflict[];
  filesModified: string[];
  synthesis: string;
  resultsByMember: Map<string, AgentResult[]>;
  orderedResults: AgentResult[];
  duration: number;
  totalTokens: number;
  totalCost: number;
}

export interface ResultConflict {
  type: 'file_conflict' | 'gate_failed';
  description: string;
  involvedMembers: string[];
  filePath?: string;
  resultIds: string[];
}

// ─── ResultAggregator ─────────────────────────────────────────────────────────

export class ResultAggregator {
  constructor(private readonly store: OrchestratorStore) {}

  /**
   * Aggregate all results for a team.
   * Returns a structured AggregatedResult with conflict detection.
   */
  aggregate(teamId: string): AggregatedResult {
    const team = this.store.getTeam(teamId);
    if (!team) {
      throw new Error(`ResultAggregator: team ${teamId} not found`);
    }

    const results = this.store.getResults(teamId);
    const members = this.store.queryMembers({ teamId });

    // Build member map
    const memberMap = new Map<string, TeamMember>(
      members.map((row) => {
        const m = this.store._rowToMember(row);
        return [m.id, m];
      }),
    );

    // Parse result rows
    const agentResults: AgentResult[] = results.map((row) => ({
      id: row.id,
      teamId: row.team_id,
      memberId: row.member_id,
      memberName: row.member_name,
      memberRole: row.member_role,
      taskId: row.task_id ?? undefined,
      content: row.content,
      filesModified: JSON.parse(row.files_modified_json) as string[],
      gateApproved: row.gate_approved === null
        ? undefined
        : row.gate_approved === 1,
      producedAt: row.produced_at,
      tokensUsed: row.tokens_used,
    }));

    // Group by member
    const resultsByMember = new Map<string, AgentResult[]>();
    for (const result of agentResults) {
      const existing = resultsByMember.get(result.memberName) ?? [];
      existing.push(result);
      resultsByMember.set(result.memberName, existing);
    }

    // Detect conflicts
    const conflicts = this._detectConflicts(agentResults);

    // Collect all modified files (deduplicated)
    const filesModified = [
      ...new Set(agentResults.flatMap((r) => r.filesModified)),
    ].sort();

    // Order results by wave (architect → implementer → test → docs)
    const orderedResults = this._orderResults(agentResults, memberMap);

    // Build synthesis
    const synthesis = this._buildSynthesis(team, orderedResults, conflicts);

    // Compute stats
    const successfulResults = agentResults.filter(
      (r) => r.gateApproved !== false,
    ).length;
    const failedGates = agentResults.filter((r) => r.gateApproved === false).length;
    const totalTokens = agentResults.reduce((sum, r) => sum + r.tokensUsed, 0);

    const duration =
      team.completedAt && team.createdAt
        ? new Date(team.completedAt).getTime() - new Date(team.createdAt).getTime()
        : 0;

    return {
      teamId,
      teamName: team.name,
      teamGoal: team.goal,
      totalResults: agentResults.length,
      successfulResults,
      failedGates,
      conflicts,
      filesModified,
      synthesis,
      resultsByMember,
      orderedResults,
      duration,
      totalTokens,
      totalCost: team.totalCostUsd,
    };
  }

  /**
   * Format the aggregated result as a Markdown summary for the supervisor.
   */
  format(agg: AggregatedResult): string {
    const lines: string[] = [
      `# Team "${agg.teamName}" — Aggregated Results`,
      '',
      `**Goal:** ${agg.teamGoal}`,
      '',
      `**Status:** ${agg.conflicts.length === 0 ? '✅ Clean merge' : `⚠ ${agg.conflicts.length} conflict${agg.conflicts.length !== 1 ? 's' : ''}`}`,
      '',
      '## Summary',
      '',
      `- **Results:** ${agg.totalResults} total (${agg.successfulResults} successful, ${agg.failedGates} failed gates)`,
      `- **Files modified:** ${agg.filesModified.length}`,
      `- **Duration:** ${Math.round(agg.duration / 60000)} minutes`,
      `- **Tokens used:** ${agg.totalTokens.toLocaleString()}`,
      `- **Cost:** $${agg.totalCost.toFixed(4)}`,
      '',
    ];

    if (agg.conflicts.length > 0) {
      lines.push('## ⚠ Conflicts', '');
      for (const conflict of agg.conflicts) {
        lines.push(`### ${conflict.type}`);
        lines.push(conflict.description);
        lines.push(`- Members: ${conflict.involvedMembers.join(', ')}`);
        if (conflict.filePath) lines.push(`- File: \`${conflict.filePath}\``);
        lines.push('');
      }
    }

    lines.push('## Outputs by Member', '');
    for (const [memberName, results] of agg.resultsByMember.entries()) {
      lines.push(`### ${memberName} (${results.length} result${results.length !== 1 ? 's' : ''})`);
      for (const result of results) {
        const gate = result.gateApproved === true
          ? '✅'
          : result.gateApproved === false
            ? '❌'
            : '';
        lines.push(`- ${gate} ${result.content.slice(0, 100)}…`);
        if (result.filesModified.length > 0) {
          lines.push(`  - Files: ${result.filesModified.slice(0, 3).join(', ')}${result.filesModified.length > 3 ? '…' : ''}`);
        }
      }
      lines.push('');
    }

    if (agg.filesModified.length > 0) {
      lines.push('## Files Modified', '');
      const truncated = agg.filesModified.slice(0, 20);
      for (const file of truncated) {
        lines.push(`- \`${file}\``);
      }
      if (agg.filesModified.length > 20) {
        lines.push(`- _(and ${agg.filesModified.length - 20} more)_`);
      }
      lines.push('');
    }

    lines.push('## Synthesis', '', agg.synthesis);

    return lines.join('\n');
  }

  // ─── Conflict detection ────────────────────────────────────────────────────

  private _detectConflicts(results: AgentResult[]): ResultConflict[] {
    const conflicts: ResultConflict[] = [];

    // 1) Gate failures
    const gateFailed = results.filter((r) => r.gateApproved === false);
    if (gateFailed.length > 0) {
      conflicts.push({
        type: 'gate_failed',
        description: `${gateFailed.length} result${gateFailed.length !== 1 ? 's' : ''} failed quality gate review`,
        involvedMembers: [...new Set(gateFailed.map((r) => r.memberName))],
        resultIds: gateFailed.map((r) => r.id),
      });
    }

    // 2) File conflicts (two members modified the same file)
    const fileToResults = new Map<string, AgentResult[]>();
    for (const result of results) {
      for (const file of result.filesModified) {
        const existing = fileToResults.get(file) ?? [];
        existing.push(result);
        fileToResults.set(file, existing);
      }
    }

    for (const [file, rs] of fileToResults.entries()) {
      if (rs.length > 1) {
        const uniqueMembers = [...new Set(rs.map((r) => r.memberName))];
        if (uniqueMembers.length > 1) {
          conflicts.push({
            type: 'file_conflict',
            description: `File \`${file}\` was modified by multiple members: ${uniqueMembers.join(', ')}`,
            involvedMembers: uniqueMembers,
            filePath: file,
            resultIds: rs.map((r) => r.id),
          });
        }
      }
    }

    return conflicts;
  }

  // ─── Result ordering ───────────────────────────────────────────────────────

  private _orderResults(
    results: AgentResult[],
    memberMap: Map<string, TeamMember>,
  ): AgentResult[] {
    // Sort by wave number, then by produced_at
    return [...results].sort((a, b) => {
      const aMember = memberMap.get(a.memberId);
      const bMember = memberMap.get(b.memberId);
      const aWave = aMember?.wave ?? 999;
      const bWave = bMember?.wave ?? 999;

      if (aWave !== bWave) return aWave - bWave;

      return a.producedAt.localeCompare(b.producedAt);
    });
  }

  // ─── Synthesis builder ─────────────────────────────────────────────────────

  private _buildSynthesis(
    team: OrchestratorTeam,
    orderedResults: AgentResult[],
    conflicts: ResultConflict[],
  ): string {
    const parts: string[] = [
      `The team "${team.name}" worked on: ${team.goal}`,
      '',
    ];

    if (orderedResults.length === 0) {
      parts.push('No results were produced by team members.');
      return parts.join('\n');
    }

    parts.push('**Work completed:**');
    for (const result of orderedResults) {
      const summary = result.content.split('\n')[0]?.slice(0, 120) ?? '';
      parts.push(`- **${result.memberName}** (${result.memberRole}): ${summary}`);
    }

    parts.push('');

    if (conflicts.length > 0) {
      parts.push(
        `⚠ **${conflicts.length} conflict${conflicts.length !== 1 ? 's' : ''} detected.** ` +
          `Manual merge or review required before final integration.`,
      );
    } else {
      parts.push('✅ All outputs are non-conflicting and ready for integration.');
    }

    return parts.join('\n');
  }
}
src/OrchestratorEngine.ts
TypeScript

/**
 * OrchestratorEngine — main coordination loop for @locoworker/orchestrator
 *
 * OrchestratorEngine is the beating heart of the multi-agent system.
 * It drives a team through all lifecycle phases:
 *
 *   planning → spawning → running → gating → completing → done/failed/aborted
 *
 * Integration with all subsystems:
 *  - OrchestratorStore (persistence)
 *  - DelegationPlanner (plan generation)
 *  - AgentRegistry + AgentSpawner (member creation)
 *  - MessageRouter (communication)
 *  - FileLockManager (conflict prevention)
 *  - ResultAggregator (synthesis)
 *  - EventBus (for live updates to UI)
 *  - HooksRegistry (for beforeTurn hooks to process markers)
 *
 * Core responsibilities:
 *  - Create a team from a goal
 *  - Execute the delegation plan wave-by-wave
 *  - Gate enforcement (wait for human approval at gate points)
 *  - Monitor member progress (poll shared tasks)
 *  - Detect and handle failures (retry, skip, abort)
 *  - Aggregate results and transition team to 'done'
 *  - Expose control APIs (abort, pause, resume, approve gate)
 *
 * Design patterns (consistent with KairosAgent):
 *  - No hard dependency on packages/core (uses shims)
 *  - EventBus and HooksRegistry are injected
 *  - All state persisted to OrchestratorStore (survives restarts)
 *  - Safe to instantiate multiple engines for different teams
 *
 * Note: This implementation is a reference scaffold. A production engine
 * would also include sophisticated scheduling, adaptive retry, and
 * integration with packages/kairos for task queue management.
 */

import type { OrchestratorStore } from './OrchestratorStore.js';
import type { AgentRegistry } from './AgentRegistry.js';
import type { DelegationPlanner } from './DelegationPlanner.js';
import type { AgentSpawner, SessionManagerLike, CoreModelConfig } from './AgentSpawner.js';
import type { MessageRouter } from './MessageRouter.js';
import type { FileLockManager } from './FileLockManager.js';
import type { ResultAggregator } from './ResultAggregator.js';
import type {
  OrchestratorTeam,
  TeamMember,
  SharedTask,
  DelegationPlan,
  QualityGate,
  TeamTopology,
  SpawnRequest,
} from './types/orchestrator.types.js';

// ─── Core interface shims ─────────────────────────────────────────────────────

interface EventBusLike {
  onAny(handler: (event: { type: string; [key: string]: unknown }) => void | Promise<void>): () => void;
  emit(event: { type: string; [key: string]: unknown }): Promise<void>;
}

interface HooksRegistryLike {
  register(
    name: string,
    handler: (payload: unknown) => unknown | Promise<unknown>,
    priority?: number,
  ): void;
}

// ─── Option types ─────────────────────────────────────────────────────────────

export interface OrchestratorEngineOptions {
  /** Default model config for all members unless overridden. */
  defaultModelConfig: CoreModelConfig;

  /** Working directory for all members. */
  workingDirectory: string;

  /** If true, log engine state changes to console. */
  verbose?: boolean;

  /** Poll interval for checking shared task status (ms). Default: 5000. */
  pollIntervalMs?: number;

  /** Max automatic retries for failed tasks. Default: 1. */
  maxTaskRetries?: number;
}

export interface CreateTeamRequest {
  name: string;
  goal: string;
  topologyHint?: TeamTopology;
  supervisorSessionId: string;
  tags?: string[];
  metadata?: Record<string, unknown>;
}

export interface TeamExecutionResult {
  teamId: string;
  status: 'done' | 'failed' | 'aborted';
  duration: number;
  totalCost: number;
  aggregatedResult?: ReturnType<ResultAggregator['aggregate']>;
  error?: string;
}

// ─── OrchestratorEngine ───────────────────────────────────────────────────────

export class OrchestratorEngine {
  private _eventBus: EventBusLike | null = null;
  private _unsubscribe: (() => void) | null = null;
  private readonly _verbose: boolean;
  private readonly _pollIntervalMs: number;
  private readonly _maxTaskRetries: number;
  private readonly _runningTeams = new Set<string>();

  constructor(
    private readonly store: OrchestratorStore,
    private readonly registry: AgentRegistry,
    private readonly planner: DelegationPlanner,
    private readonly spawner: AgentSpawner,
    private readonly router: MessageRouter,
    private readonly lockManager: FileLockManager,
    private readonly aggregator: ResultAggregator,
    private readonly options: OrchestratorEngineOptions,
  ) {
    this._verbose = options.verbose ?? false;
    this._pollIntervalMs = options.pollIntervalMs ?? 5000;
    this._maxTaskRetries = options.maxTaskRetries ?? 1;
  }

  // ─── Lifecycle ──────────────────────────────────────────────────────────────

  /**
   * Attach to EventBus and HooksRegistry.
   * Registers hooks that process [MESSAGE:]/[BROADCAST] markers.
   */
  start(eventBus: EventBusLike, hooks: HooksRegistryLike): this {
    this._eventBus = eventBus;
    this.router.attachEventBus(eventBus);

    // Register beforeTurn hook to process message markers
    hooks.register(
      'beforeTurn',
      async (payload: unknown) => {
        // Extract session ID from payload, look up member
        const p = payload as { sessionId?: string; messages?: Array<{ role: string; content: string }> };
        if (!p.sessionId || !p.messages) return payload;

        // Find member by session ID
        const memberRows = this.store.db
          .prepare<[string], { id: string; team_id: string; name: string }>(
            'SELECT id, team_id, name FROM orch_members WHERE session_id = ?',
          )
          .all(p.sessionId);

        if (memberRows.length === 0) return payload;

        const memberRow = memberRows[0];
        const member = this.store.getMember(memberRow.id);
        if (!member) return payload;

        const team = this.store.getTeam(memberRow.team_id);
        if (!team) return payload;

        // Extract markers from assistant messages
        for (const msg of p.messages) {
          if (msg.role === 'assistant' && typeof msg.content === 'string') {
            const extracted = this.router.extractMarkers(msg.content);
            if (extracted.length > 0) {
              // Build member map
              const allMembers = this.store.queryMembers({ teamId: team.id });
              const membersByName = new Map<string, TeamMember>(
                allMembers.map((row) => {
                  const m = this.store._rowToMember(row);
                  return [m.name.toLowerCase(), m];
                }),
              );

              const supervisorMember = team.supervisorMemberId
                ? this.store.getMember(team.supervisorMemberId)
                : undefined;

              this.router.processExtractedMarkers(
                team.id,
                member,
                extracted,
                membersByName,
                supervisorMember,
              );
            }
          }
        }

        return payload;
      },
      30, // priority 30 — before temporal/wiki hooks
    );

    this._log('OrchestratorEngine started');
    return this;
  }

  stop(): void {
    if (this._unsubscribe) {
      this._unsubscribe();
      this._unsubscribe = null;
    }
    this._eventBus = null;
    this._log('OrchestratorEngine stopped');
  }

  // ─── Team creation ──────────────────────────────────────────────────────────

  /**
   * Create a new team and generate a delegation plan.
   * Returns the team and plan, but does NOT execute it yet.
   * Call execute(teamId) to start execution.
   */
  async createTeam(request: CreateTeamRequest): Promise<{
    team: OrchestratorTeam;
    plan: DelegationPlan;
  }> {
    this._log(`Creating team "${request.name}" for goal: ${request.goal}`);

    // 1) Create team record (status: planning)
    const team = this.store.createTeam({
      name: request.name,
      goal: request.goal,
      status: 'planning',
      topology: request.topologyHint ?? 'parallel',
      supervisorSessionId: request.supervisorSessionId,
      maxConcurrentMembers: 8,
      maxTotalMembers: 20,
      gates: [],
      tags: request.tags ?? [],
      metadata: request.metadata ?? {},
    });

    // 2) Generate delegation plan
    const plan = await this.planner.plan(team.id, request.goal, request.topologyHint);

    // 3) Validate plan
    this.planner.validate(plan);

    // 4) Update team with plan's topology and gates
    this.store.updateTeam(team.id, {
      topology: plan.topology,
      gates: plan.gates,
    });

    // 5) Create shared tasks from plan
    for (const wave of plan.waves) {
      for (const member of wave.members) {
        for (let i = 0; i < member.tasks.length; i++) {
          const task = member.tasks[i];
          this.store.createSharedTask({
            teamId: team.id,
            title: task.title,
            description: task.description,
            status: 'pending',
            blockedBy: task.blockedBy,
            parallelSafe: task.parallelSafe,
            filePaths: task.filePaths,
            tags: [],
            order: wave.waveNumber * 1000 + i,
          });
        }
      }
    }

    this._log(`Team "${request.name}" created with ${plan.waves.length} wave(s)`);

    await this._emit({
      type: 'orchestrator_team_created',
      teamId: team.id,
      teamName: team.name,
      wavesCount: plan.waves.length,
    });

    return { team, plan };
  }

  // ─── Execution ──────────────────────────────────────────────────────────────

  /**
   * Execute a team's delegation plan end-to-end.
   * Drives the team through all lifecycle phases until completion.
   */
  async execute(teamId: string): Promise<TeamExecutionResult> {
    if (this._runningTeams.has(teamId)) {
      throw new Error(`Team ${teamId} is already executing`);
    }

    this._runningTeams.add(teamId);
    const start = Date.now();

    try {
      const team = this.store.getTeam(teamId);
      if (!team) {
        throw new Error(`Team ${teamId} not found`);
      }

      this._log(`Executing team "${team.name}"`);

      // Load the plan from shared tasks
      const plan = await this._reconstructPlan(teamId);

      // Phase 1: Spawn all members
      await this._phaseSpawn(teamId, plan);

      // Phase 2: Run waves
      await this._phaseRun(teamId, plan);

      // Phase 3: Completing
      this.store.transitionTeam(teamId, 'completing');

      // Phase 4: Aggregate results
      const aggregatedResult = this.aggregator.aggregate(teamId);

      // Phase 5: Done
      const finalTeam = this.store.transitionTeam(teamId, 'done');
      if (!finalTeam) throw new Error('Team lost during execution');

      const duration = Date.now() - start;
      this._log(`Team "${team.name}" completed in ${Math.round(duration / 1000)}s`);

      await this._emit({
        type: 'orchestrator_team_completed',
        teamId,
        teamName: team.name,
        duration,
        totalCost: finalTeam.totalCostUsd,
      });

      return {
        teamId,
        status: 'done',
        duration,
        totalCost: finalTeam.totalCostUsd,
        aggregatedResult,
      };
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : String(err);
      this._log(`Team execution failed: ${errorMessage}`);

      this.store.transitionTeam(teamId, 'failed', { errorMessage });

      await this._emit({
        type: 'orchestrator_team_failed',
        teamId,
        errorMessage,
      });

      return {
        teamId,
        status: 'failed',
        duration: Date.now() - start,
        totalCost: 0,
        error: errorMessage,
      };
    } finally {
      this._runningTeams.delete(teamId);
      await this.spawner.teardownTeam(teamId);
    }
  }

  /**
   * Abort a running team execution.
   */
  async abort(teamId: string, reason = 'User aborted'): Promise<void> {
    this._log(`Aborting team ${teamId}: ${reason}`);
    this.store.transitionTeam(teamId, 'aborted', { errorMessage: reason });
    await this.spawner.teardownTeam(teamId);
    this._runningTeams.delete(teamId);
  }

  // ─── Phase implementations ──────────────────────────────────────────────────

  private async _phaseSpawn(teamId: string, plan: DelegationPlan): Promise<void> {
    this.store.transitionTeam(teamId, 'spawning');
    this._log(`[${teamId}] Phase: spawning`);

    // Spawn members wave by wave
    for (const wave of plan.waves) {
      const requests: SpawnRequest[] = wave.members.map((m) => ({
        teamId,
        memberName: m.name,
        role: m.role,
        fileScope: m.fileScope,
        customPrompt: m.prompt,
        wave: wave.waveNumber,
      }));

      const results = await this.spawner.spawnWave(requests, true);

      for (const result of results) {
        if (!result.success) {
          throw new Error(`Failed to spawn member "${result.member.name}": ${result.error}`);
        }
      }

      this._log(`[${teamId}] Spawned wave ${wave.waveNumber} (${requests.length} members)`);
    }

    // Set first member as supervisor if not already set
    const team = this.store.getTeam(teamId);
    if (team && !team.supervisorMemberId) {
      const firstMember = this.store.queryMembers({ teamId, wave: 0 })[0];
      if (firstMember) {
        this.store.updateTeam(teamId, {
          supervisorMemberId: this.store._rowToMember(firstMember).id,
        });
      }
    }
  }

  private async _phaseRun(teamId: string, plan: DelegationPlan): Promise<void> {
    this.store.transitionTeam(teamId, 'running');
    this._log(`[${teamId}] Phase: running`);

    // Run waves sequentially (or in parallel for parallel topology)
    for (const wave of plan.waves) {
      await this._runWave(teamId, wave);
    }
  }

  private async _runWave(teamId: string, wave: typeof DelegationPlan.prototype.waves[0]): Promise<void> {
    this._log(`[${teamId}] Running wave ${wave.waveNumber}`);

    // Get all ready tasks for this wave
    const readyTasks = this.store.querySharedTasks({
      teamId,
      status: 'pending',
      readyOnly: true,
    });

    // Assign tasks to members
    for (const taskRow of readyTasks) {
      const task = this.store._rowToSharedTask(taskRow);
      // Find a member who can work on this task
      const members = this.store.queryMembers({ teamId, wave: wave.waveNumber, status: 'idle' });
      if (members.length > 0) {
        const member = this.store._rowToMember(members[0]);
        this.store.transitionSharedTask(task.id, 'in_progress', { assignedMemberId: member.id });
        this._log(`[${teamId}] Assigned task "${task.title}" to ${member.name}`);
      }
    }

    // Poll until all wave tasks are complete
    await this._pollUntilWaveComplete(teamId, wave.waveNumber);
  }

  private async _pollUntilWaveComplete(teamId: string, waveNumber: number): Promise<void> {
    // eslint-disable-next-line no-constant-condition
    while (true) {
      const inProgress = this.store.querySharedTasks({
        teamId,
        status: ['in_progress', 'pending', 'blocked'],
      });

      if (inProgress.length === 0) break;

      await new Promise((resolve) => setTimeout(resolve, this._pollIntervalMs));
    }
  }

  private async _reconstructPlan(teamId: string): Promise<DelegationPlan> {
    const team = this.store.getTeam(teamId);
    if (!team) throw new Error('Team not found');

    const members = this.store.queryMembers({ teamId });
    const tasks = this.store.querySharedTasks({ teamId });

    // Group members by wave
    const waveMap = new Map<number, typeof members>();
    for (const memberRow of members) {
      const member = this.store._rowToMember(memberRow);
      const existing = waveMap.get(member.wave) ?? [];
      existing.push(memberRow);
      waveMap.set(member.wave, existing);
    }

    const waves: DelegationPlan['waves'] = [];
    for (const [waveNum, memberRows] of [...waveMap.entries()].sort((a, b) => a[0] - b[0])) {
      waves.push({
        waveNumber: waveNum,
        members: memberRows.map((row) => {
          const member = this.store._rowToMember(row);
          return {
            name: member.name,
            role: member.role,
            prompt: member.customPrompt ?? '',
            fileScope: member.fileScope,
            tasks: [],
          };
        }),
      });
    }

    return {
      teamId,
      topology: team.topology,
      waves,
      gates: team.gates,
      estimatedTokens: 0,
      estimatedCostUsd: 0,
      reasoning: 'Reconstructed from DB',
    };
  }

  // ─── Helpers ───────────────────────────────────────────────────────────────

  private async _emit(event: { type: string; [key: string]: unknown }): Promise<void> {
    if (this._eventBus) {
      try {
        await this._eventBus.emit(event);
      } catch {
        // Never crash on EventBus failure
      }
    }
  }

  private _log(msg: string): void {
    if (this._verbose) {
      console.log(`[OrchestratorEngine] ${msg}`);
    }
  }
}
src/OrchestratorReporter.ts
TypeScript

/**
 * OrchestratorReporter — ORCHESTRATOR_REPORT.md generator for @locoworker/orchestrator
 *
 * Generates a human-readable Markdown report covering:
 *  - Team timeline (created → spawning → running → done)
 *  - Member status table (role, wave, status, tokens, cost)
 *  - Shared task list with dependencies
 *  - Message log excerpt (broadcast + supervisor messages)
 *  - File lock conflicts (if any)
 *  - Result synthesis (from ResultAggregator)
 *  - Cost breakdown
 *  - Suggested next actions
 *
 * Design:
 *  - OrchestratorReporter is read-only (no DB writes)
 *  - It composes data from OrchestratorStore, ResultAggregator, AgentRegistry
 *  - Output is written to `.locoworker/orchestrator/ORCHESTRATOR_REPORT.md`
 */

import { writeFile, mkdir } from 'node:fs/promises';
import { dirname } from 'node:path';
import type { OrchestratorStore } from './OrchestratorStore.js';
import type { ResultAggregator } from './ResultAggregator.js';
import type { AgentRegistry } from './AgentRegistry.js';

export interface OrchestratorReporterOptions {
  outputPath?: string;
  maxMessages?: number;
  maxTasks?: number;
}

export interface ReportResult {
  outputPath: string;
  generatedAt: string;
  estimatedTokens: number;
}

export class OrchestratorReporter {
  constructor(
    private readonly store: OrchestratorStore,
    private readonly aggregator: ResultAggregator,
    private readonly registry: AgentRegistry,
  ) {}

  async generate(
    teamId: string,
    options: OrchestratorReporterOptions = {},
  ): Promise<ReportResult> {
    const outputPath = options.outputPath ?? '.locoworker/orchestrator/ORCHESTRATOR_REPORT.md';
    const maxMessages = options.maxMessages ?? 50;
    const maxTasks = options.maxTasks ?? 50;
    const generatedAt = new Date().toISOString();

    const team = this.store.getTeam(teamId);
    if (!team) throw new Error(`Team ${teamId} not found`);

    const sections: string[] = [];

    sections.push(this._buildHeader(team, generatedAt));
    sections.push(this._buildTimeline(team));
    sections.push(this._buildMemberTable(teamId));
    sections.push(this._buildTaskList(teamId, maxTasks));
    sections.push(this._buildMessageLog(teamId, maxMessages));
    sections.push(this._buildFileLocks(teamId));

    if (team.status === 'done' || team.status === 'failed') {
      sections.push(this._buildResultSynthesis(teamId));
    }

    sections.push(this._buildCostBreakdown(team));
    sections.push(this._buildSuggestedActions(team));
    sections.push(this._buildFooter());

    const report = sections.join('\n\n');

    await mkdir(dirname(outputPath), { recursive: true });
    await writeFile(outputPath, report, 'utf8');

    return {
      outputPath,
      generatedAt,
      estimatedTokens: Math.ceil(report.length / 4),
    };
  }

  private _buildHeader(team: typeof OrchestratorStore.prototype._rowToTeam extends (row: infer R) => infer T ? T : never, generatedAt: string): string {
    const statusEmoji = {
      planning: '📋',
      spawning: '🚀',
      running: '⚙️',
      gating: '🚦',
      completing: '🔄',
      done: '✅',
      failed: '❌',
      aborted: '🛑',
    }[team.status] ?? '❓';

    return [
      `# 🤝 Orchestrator Report — Team "${team.name}"`,
      '',
      `> **Status:** ${statusEmoji} ${team.status}  `,
      `> **Generated:** ${generatedAt.slice(0, 16).replace('T', ' ')} UTC  `,
      `> **Team ID:** \`${team.id}\``,
      '',
      `## Goal`,
      '',
      team.goal,
      '',
      '---',
    ].join('\n');
  }

  private _buildTimeline(team: typeof OrchestratorStore.prototype._rowToTeam extends (row: infer R) => infer T ? T : never): string {
    const lines = [
      '## 📅 Timeline',
      '',
      '| Event | Timestamp |',
      '|-------|-----------|',
      `| Created | ${team.createdAt.slice(0, 16)} UTC |`,
    ];

    if (team.completedAt) {
      const duration = (
        new Date(team.completedAt).getTime() - new Date(team.createdAt).getTime()
      ) / 60000;
      lines.push(`| Completed | ${team.completedAt.slice(0, 16)} UTC |`);
      lines.push(`| Duration | ${Math.round(duration)} minutes |`);
    }

    return lines.join('\n');
  }

  private _buildMemberTable(teamId: string): string {
    const members = this.store.queryMembers({ teamId });

    if (members.length === 0) {
      return '## 👥 Team Members\n\n_No members spawned._';
    }

    const lines = [
      '## 👥 Team Members',
      '',
      '| Name | Role | Wave | Status | Tokens | Cost |',
      '|------|------|------|--------|--------|------|',
    ];

    for (const row of members) {
      const member = this.store._rowToMember(row);
      const statusEmoji = {
        spawning: '🔄',
        idle: '⏸',
        working: '⚙️',
        waiting: '⏳',
        reviewing: '🔍',
        shutdown_req: '🛑',
        done: '✅',
        failed: '❌',
        evicted: '🚫',
      }[member.status] ?? '❓';

      lines.push(
        `| **${member.name}** | ${member.role} | ${member.wave} | ${statusEmoji} ${member.status} | ` +
          `${member.tokensUsed.toLocaleString()} | $${member.costUsd.toFixed(4)} |`,
      );
    }

    return lines.join('\n');
  }

  private _buildTaskList(teamId: string, maxTasks: number): string {
    const tasks = this.store.querySharedTasks({ teamId, limit: maxTasks });

    if (tasks.length === 0) {
      return '## ✅ Shared Task List\n\n_No tasks defined._';
    }

    const lines = [
      '## ✅ Shared Task List',
      '',
      '| Title | Status | Assigned | Blocked By |',
      '|-------|--------|----------|------------|',
    ];

    for (const row of tasks) {
      const task = this.store._rowToSharedTask(row);
      const statusEmoji = {
        pending: '⏳',
        in_progress: '⚙️',
        completed: '✅',
        blocked: '🚫',
        failed: '❌',
        skipped: '⏭',
        gating: '🚦',
      }[task.status] ?? '❓';

      const assigned = task.assignedMemberId
        ? this.store.getMember(task.assignedMemberId)?.name ?? '—'
        : '—';

      const blockers = task.blockedBy.length > 0
        ? task.blockedBy.slice(0, 2).join(', ') + (task.blockedBy.length > 2 ? '…' : '')
        : '—';

      lines.push(
        `| ${task.title} | ${statusEmoji} ${task.status} | ${assigned} | ${blockers} |`,
      );
    }

    return lines.join('\n');
  }

  private _buildMessageLog(teamId: string, maxMessages: number): string {
    const messages = this.store.getMessages(teamId, { limit: maxMessages });

    if (messages.length === 0) {
      return '## 💬 Message Log\n\n_No messages sent._';
    }

    const lines = [
      '## 💬 Message Log',
      '',
      `Showing last ${Math.min(messages.length, maxMessages)} messages:`,
      '',
    ];

    for (const msg of messages.slice(-maxMessages)) {
      const time = msg.sent_at.slice(11, 16);
      const from = msg.from_member_name ?? 'system';
      const to = msg.to_member_name
        ? ` → ${msg.to_member_name}`
        : msg.channel === 'broadcast'
          ? ' → [broadcast]'
          : '';
      const preview = msg.content.slice(0, 80).replace(/\n/g, ' ');
      lines.push(`- **${time}** ${from}${to}: ${preview}${msg.content.length > 80 ? '…' : ''}`);
    }

    return lines.join('\n');
  }

  private _buildFileLocks(teamId: string): string {
    const locks = this.store.getActiveFileLocks(teamId);

    if (locks.length === 0) {
      return '## 🔒 File Locks\n\n_No active locks._';
    }

    const lines = [
      '## 🔒 Active File Locks',
      '',
      '| File | Held By | Acquired | Expires |',
      '|------|---------|----------|---------|',
    ];

    for (const row of locks) {
      const lock = this.store._rowToFileLock(row);
      lines.push(
        `| \`${lock.filePath}\` | ${lock.memberName} | ` +
          `${lock.acquiredAt.slice(11, 16)} | ${lock.expiresAt.slice(11, 16)} |`,
      );
    }

    return lines.join('\n');
  }

  private _buildResultSynthesis(teamId: string): string {
    try {
      const agg = this.aggregator.aggregate(teamId);
      const formatted = this.aggregator.format(agg);
      return '## 📊 Result Synthesis\n\n' + formatted;
    } catch (err) {
      return `## 📊 Result Synthesis\n\n_Error aggregating results: ${err instanceof Error ? err.message : String(err)}_`;
    }
  }

  private _buildCostBreakdown(team: typeof OrchestratorStore.prototype._rowToTeam extends (row: infer R) => infer T ? T : never): string {
    const members = this.store.queryMembers({ teamId: team.id });
    const lines = [
      '## 💰 Cost Breakdown',
      '',
      '| Member | Tokens | Cost (USD) |',
      '|--------|--------|------------|',
    ];

    for (const row of members) {
      const member = this.store._rowToMember(row);
      lines.push(
        `| ${member.name} | ${member.tokensUsed.toLocaleString()} | $${member.costUsd.toFixed(4)} |`,
      );
    }

    lines.push(
      `| **Total** | **${team.totalTokensUsed.toLocaleString()}** | **$${team.totalCostUsd.toFixed(4)}** |`,
    );

    return lines.join('\n');
  }

  private _buildSuggestedActions(team: typeof OrchestratorStore.prototype._rowToTeam extends (row: infer R) => infer T ? T : never): string {
    const actions: string[] = [];

    if (team.status === 'gating') {
      actions.push('- 🚦 **Quality gate active** — review and approve to continue.');
    }

    if (team.status === 'failed') {
      actions.push('- ❌ **Team failed** — review error logs and retry failed tasks.');
    }

    const failedTasks = this.store.querySharedTasks({ teamId: team.id, status: 'failed' });
    if (failedTasks.length > 0) {
      actions.push(`- ⚠ **${failedTasks.length} task(s) failed** — review and retry or skip.`);
    }

    const activeLocks = this.store.getActiveFileLocks(team.id);
    if (activeLocks.length > 5) {
      actions.push(`- 🔒 **${activeLocks.length} active file locks** — may indicate contention.`);
    }

    if (team.status === 'done') {
      actions.push('- ✅ **Team complete** — review aggregated results and integrate outputs.');
    }

    if (actions.length === 0) {
      actions.push('- ℹ️ No immediate actions required.');
    }

    return '## 💡 Suggested Actions\n\n' + actions.join('\n');
  }

  private _buildFooter(): string {
    return [
      '---',
      '',
      `_Report generated by LocoWorker Orchestrator · ${new Date().toISOString()}_`,
    ].join('\n');
  }
}
src/index.ts
TypeScript

/**
 * @locoworker/orchestrator — public API barrel export
 *
 * Core classes:
 *   OrchestratorStore    — SQLite persistence (7 tables: teams/members/messages/tasks/locks/results/meta)
 *   AgentRegistry        — Capability-based role catalogue + definition management
 *   AgentSpawner         — Sub-agent session factory (creates sessions via SessionManagerLike)
 *   MessageRouter        — Peer-to-peer + broadcast messaging (TeammateTool.write/broadcast pattern)
 *   DelegationPlanner    — Goal → typed delegation plan with waves, deps, gates
 *   FileLockManager      — File-scope lock enforcement with glob matching
 *   ResultAggregator     — Collects sub-agent outputs, detects conflicts, synthesizes
 *   OrchestratorEngine   — Main coordination loop (planning → spawning → running → completing → done)
 *   OrchestratorReporter — ORCHESTRATOR_REPORT.md generator
 */

// ─── Classes ──────────────────────────────────────────────────────────────────
export { OrchestratorStore } from './OrchestratorStore.js';
export { AgentRegistry } from './AgentRegistry.js';
export { AgentSpawner } from './AgentSpawner.js';
export { MessageRouter } from './MessageRouter.js';
export { DelegationPlanner } from './DelegationPlanner.js';
export { FileLockManager } from './FileLockManager.js';
export { ResultAggregator } from './ResultAggregator.js';
export { OrchestratorEngine } from './OrchestratorEngine.js';
export { OrchestratorReporter } from './OrchestratorReporter.js';

// ─── Types ────────────────────────────────────────────────────────────────────
export type {
  // Team model
  OrchestratorTeam,
  TeamStatus,
  TeamTopology,

  // Member model
  TeamMember,
  MemberStatus,
  AgentRole,
  AgentPermissionProfile,
  AgentDefinition,

  // Shared task model
  SharedTask,
  SharedTaskStatus,

  // Message model
  TeamMessage,
  MessageChannel,

  // Quality gate model
  QualityGate,
  GateType,

  // Result model
  AgentResult,

  // File lock model
  FileLock,
  FileLockStatus,

  // DB row types
  TeamRow,
  MemberRow,
  SharedTaskRow,
  MessageRow,
  ResultRow,
  FileLockRow,

  // Query types
  TeamQueryOptions,
  MemberQueryOptions,
  SharedTaskQueryOptions,
  OrchestratorStoreStats,

  // Spawn types
  SpawnRequest,
  SpawnResult,

  // Delegation plan
  DelegationPlan,
  DelegationWave,
} from './types/orchestrator.types.js';

export type {
  // AgentSpawner
  CoreModelConfig,
  CoreSessionConfig,
  CoreSession,
  SessionManagerLike,
  AgentSpawnerOptions,
  TeardownResult,
} from './AgentSpawner.js';

export type {
  // MessageRouter
  SendOptions,
  ExtractedMessage,
  RouterStats,
} from './MessageRouter.js';

export type {
  // FileLockManager
  LockConflict,
  LockAcquisitionResult,
} from './FileLockManager.js';

export type {
  // ResultAggregator
  AggregatedResult,
  ResultConflict,
} from './ResultAggregator.js';

export type {
  // OrchestratorEngine
  OrchestratorEngineOptions,
  CreateTeamRequest,
  TeamExecutionResult,
} from './OrchestratorEngine.js';

export type {
  // OrchestratorReporter
  OrchestratorReporterOptions,
  ReportResult,
} from './OrchestratorReporter.js';

// ─── Zod schemas ─────────────────────────────────────────────────────────────
export {
  AgentDefinitionSchema,
  OrchestratorTeamSchema,
} from './types/orchestrator.types.js';

// ─── Constants ────────────────────────────────────────────────────────────────
export {
  ORCHESTRATOR_SCHEMA_VERSION,
  DEFAULT_ORCHESTRATOR_DB,
  DEFAULT_MAX_CONCURRENT_MEMBERS,
  DEFAULT_MAX_TOTAL_MEMBERS,
  DEFAULT_LOCK_TTL_MINUTES,
  MAX_MESSAGES_PER_QUERY,
  DELEGATE_MARKER_REGEX,
  APPROVE_PLAN_MARKER,
  REJECT_PLAN_MARKER_REGEX,
  BUILT_IN_AGENT_DEFINITIONS,
} from './types/orchestrator.types.js';


Pass 7 — Part 2 Continuation (Test Suites)
src/tests/OrchestratorStore.test.ts
TypeScript

/**
 * OrchestratorStore test suite
 * Run with: bun test src/tests/OrchestratorStore.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test';
import { OrchestratorStore } from '../OrchestratorStore.js';
import type {
  OrchestratorTeam,
  TeamMember,
  SharedTask,
  TeamMessage,
  AgentResult,
  FileLock,
} from '../types/orchestrator.types.js';

// ─── Fixtures ─────────────────────────────────────────────────────────────────

function makeTeamInput(
  overrides: Partial<Omit<OrchestratorTeam, 'id' | 'createdAt' | 'updatedAt' | 'totalTokensUsed' | 'totalCostUsd'>> = {},
): Omit<OrchestratorTeam, 'id' | 'createdAt' | 'updatedAt' | 'totalTokensUsed' | 'totalCostUsd'> {
  return {
    name: 'test-team',
    goal: 'Build a feature',
    status: 'planning',
    topology: 'parallel',
    supervisorSessionId: 'session-123',
    maxConcurrentMembers: 8,
    maxTotalMembers: 20,
    gates: [],
    tags: ['test'],
    metadata: {},
    ...overrides,
  };
}

function makeMemberInput(
  teamId: string,
  overrides: Partial<Omit<TeamMember, 'id' | 'createdAt' | 'updatedAt' | 'tokensUsed' | 'costUsd'>> = {},
): Omit<TeamMember, 'id' | 'createdAt' | 'updatedAt' | 'tokensUsed' | 'costUsd'> {
  return {
    teamId,
    name: 'implementer',
    role: 'implementer',
    status: 'spawning',
    permissionProfile: 'standard',
    fileScope: ['src/**/*.ts'],
    wave: 0,
    dependsOn: [],
    assignedTaskIds: [],
    ...overrides,
  };
}

function makeTaskInput(
  teamId: string,
  overrides: Partial<Omit<SharedTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'>> = {},
): Omit<SharedTask, 'id' | 'createdAt' | 'updatedAt' | 'attemptCount'> {
  return {
    teamId,
    title: 'Implement feature',
    status: 'pending',
    blockedBy: [],
    parallelSafe: true,
    filePaths: ['src/feature.ts'],
    tags: [],
    order: 0,
    ...overrides,
  };
}

// ─── Setup ────────────────────────────────────────────────────────────────────

let store: OrchestratorStore;

beforeEach(() => {
  store = new OrchestratorStore(':memory:');
});

afterEach(() => {
  store.close();
});

// ─── Team CRUD ────────────────────────────────────────────────────────────────

describe('OrchestratorStore — team CRUD', () => {
  it('creates a team with a generated UUID', () => {
    const team = store.createTeam(makeTeamInput());
    expect(team.id).toMatch(/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/);
  });

  it('persists team fields correctly', () => {
    const input = makeTeamInput({
      name: 'my-team',
      goal: 'Ship feature X',
      topology: 'wave',
      tags: ['urgent', 'frontend'],
    });
    const team = store.createTeam(input);
    const fetched = store.getTeam(team.id);
    expect(fetched).toBeDefined();
    expect(fetched!.name).toBe('my-team');
    expect(fetched!.goal).toBe('Ship feature X');
    expect(fetched!.topology).toBe('wave');
    expect(fetched!.tags).toEqual(['urgent', 'frontend']);
  });

  it('returns undefined for unknown team id', () => {
    expect(store.getTeam('00000000-0000-0000-0000-000000000000')).toBeUndefined();
  });

  it('updates team and bumps updated_at', () => {
    const team = store.createTeam(makeTeamInput());
    const before = team.updatedAt;
    const updated = store.updateTeam(team.id, { status: 'running' });
    expect(updated!.status).toBe('running');
    expect(updated!.updatedAt >= before).toBe(true);
  });

  it('transitionTeam sets completedAt on terminal status', () => {
    const team = store.createTeam(makeTeamInput());
    const done = store.transitionTeam(team.id, 'done');
    expect(done!.status).toBe('done');
    expect(done!.completedAt).toBeDefined();
  });

  it('queryTeams filters by status', () => {
    store.createTeam(makeTeamInput({ name: 'team-1', status: 'planning' }));
    store.createTeam(makeTeamInput({ name: 'team-2', status: 'running' }));
    store.createTeam(makeTeamInput({ name: 'team-3', status: 'done' }));

    const running = store.queryTeams({ status: 'running' });
    expect(running.length).toBe(1);
    expect(running[0].name).toBe('team-2');
  });
});

// ─── Member CRUD ──────────────────────────────────────────────────────────────

describe('OrchestratorStore — member CRUD', () => {
  let teamId: string;

  beforeEach(() => {
    const team = store.createTeam(makeTeamInput());
    teamId = team.id;
  });

  it('creates a member with generated UUID', () => {
    const member = store.createMember(makeMemberInput(teamId));
    expect(member.id).toMatch(/^[0-9a-f-]{36}$/);
  });

  it('persists member fields correctly', () => {
    const input = makeMemberInput(teamId, {
      name: 'test-runner',
      role: 'test_runner',
      fileScope: ['tests/**'],
      wave: 2,
    });
    const member = store.createMember(input);
    const fetched = store.getMember(member.id);
    expect(fetched!.name).toBe('test-runner');
    expect(fetched!.role).toBe('test_runner');
    expect(fetched!.fileScope).toEqual(['tests/**']);
    expect(fetched!.wave).toBe(2);
  });

  it('getMemberByName finds member by team + name', () => {
    store.createMember(makeMemberInput(teamId, { name: 'architect' }));
    const found = store.getMemberByName(teamId, 'architect');
    expect(found).toBeDefined();
    expect(found!.name).toBe('architect');
  });

  it('queryMembers filters by status', () => {
    store.createMember(makeMemberInput(teamId, { name: 'm1', status: 'idle' }));
    store.createMember(makeMemberInput(teamId, { name: 'm2', status: 'working' }));
    store.createMember(makeMemberInput(teamId, { name: 'm3', status: 'idle' }));

    const idle = store.queryMembers({ teamId, status: 'idle' });
    expect(idle.length).toBe(2);
  });

  it('transitionMember sets startedAt when transitioning to working', () => {
    const member = store.createMember(makeMemberInput(teamId));
    const working = store.transitionMember(member.id, 'working');
    expect(working!.status).toBe('working');
    expect(working!.startedAt).toBeDefined();
  });

  it('transitionMember sets completedAt on terminal status', () => {
    const member = store.createMember(makeMemberInput(teamId));
    const done = store.transitionMember(member.id, 'done');
    expect(done!.completedAt).toBeDefined();
  });
});

// ─── Shared tasks ─────────────────────────────────────────────────────────────

describe('OrchestratorStore — shared tasks', () => {
  let teamId: string;

  beforeEach(() => {
    teamId = store.createTeam(makeTeamInput()).id;
  });

  it('creates a shared task', () => {
    const task = store.createSharedTask(makeTaskInput(teamId));
    expect(task.id).toBeDefined();
    expect(task.attemptCount).toBe(0);
  });

  it('querySharedTasks filters by status', () => {
    store.createSharedTask(makeTaskInput(teamId, { title: 't1', status: 'pending' }));
    store.createSharedTask(makeTaskInput(teamId, { title: 't2', status: 'completed' }));
    store.createSharedTask(makeTaskInput(teamId, { title: 't3', status: 'pending' }));

    const pending = store.querySharedTasks({ teamId, status: 'pending' });
    expect(pending.length).toBe(2);
  });

  it('querySharedTasks readyOnly returns only unblocked tasks', () => {
    const t1 = store.createSharedTask(makeTaskInput(teamId, { title: 't1', status: 'completed' }));
    store.createSharedTask(makeTaskInput(teamId, { title: 't2', status: 'pending', blockedBy: [t1.id] }));
    store.createSharedTask(makeTaskInput(teamId, { title: 't3', status: 'pending', blockedBy: ['ghost'] }));
    store.createSharedTask(makeTaskInput(teamId, { title: 't4', status: 'pending', blockedBy: [] }));

    const ready = store.querySharedTasks({ teamId, readyOnly: true });
    // t2 should be ready (blocker completed), t3 blocked by ghost (not ready), t4 ready
    const titles = ready.map((r) => store._rowToSharedTask(r).title);
    expect(titles).toContain('t2');
    expect(titles).toContain('t4');
    expect(titles).not.toContain('t3');
  });

  it('transitionSharedTask increments attemptCount on in_progress', () => {
    const task = store.createSharedTask(makeTaskInput(teamId));
    store.transitionSharedTask(task.id, 'in_progress');
    const updated = store.getSharedTask(task.id);
    expect(updated!.attemptCount).toBe(1);
  });

  it('autoSkipBlockedTasks skips tasks whose blockers all failed', () => {
    const failed = store.createSharedTask(makeTaskInput(teamId, { title: 'failed', status: 'failed' }));
    store.createSharedTask(makeTaskInput(teamId, { title: 'blocked', status: 'blocked', blockedBy: [failed.id] }));

    const skipped = store.autoSkipBlockedTasks(teamId);
    expect(skipped).toBe(1);

    const blockedTask = store.querySharedTasks({ teamId, status: 'skipped' });
    expect(blockedTask.length).toBe(1);
  });
});

// ─── Messages ─────────────────────────────────────────────────────────────────

describe('OrchestratorStore — messages', () => {
  let teamId: string;
  let member1: TeamMember;
  let member2: TeamMember;

  beforeEach(() => {
    teamId = store.createTeam(makeTeamInput()).id;
    member1 = store.createMember(makeMemberInput(teamId, { name: 'm1' }));
    member2 = store.createMember(makeMemberInput(teamId, { name: 'm2' }));
  });

  it('sends a direct message', () => {
    const msg = store.sendMessage({
      teamId,
      fromMemberId: member1.id,
      fromMemberName: member1.name,
      toMemberId: member2.id,
      toMemberName: member2.name,
      channel: 'direct',
      content: 'Hello',
    });
    expect(msg.id).toBeDefined();
    expect(msg.read).toBe(false);
  });

  it('sends a broadcast message', () => {
    const msg = store.sendMessage({
      teamId,
      fromMemberId: member1.id,
      fromMemberName: member1.name,
      channel: 'broadcast',
      content: 'Team update',
    });
    expect(msg.channel).toBe('broadcast');
  });

  it('getMessages filters to target member', () => {
    store.sendMessage({
      teamId,
      fromMemberId: member1.id,
      fromMemberName: member1.name,
      toMemberId: member2.id,
      toMemberName: member2.name,
      channel: 'direct',
      content: 'Direct to m2',
    });
    store.sendMessage({
      teamId,
      fromMemberId: member1.id,
      fromMemberName: member1.name,
      channel: 'broadcast',
      content: 'Broadcast',
    });

    const inbox = store.getMessages(teamId, { memberId: member2.id });
    expect(inbox.length).toBe(2); // direct + broadcast
  });

  it('markMessagesRead updates read flag', () => {
    const msg = store.sendMessage({
      teamId,
      fromMemberId: member1.id,
      fromMemberName: member1.name,
      toMemberId: member2.id,
      toMemberName: member2.name,
      channel: 'direct',
      content: 'Test',
    });

    const marked = store.markMessagesRead([msg.id]);
    expect(marked).toBe(1);

    const unread = store.getMessages(teamId, { unreadOnly: true });
    expect(unread.length).toBe(0);
  });
});

// ─── File locks ───────────────────────────────────────────────────────────────

describe('OrchestratorStore — file locks', () => {
  let teamId: string;
  let member: TeamMember;

  beforeEach(() => {
    teamId = store.createTeam(makeTeamInput()).id;
    member = store.createMember(makeMemberInput(teamId, { name: 'locker' }));
  });

  it('acquires a file lock', () => {
    const result = store.acquireFileLock(teamId, member.id, member.name, 'src/foo.ts');
    expect('id' in result).toBe(true);
    if ('id' in result) {
      expect(result.status).toBe('held');
    }
  });

  it('returns conflict when file already locked', () => {
    const member2 = store.createMember(makeMemberInput(teamId, { name: 'm2' }));
    store.acquireFileLock(teamId, member.id, member.name, 'src/foo.ts');
    const result = store.acquireFileLock(teamId, member2.id, member2.name, 'src/foo.ts');

    expect('conflict' in result).toBe(true);
    if ('conflict' in result) {
      expect(result.conflict.memberName).toBe(member.name);
    }
  });

  it('releaseFileLock marks lock as released', () => {
    const lock = store.acquireFileLock(teamId, member.id, member.name, 'src/foo.ts');
    if ('id' in lock) {
      const released = store.releaseFileLock(lock.id);
      expect(released).toBe(true);

      // Now can acquire again
      const result2 = store.acquireFileLock(teamId, member.id, member.name, 'src/foo.ts');
      expect('id' in result2).toBe(true);
    }
  });

  it('releaseAllLocksForMember releases multiple locks', () => {
    store.acquireFileLock(teamId, member.id, member.name, 'src/a.ts');
    store.acquireFileLock(teamId, member.id, member.name, 'src/b.ts');

    const count = store.releaseAllLocksForMember(teamId, member.id);
    expect(count).toBe(2);
  });

  it('expireStaleFileLocks expires locks past TTL', () => {
    // Acquire a lock with 0 minute TTL (expires immediately)
    const lock = store.acquireFileLock(teamId, member.id, member.name, 'src/foo.ts', undefined, 0);

    if ('id' in lock) {
      // Force expiry by setting expires_at to past
      store.db
        .prepare("UPDATE orch_file_locks SET expires_at = datetime('now', '-1 minute') WHERE id = ?")
        .run(lock.id);

      const expired = store.expireStaleFileLocks(teamId);
      expect(expired).toBe(1);

      const active = store.getActiveFileLocks(teamId);
      expect(active.length).toBe(0);
    }
  });
});

// ─── Results ──────────────────────────────────────────────────────────────────

describe('OrchestratorStore — results', () => {
  let teamId: string;
  let member: TeamMember;

  beforeEach(() => {
    teamId = store.createTeam(makeTeamInput()).id;
    member = store.createMember(makeMemberInput(teamId, { name: 'worker' }));
  });

  it('appends a result', () => {
    const result = store.appendResult({
      teamId,
      memberId: member.id,
      memberName: member.name,
      memberRole: member.role,
      content: 'Completed task X',
      filesModified: ['src/feature.ts'],
      tokensUsed: 1500,
    });
    expect(result.id).toBeDefined();
    expect(result.producedAt).toBeDefined();
  });

  it('getResults returns results for a team', () => {
    store.appendResult({
      teamId,
      memberId: member.id,
      memberName: member.name,
      memberRole: member.role,
      content: 'Result 1',
      filesModified: [],
      tokensUsed: 100,
    });
    store.appendResult({
      teamId,
      memberId: member.id,
      memberName: member.name,
      memberRole: member.role,
      content: 'Result 2',
      filesModified: [],
      tokensUsed: 200,
    });

    const results = store.getResults(teamId);
    expect(results.length).toBe(2);
  });

  it('getResults filters by memberId', () => {
    const m2 = store.createMember(makeMemberInput(teamId, { name: 'm2' }));
    store.appendResult({
      teamId,
      memberId: member.id,
      memberName: member.name,
      memberRole: member.role,
      content: 'From m1',
      filesModified: [],
      tokensUsed: 100,
    });
    store.appendResult({
      teamId,
      memberId: m2.id,
      memberName: m2.name,
      memberRole: m2.role,
      content: 'From m2',
      filesModified: [],
      tokensUsed: 100,
    });

    const m1Results = store.getResults(teamId, member.id);
    expect(m1Results.length).toBe(1);
    expect(m1Results[0].member_name).toBe('worker');
  });
});

// ─── Stats ────────────────────────────────────────────────────────────────────

describe('OrchestratorStore — getStats', () => {
  it('reports correct counts', () => {
    const t1 = store.createTeam(makeTeamInput({ name: 't1', status: 'running' }));
    const t2 = store.createTeam(makeTeamInput({ name: 't2', status: 'done' }));

    store.createMember(makeMemberInput(t1.id, { name: 'm1', status: 'working' }));
    store.createMember(makeMemberInput(t1.id, { name: 'm2', status: 'done' }));

    store.sendMessage({
      teamId: t1.id,
      channel: 'broadcast',
      content: 'Test',
    });

    const stats = store.getStats();
    expect(stats.totalTeams).toBe(2);
    expect(stats.activeTeams).toBe(1); // only t1 is running
    expect(stats.totalMembers).toBe(2);
    expect(stats.activeMembers).toBe(1); // only m1 is working
    expect(stats.totalMessages).toBe(1);
  });
});
src/tests/AgentRegistry.test.ts
TypeScript

/**
 * AgentRegistry test suite
 * Run with: bun test src/tests/AgentRegistry.test.ts
 */

import { describe, it, expect, beforeEach } from 'bun:test';
import { AgentRegistry } from '../AgentRegistry.js';
import type { AgentDefinition } from '../types/orchestrator.types.js';

// ─── Fixtures ─────────────────────────────────────────────────────────────────

function makeCustomDef(
  overrides: Partial<AgentDefinition> = {},
): AgentDefinition {
  return {
    name: 'custom-agent',
    role: 'custom',
    systemPromptSuffix: 'You are a custom agent.',
    allowedTools: [],
    deniedTools: [],
    permissionProfile: 'standard',
    capabilities: ['custom'],
    description: 'A custom agent for testing',
    supportsParallel: true,
    ...overrides,
  };
}

// ─── Setup ────────────────────────────────────────────────────────────────────

let registry: AgentRegistry;

beforeEach(() => {
  registry = new AgentRegistry();
});

// ─── Built-in definitions ─────────────────────────────────────────────────────

describe('AgentRegistry — built-in definitions', () => {
  it('registers 8 built-in definitions on construction', () => {
    const all = registry.listAll();
    expect(all.length).toBe(8);
  });

  it('built-in definitions include expected roles', () => {
    expect(registry.get('supervisor')).toBeDefined();
    expect(registry.get('architect')).toBeDefined();
    expect(registry.get('implementer')).toBeDefined();
    expect(registry.get('test-runner')).toBeDefined();
    expect(registry.get('security-reviewer')).toBeDefined();
    expect(registry.get('doc-writer')).toBeDefined();
    expect(registry.get('researcher')).toBeDefined();
    expect(registry.get('reviewer')).toBeDefined();
  });

  it('getByRole finds first matching role', () => {
    const def = registry.getByRole('architect');
    expect(def).toBeDefined();
    expect(def!.role).toBe('architect');
  });

  it('getAllByRole returns all matching definitions', () => {
    const impls = registry.getAllByRole('implementer');
    expect(impls.length).toBeGreaterThanOrEqual(1);
  });
});

// ─── Registration ─────────────────────────────────────────────────────────────

describe('AgentRegistry — custom registration', () => {
  it('registers a custom definition', () => {
    const def = makeCustomDef({ name: 'my-agent' });
    registry.register(def);
    expect(registry.has('my-agent')).toBe(true);
    expect(registry.get('my-agent')).toEqual(def);
  });

  it('overwrites existing definition with same name', () => {
    const def1 = makeCustomDef({ name: 'test', description: 'v1' });
    const def2 = makeCustomDef({ name: 'test', description: 'v2' });
    registry.register(def1);
    registry.register(def2);

    const fetched = registry.get('test');
    expect(fetched!.description).toBe('v2');
  });

  it('throws on invalid definition (Zod validation)', () => {
    const invalid = {
      name: 'INVALID NAME WITH SPACES',
      role: 'custom',
      systemPromptSuffix: '',
      capabilities: [],
      description: '',
    } as AgentDefinition;

    expect(() => registry.register(invalid)).toThrow();
  });

  it('unregister removes a definition', () => {
    const def = makeCustomDef({ name: 'temp' });
    registry.register(def);
    expect(registry.has('temp')).toBe(true);

    const removed = registry.unregister('temp');
    expect(removed).toBe(true);
    expect(registry.has('temp')).toBe(false);
  });

  it('unregister returns false for non-existent name', () => {
    expect(registry.unregister('ghost')).toBe(false);
  });
});

// ─── Capability-based discovery ───────────────────────────────────────────────

describe('AgentRegistry — findByCapabilities', () => {
  it('finds definitions matching all required capabilities', () => {
    const results = registry.findByCapabilities(['typescript', 'testing']);
    // test-runner has both 'testing' and 'typescript'
    expect(results.length).toBeGreaterThanOrEqual(1);
    expect(results.some((d) => d.role === 'test_runner')).toBe(true);
  });

  it('returns empty array when no matches', () => {
    const results = registry.findByCapabilities(['nonexistent-capability']);
    expect(results.length).toBe(0);
  });

  it('bestMatch returns highest-scoring definition', () => {
    const best = registry.bestMatch(['security', 'owasp']);
    expect(best).toBeDefined();
    expect(best!.role).toBe('security_reviewer');
  });

  it('bestMatch returns undefined when no match', () => {
    const best = registry.bestMatch(['ghost-capability']);
    expect(best).toBeUndefined();
  });

  it('getParallelSafe returns only definitions supporting parallel', () => {
    const parallel = registry.getParallelSafe();
    for (const def of parallel) {
      expect(def.supportsParallel).toBe(true);
    }
  });
});

// ─── System prompt generation ─────────────────────────────────────────────────

describe('AgentRegistry — buildSystemPromptSuffix', () => {
  it('generates a prompt with team context', () => {
    const suffix = registry.buildSystemPromptSuffix('implementer', {
      teamName: 'test-team',
      memberName: 'impl-1',
      fileScope: ['src/**/*.ts'],
      teamGoal: 'Build feature X',
    });

    expect(suffix).toContain('test-team');
    expect(suffix).toContain('impl-1');
    expect(suffix).toContain('src/**/*.ts');
    expect(suffix).toContain('Build feature X');
  });

  it('includes role instructions from definition', () => {
    const suffix = registry.buildSystemPromptSuffix('architect', {
      teamName: 'team',
      memberName: 'arch',
      fileScope: [],
    });

    expect(suffix).toContain('architect');
  });

  it('includes custom prompt when provided', () => {
    const suffix = registry.buildSystemPromptSuffix('implementer', {
      teamName: 'team',
      memberName: 'impl',
      fileScope: [],
      customPrompt: 'Focus on performance optimizations',
    });

    expect(suffix).toContain('Focus on performance optimizations');
  });

  it('includes communication instructions', () => {
    const suffix = registry.buildSystemPromptSuffix('implementer', {
      teamName: 'team',
      memberName: 'impl',
      fileScope: [],
    });

    expect(suffix).toContain('[MESSAGE: teammate-name]');
    expect(suffix).toContain('[BROADCAST]');
  });

  it('warns about file scope restrictions', () => {
    const suffix = registry.buildSystemPromptSuffix('implementer', {
      teamName: 'team',
      memberName: 'impl',
      fileScope: ['src/auth/**'],
    });

    expect(suffix).toContain('file scope');
    expect(suffix).toContain('src/auth/**');
  });
});

// ─── Tool allowlist resolution ────────────────────────────────────────────────

describe('AgentRegistry — resolveToolList', () => {
  const availableTools = [
    'read_file',
    'write_file',
    'run_shell',
    'search_files',
    'wiki_write',
  ];

  it('returns null when definition has no restrictions', () => {
    const def = makeCustomDef({ name: 'open', allowedTools: [], deniedTools: [] });
    registry.register(def);

    const tools = registry.resolveToolList('open', availableTools);
    expect(tools).toBeNull();
  });

  it('filters to allowedTools when set', () => {
    const def = makeCustomDef({
      name: 'restricted',
      allowedTools: ['read_file', 'search_files'],
      deniedTools: [],
    });
    registry.register(def);

    const tools = registry.resolveToolList('restricted', availableTools);
    expect(tools).toEqual(['read_file', 'search_files']);
  });

  it('removes deniedTools', () => {
    const def = makeCustomDef({
      name: 'no-shell',
      allowedTools: [],
      deniedTools: ['run_shell'],
    });
    registry.register(def);

    const tools = registry.resolveToolList('no-shell', availableTools);
    expect(tools).not.toContain('run_shell');
    expect(tools).toContain('read_file');
  });

  it('allowedTools takes precedence over deniedTools', () => {
    const def = makeCustomDef({
      name: 'conflict',
      allowedTools: ['read_file', 'write_file'],
      deniedTools: ['write_file'],
    });
    registry.register(def);

    const tools = registry.resolveToolList('conflict', availableTools);
    // allowed is applied first, so only read_file and write_file
    // then denied is removed
    expect(tools).toEqual(['read_file']);
  });
});

// ─── Markdown reporting ───────────────────────────────────────────────────────

describe('AgentRegistry — toMarkdown', () => {
  it('generates a Markdown table', () => {
    const md = registry.toMarkdown();
    expect(md).toContain('## Agent Registry');
    expect(md).toContain('| Name | Role |');
    expect(md).toContain('supervisor');
    expect(md).toContain('architect');
  });

  it('includes all registered definitions', () => {
    const custom = makeCustomDef({ name: 'my-custom' });
    registry.register(custom);

    const md = registry.toMarkdown();
    expect(md).toContain('my-custom');
  });
});

// ─── Retrieval helpers ────────────────────────────────────────────────────────

describe('AgentRegistry — retrieval', () => {
  it('listNames returns all registered names', () => {
    const names = registry.listNames();
    expect(names.length).toBe(8); // built-ins
    expect(names).toContain('supervisor');
  });

  it('getOrFallback returns name if found', () => {
    const def = registry.getOrFallback('architect', 'implementer');
    expect(def!.name).toBe('architect');
  });

  it('getOrFallback falls back to role if name not found', () => {
    const def = registry.getOrFallback('ghost', 'implementer');
    expect(def!.role).toBe('implementer');
  });
});
src/tests/AgentSpawner.test.ts
TypeScript

/**
 * AgentSpawner test suite
 * Run with: bun test src/tests/AgentSpawner.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test';
import { OrchestratorStore } from '../OrchestratorStore.js';
import { AgentRegistry } from '../AgentRegistry.js';
import { AgentSpawner } from '../AgentSpawner.js';
import type { CoreSession, SessionManagerLike, CoreSessionConfig } from '../AgentSpawner.js';
import type { SpawnRequest } from '../types/orchestrator.types.js';

// ─── Mock SessionManager ──────────────────────────────────────────────────────

class MockSessionManager implements SessionManagerLike {
  private sessions = new Map<string, CoreSession>();
  private nextId = 1;

  async create(config: CoreSessionConfig): Promise<CoreSession> {
    const session: CoreSession = {
      id: `session-${this.nextId++}`,
      modelConfig: config.modelConfig,
      workingDirectory: config.workingDirectory,
      createdAt: new Date().toISOString(),
    };
    this.sessions.set(session.id, session);
    return session;
  }

  async destroy(sessionId: string): Promise<void> {
    this.sessions.delete(sessionId);
  }

  get(sessionId: string): CoreSession | undefined {
    return this.sessions.get(sessionId);
  }

  reset(): void {
    this.sessions.clear();
    this.nextId = 1;
  }
}

// ─── Setup ────────────────────────────────────────────────────────────────────

let store: OrchestratorStore;
let registry: AgentRegistry;
let sessionManager: MockSessionManager;
let spawner: AgentSpawner;
let teamId: string;

beforeEach(() => {
  store = new OrchestratorStore(':memory:');
  registry = new AgentRegistry();
  sessionManager = new MockSessionManager();

  spawner = new AgentSpawner(
    store,
    registry,
    
