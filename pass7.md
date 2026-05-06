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



