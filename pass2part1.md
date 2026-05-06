Pass 2 of 18 — packages/core
Part 1 of 2: Package Config · All Types · EventBus · HooksRegistry · PermissionGate · ModelCapabilityRegistry
File Tree for Pass 2 (complete)
text

packages/core/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   ├── types/
│   │   ├── agent.types.ts
│   │   ├── tool.types.ts
│   │   ├── memory.types.ts
│   │   ├── provider.types.ts
│   │   ├── permission.types.ts
│   │   └── errors.types.ts
│   ├── EventBus.ts
│   ├── HooksRegistry.ts
│   ├── PermissionGate.ts
│   ├── ModelCapabilityRegistry.ts
│   ├── ProviderRouter.ts             ← Part 2
│   ├── AdaptiveCompactor.ts          ← Part 2
│   ├── TurnAssembler.ts              ← Part 2
│   ├── ToolRegistry.ts               ← Part 2
│   ├── SessionManager.ts             ← Part 2
│   ├── queryLoop.ts                  ← Part 2
│   └── buddy/
│       └── BuddyEngine.ts            ← Part 2
└── tests/
    ├── EventBus.test.ts              ← Part 2
    ├── HooksRegistry.test.ts         ← Part 2
    ├── PermissionGate.test.ts        ← Part 2
    ├── ToolRegistry.test.ts          ← Part 2
    ├── AdaptiveCompactor.test.ts     ← Part 2
    └── queryLoop.test.ts             ← Part 2
packages/core/package.json
JSON

{
  "name": "@locoworker/core",
  "version": "1.0.0",
  "description": "LocoWorker core agent engine — queryLoop, tools, permissions, providers",
  "license": "MIT",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./types": {
      "import": "./dist/types/index.js",
      "types": "./dist/types/index.d.ts"
    },
    "./buddy": {
      "import": "./dist/buddy/BuddyEngine.js",
      "types": "./dist/buddy/BuddyEngine.d.ts"
    }
  },
  "files": [
    "dist",
    "src"
  ],
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "dev": "tsc --project tsconfig.json --watch",
    "typecheck": "tsc --project tsconfig.json --noEmit",
    "test": "bun test",
    "test:watch": "bun test --watch",
    "test:coverage": "bun test --coverage",
    "eval": "bun run src/evals/run.ts",
    "lint": "biome check ./src ./tests",
    "lint:fix": "biome check --apply ./src ./tests",
    "clean": "rm -rf dist tsconfig.tsbuildinfo"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "0.36.3",
    "openai": "4.77.0",
    "zod": "3.24.1"
  },
  "devDependencies": {
    "@locoworker/shared": "workspace:*",
    "@types/node": "^22.10.0",
    "typescript": "^5.7.2"
  },
  "peerDependencies": {
    "typescript": ">=5.7.0"
  }
}
packages/core/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "tsBuildInfoFile": "./tsconfig.tsbuildinfo",
    "paths": {
      "@locoworker/shared": ["../shared/src/index.ts"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
packages/core/src/types/errors.types.ts
TypeScript

/**
 * Typed error classes for the LocoWorker core engine.
 * Never throw raw Error strings — always use these typed classes.
 */

export class LocoWorkerError extends Error {
  public readonly code: string
  public readonly context: Record<string, unknown>

  constructor(
    message: string,
    code: string,
    context: Record<string, unknown> = {}
  ) {
    super(message)
    this.name = 'LocoWorkerError'
    this.code = code
    this.context = context
    // Ensure proper prototype chain for instanceof checks
    Object.setPrototypeOf(this, new.target.prototype)
  }
}

export class PermissionDeniedError extends LocoWorkerError {
  constructor(
    public readonly tool: string,
    public readonly requiredLevel: string,
    public readonly currentLevel: string
  ) {
    super(
      `Permission denied: tool "${tool}" requires ${requiredLevel}, current max is ${currentLevel}`,
      'PERMISSION_DENIED',
      { tool, requiredLevel, currentLevel }
    )
    this.name = 'PermissionDeniedError'
  }
}

export class ToolNotFoundError extends LocoWorkerError {
  constructor(public readonly toolName: string) {
    super(
      `Tool not found: "${toolName}"`,
      'TOOL_NOT_FOUND',
      { toolName }
    )
    this.name = 'ToolNotFoundError'
  }
}

export class ToolTimeoutError extends LocoWorkerError {
  constructor(
    public readonly toolName: string,
    public readonly timeoutMs: number
  ) {
    super(
      `Tool "${toolName}" timed out after ${timeoutMs}ms`,
      'TOOL_TIMEOUT',
      { toolName, timeoutMs }
    )
    this.name = 'ToolTimeoutError'
  }
}

export class ToolExecutionError extends LocoWorkerError {
  constructor(
    public readonly toolName: string,
    public readonly cause: unknown
  ) {
    super(
      `Tool "${toolName}" execution failed: ${cause instanceof Error ? cause.message : String(cause)}`,
      'TOOL_EXECUTION_ERROR',
      { toolName, cause: String(cause) }
    )
    this.name = 'ToolExecutionError'
    if (cause instanceof Error) {
      this.stack = `${this.stack}\nCaused by: ${cause.stack}`
    }
  }
}

export class ProviderError extends LocoWorkerError {
  constructor(
    public readonly provider: string,
    public readonly modelId: string,
    public readonly cause: unknown,
    public readonly statusCode?: number
  ) {
    super(
      `Provider "${provider}" (${modelId}) failed: ${cause instanceof Error ? cause.message : String(cause)}`,
      'PROVIDER_ERROR',
      { provider, modelId, statusCode, cause: String(cause) }
    )
    this.name = 'ProviderError'
  }
}

export class ProviderRateLimitError extends LocoWorkerError {
  constructor(
    public readonly provider: string,
    public readonly retryAfterMs?: number
  ) {
    super(
      `Provider "${provider}" rate limit exceeded${retryAfterMs ? ` — retry after ${retryAfterMs}ms` : ''}`,
      'PROVIDER_RATE_LIMIT',
      { provider, retryAfterMs }
    )
    this.name = 'ProviderRateLimitError'
  }
}

export class CostCapExceededError extends LocoWorkerError {
  constructor(
    public readonly capUsd: number,
    public readonly currentUsd: number,
    public readonly capType: 'session' | 'daily' | 'eval'
  ) {
    super(
      `${capType} cost cap of $${capUsd.toFixed(4)} exceeded (current: $${currentUsd.toFixed(4)})`,
      'COST_CAP_EXCEEDED',
      { capUsd, currentUsd, capType }
    )
    this.name = 'CostCapExceededError'
  }
}

export class MaxTurnsExceededError extends LocoWorkerError {
  constructor(
    public readonly maxTurns: number,
    public readonly sessionId: string
  ) {
    super(
      `Session "${sessionId}" exceeded maximum turns (${maxTurns})`,
      'MAX_TURNS_EXCEEDED',
      { maxTurns, sessionId }
    )
    this.name = 'MaxTurnsExceededError'
  }
}

export class WorkspaceBoundaryError extends LocoWorkerError {
  constructor(
    public readonly attemptedPath: string,
    public readonly boundary: string
  ) {
    super(
      `Path "${attemptedPath}" is outside workspace boundary "${boundary}"`,
      'WORKSPACE_BOUNDARY_VIOLATION',
      { attemptedPath, boundary }
    )
    this.name = 'WorkspaceBoundaryError'
  }
}

export class InjectionDetectedError extends LocoWorkerError {
  constructor(
    public readonly source: string,
    public readonly pattern: string
  ) {
    super(
      `Potential prompt injection detected in "${source}": matched pattern "${pattern}"`,
      'INJECTION_DETECTED',
      { source, pattern }
    )
    this.name = 'InjectionDetectedError'
  }
}

export class SessionNotFoundError extends LocoWorkerError {
  constructor(public readonly sessionId: string) {
    super(
      `Session not found: "${sessionId}"`,
      'SESSION_NOT_FOUND',
      { sessionId }
    )
    this.name = 'SessionNotFoundError'
  }
}

export class ContextBudgetError extends LocoWorkerError {
  constructor(
    public readonly tokenCount: number,
    public readonly maxTokens: number
  ) {
    super(
      `Context budget exhausted: ${tokenCount} tokens exceeds max ${maxTokens}`,
      'CONTEXT_BUDGET_EXHAUSTED',
      { tokenCount, maxTokens }
    )
    this.name = 'ContextBudgetError'
  }
}
packages/core/src/types/permission.types.ts
TypeScript

/**
 * Permission tier definitions for the LocoWorker PermissionGate.
 */

/**
 * The five permission tiers, in ascending order of privilege.
 * Each tier includes all privileges of lower tiers.
 */
export type PermissionLevel =
  | 'READ_ONLY'    // Tier 0: read files, search, list, query
  | 'WRITE_LOCAL'  // Tier 1: write/edit/delete files within workspace
  | 'NETWORK'      // Tier 2: web requests, MCP, messaging
  | 'SHELL'        // Tier 3: arbitrary shell command execution
  | 'DANGEROUS'    // Tier 4: system-wide, outside workspace, credentials

/**
 * Numeric rank for each permission level — used for comparison.
 */
export const PERMISSION_RANK: Readonly<Record<PermissionLevel, number>> = {
  READ_ONLY: 0,
  WRITE_LOCAL: 1,
  NETWORK: 2,
  SHELL: 3,
  DANGEROUS: 4,
} as const

/**
 * Configuration for a session's permission boundaries.
 */
export interface PermissionSet {
  /** The highest permission level allowed in this session */
  maxLevel: PermissionLevel
  /** Specific tool names always allowed, regardless of tier */
  allowlist?: string[]
  /** Specific tool names always denied, regardless of tier */
  denylist?: string[]
  /** Permission levels that require interactive user confirmation before execution */
  requireConfirmation?: PermissionLevel[]
  /** Absolute filesystem path — write operations must stay within this boundary */
  workspaceBoundary?: string
  /** If true, log all tool calls to the audit log */
  auditAllCalls?: boolean
}

/**
 * Context passed to the PermissionGate.check() method.
 */
export interface PermissionCheckPayload {
  /** The tool name being checked */
  tool: string
  /** The raw input being passed to the tool (for logging) */
  input: unknown
  /** Optional file path being targeted (for workspace boundary check) */
  targetPath?: string
}

/**
 * Result of a permission check.
 */
export interface PermissionCheckResult {
  permitted: boolean
  reason?: string
  requiresConfirmation?: boolean
}

/**
 * Default permission sets for common use cases.
 */
export const DEFAULT_PERMISSION_SETS = {
  /** Safe for untrusted codebases — read only */
  READ_ONLY: {
    maxLevel: 'READ_ONLY' as PermissionLevel,
    auditAllCalls: true,
  } satisfies PermissionSet,

  /** Standard development session */
  STANDARD: {
    maxLevel: 'WRITE_LOCAL' as PermissionLevel,
    requireConfirmation: [],
    auditAllCalls: false,
  } satisfies PermissionSet,

  /** Full developer mode with network */
  DEVELOPER: {
    maxLevel: 'NETWORK' as PermissionLevel,
    requireConfirmation: ['SHELL'] as PermissionLevel[],
    auditAllCalls: false,
  } satisfies PermissionSet,

  /** Power user — all tiers with confirmation for dangerous */
  POWER: {
    maxLevel: 'SHELL' as PermissionLevel,
    requireConfirmation: ['DANGEROUS'] as PermissionLevel[],
    auditAllCalls: true,
  } satisfies PermissionSet,
} as const
packages/core/src/types/tool.types.ts
TypeScript

import type { JSONSchema7 } from 'json-schema'
import type { PermissionLevel } from './permission.types.js'

/**
 * The result of a tool execution.
 */
export interface ToolCallResult {
  /** Matches the toolCall.id from AssistantMessage */
  toolCallId: string
  /** Tool name for logging/display */
  toolName: string
  /** The result content — string for text, object for structured */
  content: string | Record<string, unknown>
  /** Whether this result represents an error condition */
  isError: boolean
  /** Estimated token count of this result */
  tokenCount: number
  /** Execution duration in milliseconds */
  durationMs: number
  /** Metadata for audit/memory purposes */
  metadata?: Record<string, unknown>
}

/**
 * A tool call as requested by the model.
 */
export interface ToolCall {
  /** Unique ID for this tool call instance */
  id: string
  /** The registered tool name */
  name: string
  /** Parsed input arguments */
  input: Record<string, unknown>
}

/**
 * Definition of a registered tool.
 */
export interface ToolDefinition {
  /** Unique tool name — must be snake_case */
  name: string
  /** Human-readable description for the model */
  description: string
  /** JSON Schema for the tool's input */
  inputSchema: JSONSchema7
  /** Whether this tool can be executed concurrently with other tools */
  parallelSafe: boolean
  /** Minimum permission tier required to execute this tool */
  requiredPermission: PermissionLevel
  /** Execution timeout in milliseconds (default: 30000) */
  timeoutMs?: number
  /** Whether the tool can be automatically retried on transient failure */
  retryable?: boolean
  /** Whether this tool modifies state (used for compaction decisions) */
  stateful?: boolean
  /** Category tags for grouping and filtering */
  tags?: string[]
  /** The actual implementation */
  handler: ToolHandler
}

/**
 * The handler function for a tool.
 */
export type ToolHandler = (
  input: Record<string, unknown>,
  context: import('./agent.types.js').AgentContext
) => Promise<ToolCallResult>

/**
 * A lightweight tool descriptor for model consumption (no handler).
 */
export type ToolDescriptor = Omit<ToolDefinition, 'handler'>

/**
 * Format for sending tools to the Anthropic API.
 */
export interface AnthropicToolFormat {
  name: string
  description: string
  input_schema: JSONSchema7
}

/**
 * Format for sending tools to the OpenAI API.
 */
export interface OpenAIToolFormat {
  type: 'function'
  function: {
    name: string
    description: string
    parameters: JSONSchema7
  }
}
packages/core/src/types/provider.types.ts
TypeScript

/**
 * Provider and model configuration types.
 */

export type ProviderId =
  | 'anthropic'
  | 'openai'
  | 'google'
  | 'mistral'
  | 'groq'
  | 'together'
  | 'ollama'
  | 'lmstudio'
  | 'llamacpp'
  | 'custom'

/**
 * Configuration for a specific model + provider combination.
 */
export interface ModelConfig {
  /** Provider identifier */
  provider: ProviderId
  /** Model identifier (e.g., 'claude-opus-4-5', 'llama3.1:8b') */
  modelId: string
  /** API key — undefined for local/no-auth providers */
  apiKey?: string
  /** Base URL override for local or custom providers */
  baseUrl?: string
  /** Maximum tokens to generate in a response */
  maxTokens?: number
  /** Temperature 0-1 (0 = deterministic, 1 = creative) */
  temperature?: number
  /** Whether to use streaming responses */
  streaming?: boolean
  /** Request timeout in milliseconds */
  timeoutMs?: number
}

/**
 * Context budget profile — controls compaction thresholds.
 */
export interface ContextBudgetProfile {
  modelId: string
  /** Absolute maximum context tokens this model supports */
  maxContextTokens: number
  /** Fraction (0-1) at which auto compaction is triggered */
  softLimitFraction: number
  /** Fraction (0-1) at which full/emergency compaction is triggered */
  hardLimitFraction: number
  /** Tokens reserved for tool results (excluded from history budget) */
  reservedForTools: number
  /** Tokens reserved for model output (excluded from input budget) */
  reservedForOutput: number
  /** Default compaction mode for this model */
  compactionMode: CompactionMode
  /** VRAM awareness for local models */
  vramConfig?: VRAMConfig
}

/**
 * Compaction mode — controls how aggressively history is compressed.
 */
export type CompactionMode = 'micro' | 'auto' | 'full' | 'aggressive' | 'disabled'

/**
 * VRAM configuration for local model context management.
 */
export interface VRAMConfig {
  /** Available VRAM in GB */
  availableGB: number
  /** Estimated model size in GB */
  modelSizeGB: number
  /** Safe utilization rate (0-1, default 0.85) */
  safeUtilizationRate: number
}

/**
 * Capabilities of a registered model.
 */
export interface ModelCapability {
  modelId: string
  provider: ProviderId
  contextWindow: number
  supportsStreaming: boolean
  supportsToolCalling: boolean
  supportsParallelTools: boolean
  supportsVision: boolean
  supportsJsonMode: boolean
  isLocalModel: boolean
  estimatedVRAMGB?: number
  /** Cost per 1M input tokens in USD (undefined for local models) */
  costPer1MInputTokens?: number
  /** Cost per 1M output tokens in USD (undefined for local models) */
  costPer1MOutputTokens?: number
  /** Key to look up in BUILT_IN_BUDGET_PROFILES */
  preferredBudgetProfileKey: string
  notes?: string
}

/**
 * A provider descriptor for the UI registry.
 */
export interface ProviderConfig {
  id: ProviderId
  displayName: string
  apiKeyRequired: boolean
  isLocal: boolean
  defaultBaseUrl?: string
  defaultModelId: string
  models: ModelEntry[]
  status: 'available' | 'unavailable' | 'unconfigured'
}

/**
 * A model entry within a ProviderConfig.
 */
export interface ModelEntry {
  modelId: string
  displayName: string
  contextWindow: number
  costPer1MInput?: number
  costPer1MOutput?: number
  isDefault?: boolean
  capabilities: string[]
}

/**
 * Routing rules for multi-provider fallback.
 */
export interface ProviderRoutingConfig {
  /** Primary model to use */
  primary: ModelConfig
  /** Fallback model if primary fails */
  fallback?: ModelConfig
  /** Maximum cost per session before switching to fallback (USD) */
  sessionCostCapUsd?: number
  /** Maximum cost per day (USD) */
  dailyCostCapUsd?: number
  /** If true and on battery power, prefer local providers */
  preferLocalOnBattery?: boolean
}

/**
 * Usage statistics returned from a model call.
 */
export interface ModelUsage {
  inputTokens: number
  outputTokens: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  /** Computed cost in USD */
  costUsd?: number
}
packages/core/src/types/memory.types.ts
TypeScript

/**
 * Memory-related types used by the MemoryManager and AutoDream.
 * Defined here so core can reference them without importing packages/memory.
 */

export interface MemoryEntry {
  id: string
  timestamp: number
  sessionId: string
  category: MemoryCategory
  content: string
  importance: MemoryImportance
  tags: string[]
  /** Optional link to a Graphify graph node */
  graphNodeId?: string
  /** Optional link to a LLMWiki entry slug */
  wikiSlug?: string
  /** Entries older than this timestamp can be pruned */
  expiresAt?: number
}

export type MemoryCategory =
  | 'architecture'
  | 'preference'
  | 'file_location'
  | 'decision'
  | 'issue'
  | 'workaround'
  | 'research'
  | 'buddy'
  | 'credential'   // Reminder that key exists, NOT the key itself
  | 'note'

export type MemoryImportance = 'low' | 'medium' | 'high' | 'critical'

/**
 * The in-memory state of a session's memory layers.
 */
export interface MemoryState {
  /** Layer 1: System prompt (immutable during session) */
  systemPrompt: string
  /** Layer 2: Project instructions from CLAUDE.md */
  projectInstructions: string
  /** Layer 4: Persistent cross-session index from MEMORY.md */
  persistentIndex: string
  /** Approximate token count of layers 1+2+4 (excludes history) */
  baselineTokenCount: number
  /** Whether MEMORY.md has been loaded this session */
  loaded: boolean
  /** Path to MEMORY.md file */
  memoryFilePath: string
}

/**
 * Payload for a memory update after a significant turn.
 */
export interface MemoryUpdatePayload {
  turn: ConversationTurn | ToolCallResult[]
  response: AssistantMessage
  toolResults?: ToolCallResult[]
  sessionId: string
  workingDirectory: string
}

// Forward declarations to avoid circular imports
import type { ConversationTurn, AssistantMessage } from './agent.types.js'
import type { ToolCallResult } from './tool.types.js'
packages/core/src/types/agent.types.ts
TypeScript

import type { MemoryState } from './memory.types.js'
import type { PermissionSet } from './permission.types.js'
import type { ContextBudgetProfile, ModelConfig, ModelUsage } from './provider.types.js'

/**
 * The central context object passed through the agent loop.
 * Every module reads from and writes to this object.
 */
export interface AgentContext {
  /** Unique session identifier */
  sessionId: string
  /** Absolute path to the project workspace root */
  workingDirectory: string
  /** Model configuration for this session */
  model: ModelConfig
  /** Context budget profile (token limits, compaction thresholds) */
  budget: ContextBudgetProfile
  /** Permission configuration for this session */
  permissions: PermissionSet
  /** Current memory state (all 4 layers) */
  memory: MemoryState
  /** Session-level metadata — arbitrary key/value pairs */
  metadata: AgentMetadata
}

/**
 * Session metadata — tracks runtime state and configuration.
 */
export interface AgentMetadata {
  /** Maximum turns before the session is force-terminated */
  maxTurns: number
  /** Current turn counter */
  turnCount: number
  /** Whether this is a sub-agent spawned by an orchestrator */
  isSubAgent: boolean
  /** Parent session ID if this is a sub-agent */
  parentSessionId?: string
  /** Task description if this is a sub-agent */
  taskDescription?: string
  /** Git worktree ID if this session has an isolated worktree */
  worktreeId?: string
  /** Whether the session is in "undercover" mode */
  undercoverMode: boolean
  /** Running total cost for this session in USD */
  sessionCostUsd: number
  /** Session start timestamp */
  startedAt: number
  /** User-defined extra properties */
  [key: string]: unknown
}

/**
 * Events emitted by the agent loop — consumed by UI, logging, KAIROS.
 */
export type AgentEvent =
  | TurnStartEvent
  | ModelRequestEvent
  | ModelResponseEvent
  | ToolCallEvent
  | ToolResultEvent
  | CompactionTriggeredEvent
  | TurnCompleteEvent
  | SessionErrorEvent
  | PermissionDeniedEvent
  | SessionStartEvent
  | SessionEndEvent

export interface BaseEvent {
  sessionId: string
  timestamp: number
}

export interface TurnStartEvent extends BaseEvent {
  type: 'turn_start'
  data: { turnNumber: number; input: UserMessage | ToolCallResult[] }
}

export interface ModelRequestEvent extends BaseEvent {
  type: 'model_request'
  data: { modelId: string; provider: string; estimatedTokens: number }
}

export interface ModelResponseEvent extends BaseEvent {
  type: 'model_response'
  data: { response: AssistantMessage; usage: ModelUsage }
}

export interface ToolCallEvent extends BaseEvent {
  type: 'tool_call'
  data: { toolCall: ToolCall; permissionLevel: string }
}

export interface ToolResultEvent extends BaseEvent {
  type: 'tool_result'
  data: ToolCallResult
}

export interface CompactionTriggeredEvent extends BaseEvent {
  type: 'compaction_triggered'
  data: {
    mode: import('./provider.types.js').CompactionMode
    tokensBefore: number
    tokensAfter?: number
  }
}

export interface TurnCompleteEvent extends BaseEvent {
  type: 'turn_complete'
  data: { response: AssistantMessage; totalTurns: number; costUsd: number }
}

export interface SessionErrorEvent extends BaseEvent {
  type: 'session_error'
  data: { error: unknown; phase: string; fatal: boolean }
}

export interface PermissionDeniedEvent extends BaseEvent {
  type: 'permission_denied'
  data: { tool: string; requiredLevel: string; currentLevel: string }
}

export interface SessionStartEvent extends BaseEvent {
  type: 'session_start'
  data: { workingDirectory: string; model: string; permissionLevel: string }
}

export interface SessionEndEvent extends BaseEvent {
  type: 'session_end'
  data: { totalTurns: number; totalCostUsd: number; durationMs: number; reason: SessionEndReason }
}

export type SessionEndReason =
  | 'complete'
  | 'max_turns_exceeded'
  | 'cost_cap_exceeded'
  | 'provider_error'
  | 'user_cancelled'
  | 'session_timeout'

/**
 * A message from the user to the agent.
 */
export interface UserMessage {
  role: 'user'
  content: string | ContentBlock[]
}

/**
 * A message from the assistant (model response).
 */
export interface AssistantMessage {
  role: 'assistant'
  /** Text content — empty string if only tool calls */
  content: string
  /** Tool calls requested by the model */
  toolCalls?: ToolCall[]
  /** Usage statistics for cost tracking */
  usage?: ModelUsage
}

/**
 * A turn in the conversation history.
 */
export interface ConversationTurn {
  role: 'user' | 'assistant'
  content: string
  /** Estimated token count */
  tokenCount: number
  /** Importance rating affects compaction behaviour */
  importance: TurnImportance
  /** Unix timestamp */
  timestamp: number
  /** Tool calls made in this turn (if assistant) */
  toolCalls?: ToolCall[]
  /** Tool results injected in this turn (if user/tool result) */
  toolResults?: ToolCallResult[]
}

export type TurnImportance = 'normal' | 'important' | 'critical'

/**
 * A content block for multi-modal messages.
 */
export type ContentBlock =
  | { type: 'text'; text: string }
  | { type: 'image'; source: ImageSource }

export interface ImageSource {
  type: 'base64' | 'url'
  mediaType?: 'image/jpeg' | 'image/png' | 'image/gif' | 'image/webp'
  data?: string
  url?: string
}

/**
 * An assembled context ready to send to the model.
 */
export interface AssembledContext {
  /** Full system prompt (layers 1 + 2 + 4 merged) */
  systemPrompt: string
  /** Conversation history (layer 3, possibly compacted) */
  history: ConversationTurn[]
  /** Tool results from the previous turn */
  toolResults: ToolCallResult[]
  /** Current user input or next turn content */
  currentInput: UserMessage | ToolCallResult[]
  /** Total estimated token count across all layers */
  tokenCount: number
  /** Tool descriptors to send to the model */
  tools: ToolDescriptor[]
}

// Forward declarations
import type { ToolCall, ToolCallResult, ToolDescriptor } from './tool.types.js'
packages/core/src/types/index.ts
TypeScript

/**
 * Barrel export for all core types.
 * Import from '@locoworker/core/types' for type-only imports.
 */

export type * from './agent.types.js'
export type * from './errors.types.js'
export type * from './memory.types.js'
export type * from './permission.types.js'
export type * from './provider.types.js'
export type * from './tool.types.js'

// Re-export const values (not just types)
export { PERMISSION_RANK, DEFAULT_PERMISSION_SETS } from './permission.types.js'
packages/core/src/EventBus.ts
TypeScript

import type { AgentEvent } from './types/agent.types.js'

/**
 * A typed, async event bus for intra-session communication.
 *
 * Handlers are invoked concurrently via Promise.allSettled — a slow
 * or failing handler never blocks other handlers or the agent loop.
 *
 * @example
 * ```ts
 * const bus = new EventBus()
 *
 * const unsubscribe = bus.on('tool_result', async (event) => {
 *   console.log('Tool result:', event.data)
 * })
 *
 * await bus.emit({ type: 'tool_result', sessionId: 's_1', timestamp: Date.now(), data: ... })
 *
 * unsubscribe() // clean up
 * ```
 */
export class EventBus {
  private readonly listeners = new Map<string, Set<EventHandler>>()
  private readonly wildcardListeners = new Set<EventHandler>()

  /**
   * Subscribe to a specific event type.
   * Returns an unsubscribe function.
   */
  on<T extends AgentEvent>(
    eventType: T['type'],
    handler: (event: T) => void | Promise<void>
  ): UnsubscribeFn {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set())
    }

    const typedHandler = handler as EventHandler
    this.listeners.get(eventType)!.add(typedHandler)

    return () => {
      this.listeners.get(eventType)?.delete(typedHandler)
    }
  }

  /**
   * Subscribe to ALL events (wildcard listener).
   * Returns an unsubscribe function.
   */
  onAny(handler: (event: AgentEvent) => void | Promise<void>): UnsubscribeFn {
    const typedHandler = handler as EventHandler
    this.wildcardListeners.add(typedHandler)
    return () => {
      this.wildcardListeners.delete(typedHandler)
    }
  }

  /**
   * Subscribe to an event type exactly once.
   * Automatically unsubscribes after the first emission.
   */
  once<T extends AgentEvent>(
    eventType: T['type'],
    handler: (event: T) => void | Promise<void>
  ): UnsubscribeFn {
    const unsubscribe = this.on<T>(eventType, (event) => {
      unsubscribe()
      return handler(event)
    })
    return unsubscribe
  }

  /**
   * Emit an event to all subscribers.
   * All handlers are invoked concurrently. Errors are caught and logged.
   */
  async emit(event: AgentEvent): Promise<void> {
    const typeListeners = this.listeners.get(event.type) ?? new Set()
    const allHandlers = [...typeListeners, ...this.wildcardListeners]

    if (allHandlers.length === 0) return

    const results = await Promise.allSettled(
      allHandlers.map((handler) => handler(event))
    )

    // Log errors from handlers — never let them propagate to the agent loop
    for (const result of results) {
      if (result.status === 'rejected') {
        console.error(
          `[EventBus] Handler error for event "${event.type}":`,
          result.reason
        )
      }
    }
  }

  /**
   * Remove all listeners for a specific event type.
   */
  off(eventType: AgentEvent['type']): void {
    this.listeners.get(eventType)?.clear()
  }

  /**
   * Remove all listeners for all event types.
   */
  offAll(): void {
    this.listeners.clear()
    this.wildcardListeners.clear()
  }

  /**
   * Returns the number of listeners for a given event type.
   * Useful for testing and debugging.
   */
  listenerCount(eventType: AgentEvent['type']): number {
    return this.listeners.get(eventType)?.size ?? 0
  }

  /**
   * Returns true if there are any listeners for the given event type.
   */
  hasListeners(eventType: AgentEvent['type']): boolean {
    return this.listenerCount(eventType) > 0
  }
}

type EventHandler = (event: AgentEvent) => void | Promise<void>
type UnsubscribeFn = () => void
packages/core/src/HooksRegistry.ts
TypeScript

import type { AgentContext } from './types/agent.types.js'
import type { AssembledContext, ConversationTurn } from './types/agent.types.js'
import type { ToolCall, ToolCallResult } from './types/tool.types.js'

/**
 * All lifecycle hook names available in the agent loop.
 */
export type HookName =
  | 'beforeTurn'          // Before each loop iteration — can modify AssembledContext
  | 'afterTurn'           // After model responds — can inspect/log response
  | 'beforeToolCall'      // Before each tool execution — can block/modify
  | 'afterToolCall'       // After each tool execution — can inspect result
  | 'beforeCompaction'    // Before context compaction — can influence mode
  | 'afterCompaction'     // After compaction — receives token delta
  | 'onSessionStart'      // When a new session is initialized
  | 'onSessionEnd'        // When a session terminates (any reason)
  | 'onPermissionDenied'  // When a tool call is blocked by PermissionGate
  | 'onModelResponse'     // After every model response (alias for afterTurn)
  | 'onCostUpdate'        // When session cost is updated
  | 'onError'             // On any unhandled error in the agent loop

/**
 * Payload types for each hook.
 */
export interface HookPayloads {
  beforeTurn: AssembledContext
  afterTurn: ConversationTurn
  beforeToolCall: ToolCall
  afterToolCall: { toolCall: ToolCall; result: ToolCallResult }
  beforeCompaction: { mode: string; tokenCount: number }
  afterCompaction: { mode: string; tokensBefore: number; tokensAfter: number }
  onSessionStart: { sessionId: string; workingDirectory: string }
  onSessionEnd: { sessionId: string; totalTurns: number; costUsd: number }
  onPermissionDenied: { tool: string; requiredLevel: string }
  onModelResponse: ConversationTurn
  onCostUpdate: { sessionCostUsd: number; deltaCostUsd: number }
  onError: { error: unknown; phase: string }
}

/**
 * A hook handler function. It receives the payload and the agent context.
 * Optionally returns a modified payload (for transforming hooks like beforeTurn).
 */
export type HookHandler<T> = (
  payload: T,
  context: AgentContext
) => Promise<T | void> | T | void

/**
 * A registered hook with metadata.
 */
interface RegisteredHook<T> {
  id: string
  name: HookName
  handler: HookHandler<T>
  priority: number  // Lower number = runs first (default: 100)
}

/**
 * Registry for plugin-style lifecycle hooks that extend agent behaviour.
 *
 * Hooks run in priority order. A hook can return a modified payload
 * to transform the data flowing through the pipeline.
 *
 * @example
 * ```ts
 * const hooks = new HooksRegistry()
 *
 * hooks.register('beforeToolCall', async (toolCall, ctx) => {
 *   console.log(`About to call: ${toolCall.name}`)
 *   // Return nothing to pass through unchanged
 *   // Return modified toolCall to change what gets executed
 * }, { priority: 10 })
 * ```
 */
export class HooksRegistry {
  // biome-ignore noExplicitAny: heterogeneous hook map requires any
  private readonly hooks = new Map<HookName, RegisteredHook<any>[]>()

  /**
   * Register a hook handler.
   *
   * @param name - The lifecycle hook to subscribe to
   * @param handler - The handler function
   * @param options - Optional priority (lower = earlier) and ID
   * @returns A deregister function
   */
  register<K extends HookName>(
    name: K,
    handler: HookHandler<HookPayloads[K]>,
    options: { priority?: number; id?: string } = {}
  ): DeregisterFn {
    const { priority = 100, id = crypto.randomUUID() } = options

    if (!this.hooks.has(name)) {
      this.hooks.set(name, [])
    }

    const hook: RegisteredHook<HookPayloads[K]> = { id, name, handler, priority }
    const list = this.hooks.get(name)!
    list.push(hook)
    // Keep sorted by priority ascending (lower priority number = runs first)
    list.sort((a, b) => a.priority - b.priority)

    return () => {
      const current = this.hooks.get(name)
      if (current) {
        const idx = current.findIndex((h) => h.id === id)
        if (idx !== -1) current.splice(idx, 1)
      }
    }
  }

  /**
   * Run all registered handlers for a hook in priority order.
   * Each handler's return value (if any) replaces the payload for the next handler.
   *
   * @param name - The hook to run
   * @param payload - The initial payload
   * @param context - The current agent context
   * @returns The final (possibly transformed) payload
   */
  async run<K extends HookName>(
    name: K,
    payload: HookPayloads[K],
    context: AgentContext
  ): Promise<HookPayloads[K]> {
    const handlers = this.hooks.get(name) ?? []
    if (handlers.length === 0) return payload

    let current: HookPayloads[K] = payload

    for (const hook of handlers) {
      try {
        const result = await hook.handler(current, context)
        if (result !== undefined && result !== null) {
          current = result as HookPayloads[K]
        }
      } catch (err) {
        // Hook errors must never crash the agent loop
        console.error(`[HooksRegistry] Error in hook "${name}" (id: ${hook.id}):`, err)
      }
    }

    return current
  }

  /**
   * Check if any handlers are registered for a hook.
   */
  hasHandlers(name: HookName): boolean {
    return (this.hooks.get(name)?.length ?? 0) > 0
  }

  /**
   * Get the count of registered handlers for a hook.
   */
  handlerCount(name: HookName): number {
    return this.hooks.get(name)?.length ?? 0
  }

  /**
   * Remove all handlers for a specific hook.
   */
  clear(name: HookName): void {
    this.hooks.get(name)?.splice(0)
  }

  /**
   * Remove all handlers for all hooks.
   */
  clearAll(): void {
    this.hooks.clear()
  }
}

type DeregisterFn = () => void
packages/core/src/PermissionGate.ts
TypeScript

import {
  PERMISSION_RANK,
  type PermissionCheckPayload,
  type PermissionCheckResult,
  type PermissionLevel,
  type PermissionSet,
} from './types/permission.types.js'
import {
  PermissionDeniedError,
  WorkspaceBoundaryError,
} from './types/errors.types.js'
import { resolve, normalize } from 'node:path'

/**
 * Interactive confirmation callback — used for SHELL/DANGEROUS tiers.
 * Returns true if the user approved the action.
 */
export type ConfirmFn = (message: string, context: Record<string, unknown>) => Promise<boolean>

/**
 * The PermissionGate enforces the 5-tier permission system on every tool call.
 *
 * It is the critical security layer between the agent loop and tool execution.
 * All tool calls MUST pass through PermissionGate.check() before execution.
 *
 * @example
 * ```ts
 * const gate = new PermissionGate(
 *   { maxLevel: 'WRITE_LOCAL', workspaceBoundary: '/home/user/project' },
 *   async (msg) => { return await promptUser(msg) }
 * )
 *
 * const result = await gate.check('SHELL', {
 *   tool: 'bash',
 *   input: { command: 'rm -rf /' }
 * })
 *
 * if (!result.permitted) throw new PermissionDeniedError(...)
 * ```
 */
export class PermissionGate {
  constructor(
    private readonly permSet: PermissionSet,
    private readonly confirmFn: ConfirmFn = async () => false
  ) {}

  /**
   * Check whether a tool call is permitted under the current permission set.
   * This is the primary entry point — called before every tool execution.
   *
   * Priority order:
   *  1. Denylist (always block)
   *  2. Allowlist (always permit)
   *  3. Tier rank check
   *  4. Confirmation check (if tier requires it)
   *  5. Workspace boundary check (if targetPath provided)
   */
  async check(
    requiredLevel: PermissionLevel,
    payload: PermissionCheckPayload
  ): Promise<PermissionCheckResult> {
    // ── 1. Explicit denylist always wins ──────────────────────────────
    if (this.permSet.denylist?.includes(payload.tool)) {
      return {
        permitted: false,
        reason: `Tool "${payload.tool}" is on the denylist`,
      }
    }

    // ── 2. Explicit allowlist always permits (skips tier check) ───────
    if (this.permSet.allowlist?.includes(payload.tool)) {
      // Still apply workspace boundary if path provided
      if (payload.targetPath) {
        const boundaryCheck = this.checkWorkspaceBoundary(payload.targetPath)
        if (!boundaryCheck.permitted) return boundaryCheck
      }
      return { permitted: true }
    }

    // ── 3. Tier rank check ────────────────────────────────────────────
    const requiredRank = PERMISSION_RANK[requiredLevel]
    const maxRank = PERMISSION_RANK[this.permSet.maxLevel]

    if (requiredRank > maxRank) {
      return {
        permitted: false,
        reason:
          `Tool "${payload.tool}" requires ${requiredLevel} (rank ${requiredRank}), ` +
          `but session max is ${this.permSet.maxLevel} (rank ${maxRank})`,
      }
    }

    // ── 4. Confirmation check ─────────────────────────────────────────
    if (this.permSet.requireConfirmation?.includes(requiredLevel)) {
      const approved = await this.confirmFn(
        `Tool "${payload.tool}" requires ${requiredLevel} permission.\n` +
        `Input: ${JSON.stringify(payload.input, null, 2)}\n\n` +
        `Allow this operation?`,
        { tool: payload.tool, level: requiredLevel, input: payload.input }
      )

      if (!approved) {
        return {
          permitted: false,
          reason: `User declined ${requiredLevel} permission for "${payload.tool}"`,
          requiresConfirmation: true,
        }
      }
    }

    // ── 5. Workspace boundary check ───────────────────────────────────
    if (payload.targetPath) {
      const boundaryCheck = this.checkWorkspaceBoundary(payload.targetPath)
      if (!boundaryCheck.permitted) return boundaryCheck
    }

    return { permitted: true }
  }

  /**
   * Pre-check a list of anticipated tool names before the turn begins.
   * Throws if any tool is on the denylist (fail-fast).
   */
  precheck(anticipatedTools: string[]): void {
    for (const tool of anticipatedTools) {
      if (this.permSet.denylist?.includes(tool)) {
        throw new PermissionDeniedError(tool, 'ANY', 'DENIED — on denylist')
      }
    }
  }

  /**
   * Check whether a file path is within the workspace boundary.
   * Returns a PermissionCheckResult for consistency.
   */
  checkWorkspaceBoundary(targetPath: string): PermissionCheckResult {
    if (!this.permSet.workspaceBoundary) {
      return { permitted: true }
    }

    const resolvedTarget = normalize(resolve(targetPath))
    const resolvedBoundary = normalize(resolve(this.permSet.workspaceBoundary))

    if (!resolvedTarget.startsWith(resolvedBoundary)) {
      return {
        permitted: false,
        reason:
          `Path "${resolvedTarget}" is outside workspace boundary "${resolvedBoundary}"`,
      }
    }

    return { permitted: true }
  }

  /**
   * Convenience method: throw if not permitted.
   * Use in tool handlers for clean error propagation.
   */
  async assertPermitted(
    requiredLevel: PermissionLevel,
    payload: PermissionCheckPayload
  ): Promise<void> {
    const result = await this.check(requiredLevel, payload)
    if (!result.permitted) {
      throw new PermissionDeniedError(
        payload.tool,
        requiredLevel,
        this.permSet.maxLevel
      )
    }
  }

  /**
   * Convenience: assert workspace boundary or throw.
   */
  assertWorkspaceBoundary(targetPath: string): void {
    const result = this.checkWorkspaceBoundary(targetPath)
    if (!result.permitted && this.permSet.workspaceBoundary) {
      throw new WorkspaceBoundaryError(targetPath, this.permSet.workspaceBoundary)
    }
  }

  /**
   * Returns the current max permission level for this session.
   */
  get maxLevel(): PermissionLevel {
    return this.permSet.maxLevel
  }

  /**
   * Returns the current permission set (read-only).
   */
  get config(): Readonly<PermissionSet> {
    return this.permSet
  }

  /**
   * Create a PermissionGate with a non-interactive confirm function.
   * Used in tests and non-interactive sessions.
   */
  static withAutoDecline(permSet: PermissionSet): PermissionGate {
    return new PermissionGate(permSet, async () => false)
  }

  /**
   * Create a PermissionGate with auto-approve confirmation.
   * WARNING: Only use in trusted automated contexts.
   */
  static withAutoApprove(permSet: PermissionSet): PermissionGate {
    return new PermissionGate(permSet, async () => true)
  }
}
packages/core/src/ModelCapabilityRegistry.ts
TypeScript

import type {
  ContextBudgetProfile,
  ModelCapability,
  ModelConfig,
  ProviderId,
} from './types/provider.types.js'

/**
 * Built-in context budget profiles for known models.
 * Keys match ModelCapability.preferredBudgetProfileKey.
 */
export const BUILT_IN_BUDGET_PROFILES: Readonly<Record<string, ContextBudgetProfile>> = {
  'claude-opus-4-5': {
    modelId: 'claude-opus-4-5',
    maxContextTokens: 200_000,
    softLimitFraction: 0.70,
    hardLimitFraction: 0.90,
    reservedForTools: 8_000,
    reservedForOutput: 4_000,
    compactionMode: 'auto',
  },
  'claude-sonnet-4-5': {
    modelId: 'claude-sonnet-4-5',
    maxContextTokens: 200_000,
    softLimitFraction: 0.70,
    hardLimitFraction: 0.90,
    reservedForTools: 8_000,
    reservedForOutput: 4_000,
    compactionMode: 'auto',
  },
  'claude-haiku-4-5': {
    modelId: 'claude-haiku-4-5',
    maxContextTokens: 200_000,
    softLimitFraction: 0.72,
    hardLimitFraction: 0.90,
    reservedForTools: 6_000,
    reservedForOutput: 4_000,
    compactionMode: 'auto',
  },
  'gpt-4o': {
    modelId: 'gpt-4o',
    maxContextTokens: 128_000,
    softLimitFraction: 0.65,
    hardLimitFraction: 0.85,
    reservedForTools: 8_000,
    reservedForOutput: 4_000,
    compactionMode: 'auto',
  },
  'o3': {
    modelId: 'o3',
    maxContextTokens: 200_000,
    softLimitFraction: 0.65,
    hardLimitFraction: 0.85,
    reservedForTools: 8_000,
    reservedForOutput: 8_000,
    compactionMode: 'auto',
  },
  'llama3.1:8b': {
    modelId: 'llama3.1:8b',
    maxContextTokens: 8_192,
    softLimitFraction: 0.60,
    hardLimitFraction: 0.80,
    reservedForTools: 1_024,
    reservedForOutput: 1_024,
    compactionMode: 'aggressive',
    vramConfig: {
      availableGB: 8,
      modelSizeGB: 5.5,
      safeUtilizationRate: 0.85,
    },
  },
  'llama3.1:70b': {
    modelId: 'llama3.1:70b',
    maxContextTokens: 8_192,
    softLimitFraction: 0.60,
    hardLimitFraction: 0.80,
    reservedForTools: 1_024,
    reservedForOutput: 1_024,
    compactionMode: 'aggressive',
    vramConfig: {
      availableGB: 48,
      modelSizeGB: 40.0,
      safeUtilizationRate: 0.85,
    },
  },
  'phi3:mini': {
    modelId: 'phi3:mini',
    maxContextTokens: 4_096,
    softLimitFraction: 0.55,
    hardLimitFraction: 0.75,
    reservedForTools: 512,
    reservedForOutput: 512,
    compactionMode: 'aggressive',
    vramConfig: {
      availableGB: 4,
      modelSizeGB: 2.2,
      safeUtilizationRate: 0.85,
    },
  },
  'mistral:7b': {
    modelId: 'mistral:7b',
    maxContextTokens: 32_768,
    softLimitFraction: 0.62,
    hardLimitFraction: 0.82,
    reservedForTools: 2_048,
    reservedForOutput: 2_048,
    compactionMode: 'auto',
  },
  'default-small': {
    modelId: 'default-small',
    maxContextTokens: 4_096,
    softLimitFraction: 0.55,
    hardLimitFraction: 0.75,
    reservedForTools: 512,
    reservedForOutput: 512,
    compactionMode: 'aggressive',
  },
  'default-large': {
    modelId: 'default-large',
    maxContextTokens: 128_000,
    softLimitFraction: 0.65,
    hardLimitFraction: 0.85,
    reservedForTools: 8_000,
    reservedForOutput: 4_000,
    compactionMode: 'auto',
  },
} as const

/**
 * Registry of known model capabilities.
 * Used by ProviderRouter and AdaptiveCompactor to make
 * model-aware decisions.
 *
 * @example
 * ```ts
 * const registry = new ModelCapabilityRegistry()
 * const profile = registry.getBudgetProfile('claude-opus-4-5')
 * console.log(profile.maxContextTokens) // 200000
 * ```
 */
export class ModelCapabilityRegistry {
  private readonly capabilities = new Map<string, ModelCapability>()

  constructor() {
    this.registerDefaults()
  }

  /**
   * Register a new model capability entry.
   * Overwrites existing entry if the modelId already exists.
   */
  register(capability: ModelCapability): void {
    this.capabilities.set(capability.modelId, capability)
  }

  /**
   * Look up a model capability by modelId.
   */
  get(modelId: string): ModelCapability | undefined {
    return this.capabilities.get(modelId)
  }

  /**
   * Get the context budget profile for a model.
   * Falls back to a sensible dynamic profile for unknown models.
   */
  getBudgetProfile(modelId: string): ContextBudgetProfile {
    const cap = this.capabilities.get(modelId)
    const profileKey = cap?.preferredBudgetProfileKey ?? modelId

    // Try exact match first
    if (BUILT_IN_BUDGET_PROFILES[profileKey]) {
      return BUILT_IN_BUDGET_PROFILES[profileKey]!
    }

    // Try to infer from model name heuristics
    return this.buildDynamicProfile(modelId, cap)
  }

  /**
   * Get all registered model IDs.
   */
  listModelIds(): string[] {
    return [...this.capabilities.keys()]
  }

  /**
   * Get all registered capabilities for a provider.
   */
  getByProvider(provider: ProviderId): ModelCapability[] {
    return [...this.capabilities.values()].filter(
      (cap) => cap.provider === provider
    )
  }

  /**
   * Check if a model supports tool calling.
   */
  supportsTools(modelId: string): boolean {
    return this.capabilities.get(modelId)?.supportsToolCalling ?? true
  }

  /**
   * Check if a model supports parallel tool calls.
   */
  supportsParallelTools(modelId: string): boolean {
    return this.capabilities.get(modelId)?.supportsParallelTools ?? false
  }

  /**
   * Get estimated cost for a model call in USD.
   * Returns 0 for local models.
   */
  estimateCost(modelId: string, inputTokens: number, outputTokens: number): number {
    const cap = this.capabilities.get(modelId)
    if (!cap?.costPer1MInputTokens || !cap.costPer1MOutputTokens) return 0
    return (
      (inputTokens / 1_000_000) * cap.costPer1MInputTokens +
      (outputTokens / 1_000_000) * cap.costPer1MOutputTokens
    )
  }

  /**
   * Build a conservative dynamic profile for an unknown model.
   * Uses heuristics based on model name patterns.
   */
  private buildDynamicProfile(
    modelId: string,
    cap?: ModelCapability
  ): ContextBudgetProfile {
    const lowerModelId = modelId.toLowerCase()

    // Use provided context window or infer from name
    let maxContextTokens = cap?.contextWindow ?? 4_096

    if (lowerModelId.includes('128k') || lowerModelId.includes('gpt-4o')) {
      maxContextTokens = 128_000
    } else if (
      lowerModelId.includes('200k') ||
      lowerModelId.includes('claude')
    ) {
      maxContextTokens = 200_000
    } else if (lowerModelId.includes('32k')) {
      maxContextTokens = 32_768
    } else if (lowerModelId.includes('16k')) {
      maxContextTokens = 16_384
    } else if (lowerModelId.includes('8k') || lowerModelId.includes('8b')) {
      maxContextTokens = 8_192
    }

    // Local models get conservative profiles
    const isLocal = cap?.isLocalModel ?? lowerModelId.includes(':')
    const profileKey = isLocal ? 'default-small' : 'default-large'
    const baseProfile = BUILT_IN_BUDGET_PROFILES[profileKey]!

    return {
      ...baseProfile,
      modelId,
      maxContextTokens,
    }
  }

  /**
   * Register all known models with their capabilities.
   */
  private registerDefaults(): void {
    const models: ModelCapability[] = [
      // ── Anthropic ──────────────────────────────────────────────────
      {
        modelId: 'claude-opus-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        supportsJsonMode: false,
        isLocalModel: false,
        costPer1MInputTokens: 15.00,
        costPer1MOutputTokens: 75.00,
        preferredBudgetProfileKey: 'claude-opus-4-5',
      },
      {
        modelId: 'claude-sonnet-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        supportsJsonMode: false,
        isLocalModel: false,
        costPer1MInputTokens: 3.00,
        costPer1MOutputTokens: 15.00,
        preferredBudgetProfileKey: 'claude-sonnet-4-5',
      },
      {
        modelId: 'claude-haiku-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        supportsJsonMode: false,
        isLocalModel: false,
        costPer1MInputTokens: 0.25,
        costPer1MOutputTokens: 1.25,
        preferredBudgetProfileKey: 'claude-haiku-4-5',
      },
      // ── OpenAI ─────────────────────────────────────────────────────
      {
        modelId: 'gpt-4o',
        provider: 'openai',
        contextWindow: 128_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        supportsJsonMode: true,
        isLocalModel: false,
        costPer1MInputTokens: 2.50,
        costPer1MOutputTokens: 10.00,
        preferredBudgetProfileKey: 'gpt-4o',
      },
      {
        modelId: 'o3',
        provider: 'openai',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: true,
        supportsJsonMode: true,
        isLocalModel: false,
        costPer1MInputTokens: 10.00,
        costPer1MOutputTokens: 40.00,
        preferredBudgetProfileKey: 'o3',
        notes: 'Reasoning model — high latency, high quality',
      },
      // ── Ollama (local) ─────────────────────────────────────────────
      {
        modelId: 'llama3.1:8b',
        provider: 'ollama',
        contextWindow: 8_192,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        supportsJsonMode: true,
        isLocalModel: true,
        estimatedVRAMGB: 5.5,
        preferredBudgetProfileKey: 'llama3.1:8b',
      },
      {
        modelId: 'llama3.1:70b',
        provider: 'ollama',
        contextWindow: 8_192,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        supportsJsonMode: true,
        isLocalModel: true,
        estimatedVRAMGB: 40.0,
        preferredBudgetProfileKey: 'llama3.1:70b',
      },
      {
        modelId: 'phi3:mini',
        provider: 'ollama',
        contextWindow: 4_096,
        supportsStreaming: true,
        supportsToolCalling: false,
        supportsParallelTools: false,
        supportsVision: false,
        supportsJsonMode: false,
        isLocalModel: true,
        estimatedVRAMGB: 2.2,
        preferredBudgetProfileKey: 'phi3:mini',
        notes: 'Fast, low-VRAM — use for summarization, classification',
      },
      {
        modelId: 'mistral:7b',
        provider: 'ollama',
        contextWindow: 32_768,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        supportsJsonMode: true,
        isLocalModel: true,
        estimatedVRAMGB: 4.1,
        preferredBudgetProfileKey: 'mistral:7b',
      },
    ]

    for (const model of models) {
      this.capabilities.set(model.modelId, model)
    }
  }
}

/**
 * Singleton registry instance — shared across the application.
 */
export const modelRegistry = new ModelCapabilityRegistry()
Pass 2 — Part 1 Complete ✅
What was generated in Part 1:
File	Lines	Status
package.json	62	✅ Buildable
tsconfig.json	18	✅ Buildable
src/types/errors.types.ts	150	✅ Complete
src/types/permission.types.ts	110	✅ Complete
src/types/tool.types.ts	105	✅ Complete
src/types/provider.types.ts	160	✅ Complete
src/types/memory.types.ts	80	✅ Complete
src/types/agent.types.ts	195	✅ Complete
src/types/index.ts	18	✅ Complete
src/EventBus.ts	130	✅ Complete
src/HooksRegistry.ts	175	✅ Complete
src/PermissionGate.ts	180	✅ Complete
src/ModelCapabilityRegistry.ts	280	✅ Complete



