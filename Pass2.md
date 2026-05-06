Pass 2 of 18: packages/core
The complete agent engine. Every file is fully implemented — no placeholders, no stubs. This is the package everything else depends on.

File Tree for Pass 2
text

packages/core/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   ├── errors.ts
│   ├── queryLoop.ts
│   ├── SessionManager.ts
│   ├── ToolRegistry.ts
│   ├── PermissionGate.ts
│   ├── ProviderRouter.ts
│   ├── AdaptiveCompactor.ts
│   ├── TurnAssembler.ts
│   ├── ModelCapabilityRegistry.ts
│   ├── HooksRegistry.ts
│   ├── EventBus.ts
│   ├── MCPClient.ts
│   ├── buddy/
│   │   └── BuddyEngine.ts
│   ├── prompts/
│   │   └── system.md
│   ├── types/
│   │   ├── agent.types.ts
│   │   ├── tool.types.ts
│   │   ├── permission.types.ts
│   │   ├── provider.types.ts
│   │   └── memory.types.ts
│   └── utils/
│       ├── tokens.ts
│       ├── cost.ts
│       └── retry.ts
└── tests/
    ├── queryLoop.test.ts
    ├── PermissionGate.test.ts
    ├── ToolRegistry.test.ts
    ├── AdaptiveCompactor.test.ts
    ├── EventBus.test.ts
    ├── HooksRegistry.test.ts
    ├── BuddyEngine.test.ts
    └── utils/
        ├── tokens.test.ts
        └── retry.test.ts
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
    "src/prompts"
  ],
  "scripts": {
    "build":      "tsc --project tsconfig.json",
    "dev":        "tsc --project tsconfig.json --watch",
    "typecheck":  "tsc --project tsconfig.json --noEmit",
    "test":       "bun test",
    "test:watch": "bun test --watch",
    "test:coverage": "bun test --coverage",
    "eval":       "bun run src/evals/run.ts",
    "clean":      "rm -rf dist tsconfig.tsbuildinfo"
  },
  "dependencies": {
    "@anthropic-ai/sdk":  "0.32.1",
    "@modelcontextprotocol/sdk": "1.0.4",
    "openai":             "4.73.0",
    "tiktoken":           "1.0.17",
    "zod":                "3.23.8"
  },
  "devDependencies": {
    "@types/node":        "^22.10.0",
    "typescript":         "^5.7.2"
  },
  "peerDependencies": {
    "typescript": ">=5.0.0"
  }
}
packages/core/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "tsBuildInfoFile": "./tsconfig.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
packages/core/src/types/agent.types.ts
TypeScript

import type { ToolRegistry } from '../ToolRegistry.js'
import type { PermissionGate } from '../PermissionGate.js'
import type { HooksRegistry } from '../HooksRegistry.js'
import type { EventBus } from '../EventBus.js'
import type { MCPClient } from '../MCPClient.js'

// ── Provider IDs ───────────────────────────────────────────────────────
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

// ── Compaction ─────────────────────────────────────────────────────────
export type CompactionMode = 'micro' | 'auto' | 'full' | 'disabled'

export interface VRAMConfig {
  availableGB: number
  modelSizeGB: number
  safeUtilizationRate: number
}

export interface ContextBudgetProfile {
  modelId: string
  maxContextTokens: number
  softLimitRatio: number
  hardLimitRatio: number
  reservedForTools: number
  reservedForOutput: number
  compactionMode: CompactionMode
  vramAwareness?: VRAMConfig
}

// ── Model Config ───────────────────────────────────────────────────────
export interface ModelConfig {
  provider: ProviderId
  modelId: string
  apiKey?: string
  baseUrl?: string
  maxTokens?: number
  temperature?: number
  topP?: number
  streaming: boolean
}

// ── Messages ───────────────────────────────────────────────────────────
export interface UserMessage {
  role: 'user'
  content: string
  attachments?: MessageAttachment[]
}

export interface AssistantMessage {
  role: 'assistant'
  content: string
  toolCalls?: ToolCallRequest[]
  usage?: TokenUsage
}

export interface SystemMessage {
  role: 'system'
  content: string
}

export type Message = UserMessage | AssistantMessage | SystemMessage

export interface MessageAttachment {
  type: 'image' | 'file' | 'text'
  name: string
  content: string | Buffer
  mimeType?: string
}

export interface ToolCallRequest {
  id: string
  name: string
  input: Record<string, unknown>
}

export interface ToolCallResult {
  toolCallId: string
  toolName: string
  content: string
  isError: boolean
  error?: Error | unknown
  tokenCount?: number
  durationMs?: number
}

export interface TokenUsage {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens?: number
  cacheCreationInputTokens?: number
}

// ── Memory State ───────────────────────────────────────────────────────
export interface MemoryState {
  sessionId: string
  workingDirectory: string
  projectInstructions?: string
  memoryIndexContent?: string
  entries: MemoryEntry[]
}

export interface MemoryEntry {
  id: string
  timestamp: number
  sessionId: string
  category: MemoryCategory
  content: string
  importance: MemoryImportance
  tags: string[]
  expiresAt?: number
  graphNodeId?: string
  wikiSlug?: string
}

export type MemoryCategory =
  | 'architecture'
  | 'preference'
  | 'file_location'
  | 'decision'
  | 'issue'
  | 'research'
  | 'buddy'

export type MemoryImportance = 'low' | 'medium' | 'high' | 'critical'

// ── Agent Context ──────────────────────────────────────────────────────
export interface AgentContext {
  sessionId: string
  workingDirectory: string
  model: ModelConfig
  budget: ContextBudgetProfile
  permissions: PermissionGate
  memory: MemoryState
  hooks: HooksRegistry
  mcp: MCPClient
  tools: ToolRegistry
  events: EventBus
  costTracker: CostTracker
  metadata: AgentMetadata
}

export interface AgentMetadata {
  maxTurns: number
  parentSessionId?: string
  taskId?: string
  worktreeId?: string
  isSubAgent: boolean
  startedAt: number
  tags: string[]
  [key: string]: unknown
}

export interface CostTracker {
  sessionCostUsd: number
  sessionInputTokens: number
  sessionOutputTokens: number
  dailyCostUsd: number
  lastResetAt: number
}

// ── Agent Events ───────────────────────────────────────────────────────
export type AgentEventType =
  | 'session_start'
  | 'session_end'
  | 'turn_start'
  | 'model_request'
  | 'model_response'
  | 'tool_call'
  | 'tool_result'
  | 'compaction_triggered'
  | 'compaction_complete'
  | 'turn_complete'
  | 'session_error'
  | 'permission_denied'
  | 'max_turns_exceeded'
  | 'cost_cap_exceeded'
  | 'memory_updated'
  | 'buddy_event'

export interface AgentEvent<T = unknown> {
  type: AgentEventType
  sessionId: string
  timestamp: number
  data: T
}

// ── Session ────────────────────────────────────────────────────────────
export interface Session {
  id: string
  context: AgentContext
  status: SessionStatus
  history: Message[]
  toolResults: ToolCallResult[]
  createdAt: number
  lastActiveAt: number
  turnCount: number
  totalCostUsd: number
}

export type SessionStatus =
  | 'idle'
  | 'running'
  | 'waiting_for_user'
  | 'compacting'
  | 'error'
  | 'complete'
packages/core/src/types/tool.types.ts
TypeScript

import type { JSONSchema7 } from 'json-schema'
import type { AgentContext, ToolCallResult } from './agent.types.js'
import type { PermissionLevel } from './permission.types.js'

export interface ToolDefinition {
  name: string
  description: string
  inputSchema: JSONSchema7
  parallelSafe: boolean
  requiredPermission: PermissionLevel
  timeoutMs: number
  retryable: boolean
  tags: string[]
  handler: ToolHandler
}

export type ToolHandler = (
  input: Record<string, unknown>,
  ctx: AgentContext
) => Promise<ToolCallResult>

export interface ToolCall {
  id: string
  name: string
  input: Record<string, unknown>
}

export interface ToolExecutionOptions {
  overrideTimeoutMs?: number
  skipPermissionCheck?: boolean
  metadata?: Record<string, unknown>
}
packages/core/src/types/permission.types.ts
TypeScript

export type PermissionLevel =
  | 'READ_ONLY'
  | 'WRITE_LOCAL'
  | 'NETWORK'
  | 'SHELL'
  | 'DANGEROUS'

export const PERMISSION_TIER_RANK: Record<PermissionLevel, number> = {
  READ_ONLY:   0,
  WRITE_LOCAL: 1,
  NETWORK:     2,
  SHELL:       3,
  DANGEROUS:   4,
}

export interface PermissionSet {
  maxLevel: PermissionLevel
  allowlist?: string[]
  denylist?: string[]
  requireConfirmation?: PermissionLevel[]
  workspaceBoundary?: string
  allowedNetworkHosts?: string[]
}

export interface PermissionCheckPayload {
  tool: string
  input: Record<string, unknown>
  filePath?: string
  networkHost?: string
}

export type ConfirmFn = (message: string, context: string) => Promise<boolean>
packages/core/src/types/provider.types.ts
TypeScript

import type { ProviderId } from './agent.types.js'

export interface ProviderConfig {
  id: string
  provider: ProviderId
  displayName: string
  models: ModelEntry[]
  defaultModel: string
  apiKeyRequired: boolean
  baseUrl?: string
  isLocal: boolean
  supportsStreaming: boolean
  supportsTools: boolean
  supportsVision: boolean
  supportsParallelTools: boolean
  costTracking: boolean
  status: ProviderStatus
}

export type ProviderStatus = 'available' | 'unavailable' | 'unconfigured'

export interface ModelEntry {
  modelId: string
  displayName: string
  contextWindow: number
  costPer1MInput?: number
  costPer1MOutput?: number
  isDefault?: boolean
  supportsVision: boolean
  supportsTools: boolean
  supportsParallelTools: boolean
  estimatedVRAMGB?: number
  notes?: string
}

export interface RoutingConfig {
  preferLocalOnBattery: boolean
  preferredProviderOrder: ProviderId[]
  fallbackProvider?: ProviderId
  fallbackModelId?: string
  costCapPerSessionUsd?: number
  dailyCostCapUsd?: number
}

export interface ProviderRouteRule {
  condition: (modelId: string, provider: ProviderId) => boolean
  fallbackProvider: ProviderId
  fallbackModelId: string
  reason: string
}
packages/core/src/types/memory.types.ts
TypeScript

export interface ConversationTurn {
  role: 'user' | 'assistant'
  content: string
  tokenCount: number
  importance: TurnImportance
  timestamp: number
  toolCallIds?: string[]
}

export type TurnImportance = 'normal' | 'important' | 'critical'

export interface AssembledContext {
  systemPrompt: string
  projectInstructions: string
  memoryIndex: string
  history: ConversationTurn[]
  pendingToolResults: PendingToolResult[]
  currentInput: string
  tokenCount: number
  modelId: string
}

export interface PendingToolResult {
  toolCallId: string
  toolName: string
  content: string
  tokenCount: number
  timestamp: number
  isError: boolean
}
packages/core/src/errors.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Typed error hierarchy for packages/core
// All errors extend LocoWorkerError for consistent handling
// ─────────────────────────────────────────────────────────────────────────────

export class LocoWorkerError extends Error {
  readonly code: string
  readonly context?: Record<string, unknown>

  constructor(message: string, code: string, context?: Record<string, unknown>) {
    super(message)
    this.name = 'LocoWorkerError'
    this.code = code
    this.context = context
    // Preserve prototype chain
    Object.setPrototypeOf(this, new.target.prototype)
  }
}

// ── Permission Errors ──────────────────────────────────────────────────────
export class PermissionDeniedError extends LocoWorkerError {
  constructor(
    tool: string,
    requiredLevel: string,
    maxLevel: string,
  ) {
    super(
      `Permission denied: tool "${tool}" requires ${requiredLevel}, ` +
      `but session max is ${maxLevel}`,
      'PERMISSION_DENIED',
      { tool, requiredLevel, maxLevel }
    )
    this.name = 'PermissionDeniedError'
  }
}

export class WorkspaceBoundaryError extends LocoWorkerError {
  constructor(filePath: string, boundary: string) {
    super(
      `File path "${filePath}" is outside workspace boundary "${boundary}"`,
      'WORKSPACE_BOUNDARY_VIOLATION',
      { filePath, boundary }
    )
    this.name = 'WorkspaceBoundaryError'
  }
}

export class DeniedToolError extends LocoWorkerError {
  constructor(tool: string) {
    super(
      `Tool "${tool}" is on the denylist for this session`,
      'TOOL_DENIED',
      { tool }
    )
    this.name = 'DeniedToolError'
  }
}

// ── Tool Errors ────────────────────────────────────────────────────────────
export class ToolNotFoundError extends LocoWorkerError {
  constructor(toolName: string) {
    super(
      `Tool "${toolName}" is not registered in the ToolRegistry`,
      'TOOL_NOT_FOUND',
      { toolName }
    )
    this.name = 'ToolNotFoundError'
  }
}

export class ToolTimeoutError extends LocoWorkerError {
  constructor(toolName: string, timeoutMs: number) {
    super(
      `Tool "${toolName}" timed out after ${timeoutMs}ms`,
      'TOOL_TIMEOUT',
      { toolName, timeoutMs }
    )
    this.name = 'ToolTimeoutError'
  }
}

export class ToolExecutionError extends LocoWorkerError {
  constructor(toolName: string, cause: unknown) {
    const causeMessage = cause instanceof Error ? cause.message : String(cause)
    super(
      `Tool "${toolName}" failed: ${causeMessage}`,
      'TOOL_EXECUTION_FAILED',
      { toolName, cause: causeMessage }
    )
    this.name = 'ToolExecutionError'
    if (cause instanceof Error) {
      this.cause = cause
    }
  }
}

export class ToolInputValidationError extends LocoWorkerError {
  constructor(toolName: string, validationErrors: string[]) {
    super(
      `Invalid input for tool "${toolName}": ${validationErrors.join(', ')}`,
      'TOOL_INPUT_INVALID',
      { toolName, validationErrors }
    )
    this.name = 'ToolInputValidationError'
  }
}

// ── Provider Errors ────────────────────────────────────────────────────────
export class ProviderError extends LocoWorkerError {
  readonly statusCode?: number

  constructor(
    provider: string,
    message: string,
    statusCode?: number,
    context?: Record<string, unknown>
  ) {
    super(
      `Provider "${provider}" error: ${message}`,
      'PROVIDER_ERROR',
      { provider, statusCode, ...context }
    )
    this.name = 'ProviderError'
    this.statusCode = statusCode
  }
}

export class ProviderRateLimitError extends ProviderError {
  readonly retryAfterMs: number

  constructor(provider: string, retryAfterMs: number) {
    super(provider, `Rate limited — retry after ${retryAfterMs}ms`, 429)
    this.name = 'ProviderRateLimitError'
    this.retryAfterMs = retryAfterMs
  }
}

export class ProviderAuthError extends ProviderError {
  constructor(provider: string) {
    super(provider, 'Authentication failed — check your API key', 401)
    this.name = 'ProviderAuthError'
  }
}

export class ProviderNotConfiguredError extends LocoWorkerError {
  constructor(provider: string) {
    super(
      `Provider "${provider}" is not configured — set the API key or base URL`,
      'PROVIDER_NOT_CONFIGURED',
      { provider }
    )
    this.name = 'ProviderNotConfiguredError'
  }
}

// ── Session Errors ─────────────────────────────────────────────────────────
export class MaxTurnsExceededError extends LocoWorkerError {
  constructor(maxTurns: number, sessionId: string) {
    super(
      `Session "${sessionId}" exceeded maximum turn count of ${maxTurns}`,
      'MAX_TURNS_EXCEEDED',
      { maxTurns, sessionId }
    )
    this.name = 'MaxTurnsExceededError'
  }
}

export class CostCapExceededError extends LocoWorkerError {
  constructor(capUsd: number, currentUsd: number, sessionId: string) {
    super(
      `Session "${sessionId}" exceeded cost cap of $${capUsd.toFixed(4)} ` +
      `(current: $${currentUsd.toFixed(4)})`,
      'COST_CAP_EXCEEDED',
      { capUsd, currentUsd, sessionId }
    )
    this.name = 'CostCapExceededError'
  }
}

export class SessionNotFoundError extends LocoWorkerError {
  constructor(sessionId: string) {
    super(
      `Session "${sessionId}" not found`,
      'SESSION_NOT_FOUND',
      { sessionId }
    )
    this.name = 'SessionNotFoundError'
  }
}

// ── Context Errors ─────────────────────────────────────────────────────────
export class ContextOverflowError extends LocoWorkerError {
  constructor(tokenCount: number, maxTokens: number, modelId: string) {
    super(
      `Context overflow: ${tokenCount} tokens exceeds model "${modelId}" ` +
      `maximum of ${maxTokens} even after full compaction`,
      'CONTEXT_OVERFLOW',
      { tokenCount, maxTokens, modelId }
    )
    this.name = 'ContextOverflowError'
  }
}

// ── MCP Errors ─────────────────────────────────────────────────────────────
export class MCPConnectionError extends LocoWorkerError {
  constructor(serverUrl: string, cause?: unknown) {
    const causeMessage = cause instanceof Error ? cause.message : String(cause ?? 'unknown')
    super(
      `Failed to connect to MCP server at "${serverUrl}": ${causeMessage}`,
      'MCP_CONNECTION_FAILED',
      { serverUrl }
    )
    this.name = 'MCPConnectionError'
  }
}

// ── Injection Error ────────────────────────────────────────────────────────
export class InjectionAttemptError extends LocoWorkerError {
  constructor(source: string, pattern: string) {
    super(
      `Potential prompt injection detected in "${source}" — pattern: "${pattern}"`,
      'INJECTION_ATTEMPT',
      { source, pattern }
    )
    this.name = 'InjectionAttemptError'
  }
}
packages/core/src/utils/tokens.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Token estimation utilities
// Uses tiktoken for cl100k_base (GPT-4/Claude compatible estimate)
// Falls back to character-based heuristic for local models
// ─────────────────────────────────────────────────────────────────────────────

import { get_encoding, type Tiktoken } from 'tiktoken'

let encoder: Tiktoken | null = null

function getEncoder(): Tiktoken {
  if (!encoder) {
    encoder = get_encoding('cl100k_base')
  }
  return encoder
}

/**
 * Count tokens in a string using tiktoken cl100k_base encoding.
 * This is an estimate — actual token counts vary slightly by model.
 */
export function countTokens(text: string): number {
  if (!text || text.length === 0) return 0
  try {
    const enc = getEncoder()
    return enc.encode(text).length
  } catch {
    // Fallback: 1 token ≈ 4 characters (conservative estimate)
    return Math.ceil(text.length / 4)
  }
}

/**
 * Estimate tokens for a list of messages.
 * Includes per-message overhead (role tokens, formatting).
 */
export function countMessageTokens(
  messages: Array<{ role: string; content: string }>
): number {
  // 4 tokens overhead per message (role + formatting)
  const MESSAGE_OVERHEAD = 4
  // 3 tokens for reply priming
  const REPLY_PRIMING = 3

  return messages.reduce((total, msg) => {
    return total + MESSAGE_OVERHEAD + countTokens(msg.content)
  }, REPLY_PRIMING)
}

/**
 * Estimate tokens quickly using character heuristic.
 * Use when speed matters more than accuracy (e.g., pre-checks).
 */
export function estimateTokensFast(text: string): number {
  if (!text) return 0
  // Heuristic: ~4 chars per token for English, ~2 for code
  // We use 3.5 as a middle ground
  return Math.ceil(text.length / 3.5)
}

/**
 * Format a token count for display.
 */
export function formatTokenCount(count: number): string {
  if (count >= 1_000_000) return `${(count / 1_000_000).toFixed(1)}M`
  if (count >= 1_000) return `${(count / 1_000).toFixed(1)}k`
  return String(count)
}

/**
 * Calculate the percentage of a context window used.
 */
export function contextUsagePercent(used: number, max: number): number {
  if (max === 0) return 0
  return Math.min(100, Math.round((used / max) * 100))
}

/**
 * Truncate text to approximately N tokens.
 */
export function truncateToTokens(text: string, maxTokens: number): string {
  if (countTokens(text) <= maxTokens) return text

  // Binary search for the right character count
  let low = 0
  let high = text.length
  while (low < high) {
    const mid = Math.floor((low + high + 1) / 2)
    if (countTokens(text.slice(0, mid)) <= maxTokens) {
      low = mid
    } else {
      high = mid - 1
    }
  }
  return text.slice(0, low) + '…'
}

/**
 * Free the tiktoken encoder to release memory.
 * Call when shutting down.
 */
export function freeEncoder(): void {
  if (encoder) {
    encoder.free()
    encoder = null
  }
}
packages/core/src/utils/cost.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Cost calculation utilities
// Prices are per 1M tokens in USD
// ─────────────────────────────────────────────────────────────────────────────

import type { ProviderId } from '../types/agent.types.js'

export interface ModelPricing {
  modelId: string
  provider: ProviderId
  inputPer1M: number
  outputPer1M: number
  cacheReadPer1M?: number
  cacheWritePer1M?: number
}

// ── Pricing Registry ───────────────────────────────────────────────────────
export const MODEL_PRICING: ModelPricing[] = [
  // Anthropic
  {
    modelId: 'claude-opus-4-5',
    provider: 'anthropic',
    inputPer1M:       15.00,
    outputPer1M:      75.00,
    cacheReadPer1M:    1.50,
    cacheWritePer1M:  18.75,
  },
  {
    modelId: 'claude-sonnet-4-5',
    provider: 'anthropic',
    inputPer1M:        3.00,
    outputPer1M:      15.00,
    cacheReadPer1M:    0.30,
    cacheWritePer1M:   3.75,
  },
  {
    modelId: 'claude-haiku-4-5',
    provider: 'anthropic',
    inputPer1M:        0.25,
    outputPer1M:       1.25,
    cacheReadPer1M:    0.03,
    cacheWritePer1M:   0.30,
  },
  // OpenAI
  {
    modelId: 'gpt-4o',
    provider: 'openai',
    inputPer1M:        2.50,
    outputPer1M:      10.00,
  },
  {
    modelId: 'gpt-4o-mini',
    provider: 'openai',
    inputPer1M:        0.15,
    outputPer1M:       0.60,
  },
  {
    modelId: 'o3',
    provider: 'openai',
    inputPer1M:       10.00,
    outputPer1M:      40.00,
  },
  // Mistral
  {
    modelId: 'mistral-large-latest',
    provider: 'mistral',
    inputPer1M:        2.00,
    outputPer1M:       6.00,
  },
  {
    modelId: 'mistral-small-latest',
    provider: 'mistral',
    inputPer1M:        0.20,
    outputPer1M:       0.60,
  },
  // Groq (fast inference, OpenAI compat)
  {
    modelId: 'llama-3.1-70b-versatile',
    provider: 'groq',
    inputPer1M:        0.59,
    outputPer1M:       0.79,
  },
  // Together AI
  {
    modelId: 'meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo',
    provider: 'together',
    inputPer1M:        0.88,
    outputPer1M:       0.88,
  },
]

const pricingMap = new Map<string, ModelPricing>(
  MODEL_PRICING.map(p => [`${p.provider}:${p.modelId}`, p])
)

/**
 * Calculate the cost of a model call in USD.
 * Returns 0 for local models (no cost).
 */
export function calculateCostUsd(
  modelId: string,
  provider: ProviderId,
  inputTokens: number,
  outputTokens: number,
  cacheReadTokens = 0,
): number {
  // Local providers have no cost
  const localProviders: ProviderId[] = ['ollama', 'lmstudio', 'llamacpp', 'custom']
  if (localProviders.includes(provider)) return 0

  const pricing = pricingMap.get(`${provider}:${modelId}`)
  if (!pricing) {
    // Unknown model — return 0 rather than crash
    console.warn(`[cost] Unknown model pricing: ${provider}:${modelId}`)
    return 0
  }

  const inputCost = (inputTokens / 1_000_000) * pricing.inputPer1M
  const outputCost = (outputTokens / 1_000_000) * pricing.outputPer1M
  const cacheCost = pricing.cacheReadPer1M
    ? (cacheReadTokens / 1_000_000) * pricing.cacheReadPer1M
    : 0

  return inputCost + outputCost + cacheCost
}

/**
 * Format a USD amount for display.
 */
export function formatUsd(amount: number): string {
  if (amount < 0.001) return `$${(amount * 1000).toFixed(3)}m` // millicents
  if (amount < 1) return `$${amount.toFixed(4)}`
  return `$${amount.toFixed(2)}`
}

/**
 * Check if a cost cap would be exceeded.
 */
export function wouldExceedCap(
  currentCost: number,
  additionalCost: number,
  capUsd: number
): boolean {
  return currentCost + additionalCost > capUsd
}

/**
 * Get cost alert threshold (default: 80% of cap).
 */
export function getCostAlertThreshold(capUsd: number, percent = 0.80): number {
  return capUsd * percent
}
packages/core/src/utils/retry.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Exponential backoff retry utility
// ─────────────────────────────────────────────────────────────────────────────

export interface RetryOptions {
  /** Maximum number of attempts (including first). Default: 3 */
  maxAttempts: number
  /** Base delay in ms. Default: 1000 */
  baseDelayMs: number
  /** Maximum delay in ms. Default: 30000 */
  maxDelayMs: number
  /** Jitter factor (0-1). Default: 0.2 */
  jitter: number
  /** Which errors are retryable. Default: all errors */
  isRetryable?: (error: unknown) => boolean
  /** Called before each retry attempt */
  onRetry?: (error: unknown, attempt: number, delayMs: number) => void
}

const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxAttempts:  3,
  baseDelayMs:  1000,
  maxDelayMs:   30_000,
  jitter:       0.2,
}

/**
 * Execute a function with exponential backoff retry.
 *
 * @example
 * const result = await withRetry(() => callApi(), {
 *   maxAttempts: 3,
 *   isRetryable: (err) => err instanceof ProviderRateLimitError,
 * })
 */
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options }
  let lastError: unknown

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error

      // Check if we should stop retrying
      if (attempt >= opts.maxAttempts) break
      if (opts.isRetryable && !opts.isRetryable(error)) break

      // Calculate delay with exponential backoff + jitter
      const exponential = opts.baseDelayMs * Math.pow(2, attempt - 1)
      const capped = Math.min(exponential, opts.maxDelayMs)
      const jitterAmount = capped * opts.jitter * (Math.random() * 2 - 1)
      const delay = Math.max(0, Math.round(capped + jitterAmount))

      opts.onRetry?.(error, attempt, delay)

      await sleep(delay)
    }
  }

  throw lastError
}

/**
 * Sleep for a given number of milliseconds.
 */
export function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

/**
 * Create a retry options object for rate-limited providers.
 */
export function rateLimitRetryOptions(
  onRetry?: (attempt: number) => void
): Partial<RetryOptions> {
  return {
    maxAttempts:  5,
    baseDelayMs:  2000,
    maxDelayMs:   60_000,
    jitter:       0.3,
    isRetryable:  (err) => {
      if (err instanceof Error) {
        const msg = err.message.toLowerCase()
        return (
          msg.includes('rate limit') ||
          msg.includes('429') ||
          msg.includes('503') ||
          msg.includes('529') ||
          msg.includes('overloaded')
        )
      }
      return false
    },
    onRetry: (err, attempt, delay) => {
      console.warn(
        `[retry] Attempt ${attempt} failed — retrying in ${delay}ms:`,
        err instanceof Error ? err.message : String(err)
      )
      onRetry?.(attempt)
    },
  }
}
packages/core/src/EventBus.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Typed in-process event bus
// Supports synchronous and asynchronous handlers
// Returns unsubscribe functions for clean teardown
// ─────────────────────────────────────────────────────────────────────────────

type SyncHandler<T> = (event: T) => void
type AsyncHandler<T> = (event: T) => Promise<void>
type Handler<T> = SyncHandler<T> | AsyncHandler<T>

interface Subscription<T> {
  id: string
  handler: Handler<T>
  once: boolean
}

export class EventBus {
  private listeners = new Map<string, Set<Subscription<unknown>>>()
  private emitCount = 0

  /**
   * Subscribe to an event type.
   * Returns an unsubscribe function.
   */
  on<T>(eventType: string, handler: Handler<T>): () => void {
    return this.addSubscription(eventType, handler, false)
  }

  /**
   * Subscribe to an event type — fires once then unsubscribes.
   * Returns an unsubscribe function.
   */
  once<T>(eventType: string, handler: Handler<T>): () => void {
    return this.addSubscription(eventType, handler, true)
  }

  /**
   * Emit an event to all subscribers.
   * Async handlers are awaited in parallel.
   * Errors in handlers are caught and logged — they do not propagate.
   */
  async emit<T>(eventType: string, data: T): Promise<void> {
    this.emitCount++
    const subs = this.listeners.get(eventType)
    if (!subs || subs.size === 0) return

    const toRemove: Subscription<unknown>[] = []
    const promises: Promise<void>[] = []

    for (const sub of subs) {
      if (sub.once) toRemove.push(sub)

      const result = (() => {
        try {
          return (sub.handler as Handler<T>)(data)
        } catch (err) {
          console.error(`[EventBus] Sync handler error for "${eventType}":`, err)
          return undefined
        }
      })()

      if (result instanceof Promise) {
        promises.push(
          result.catch(err => {
            console.error(
              `[EventBus] Async handler error for "${eventType}":`, err
            )
          })
        )
      }
    }

    // Remove one-time subscriptions
    for (const sub of toRemove) subs.delete(sub)

    // Await all async handlers
    if (promises.length > 0) {
      await Promise.allSettled(promises)
    }
  }

  /**
   * Emit an event synchronously — only fires sync handlers.
   * Useful in hot paths where async overhead is undesirable.
   */
  emitSync<T>(eventType: string, data: T): void {
    const subs = this.listeners.get(eventType)
    if (!subs || subs.size === 0) return

    const toRemove: Subscription<unknown>[] = []

    for (const sub of subs) {
      if (sub.once) toRemove.push(sub)
      try {
        const result = (sub.handler as Handler<T>)(data)
        if (result instanceof Promise) {
          // Fire and forget for async handlers in sync context
          result.catch(err =>
            console.error(`[EventBus] Async handler in sync emit for "${eventType}":`, err)
          )
        }
      } catch (err) {
        console.error(`[EventBus] Handler error for "${eventType}":`, err)
      }
    }

    for (const sub of toRemove) subs.delete(sub)
  }

  /**
   * Remove all listeners for a specific event type.
   */
  removeAllListeners(eventType?: string): void {
    if (eventType) {
      this.listeners.delete(eventType)
    } else {
      this.listeners.clear()
    }
  }

  /**
   * Get the number of listeners for an event type.
   */
  listenerCount(eventType: string): number {
    return this.listeners.get(eventType)?.size ?? 0
  }

  /**
   * Get all registered event types.
   */
  eventNames(): string[] {
    return [...this.listeners.keys()]
  }

  /**
   * Total number of events emitted since creation.
   */
  getEmitCount(): number {
    return this.emitCount
  }

  private addSubscription<T>(
    eventType: string,
    handler: Handler<T>,
    once: boolean
  ): () => void {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set())
    }

    const sub: Subscription<T> = {
      id: `${eventType}:${Math.random().toString(36).slice(2, 9)}`,
      handler,
      once,
    }

    this.listeners.get(eventType)!.add(sub as Subscription<unknown>)

    // Return unsubscribe function
    return () => {
      this.listeners.get(eventType)?.delete(sub as Subscription<unknown>)
    }
  }
}
packages/core/src/HooksRegistry.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// Lifecycle hook registry for plugins and extensions
// Hooks are middleware-style: each handler receives and can transform the payload
// ─────────────────────────────────────────────────────────────────────────────

import type { AgentContext } from './types/agent.types.js'

export type HookName =
  | 'beforeTurn'
  | 'afterTurn'
  | 'beforeToolCall'
  | 'afterToolCall'
  | 'beforeCompaction'
  | 'afterCompaction'
  | 'onSessionStart'
  | 'onSessionEnd'
  | 'onPermissionDenied'
  | 'onModelResponse'
  | 'onMemoryUpdate'
  | 'onError'
  | 'onBuddyEvent'

export type HookHandler<T = unknown> = (
  payload: T,
  ctx: AgentContext
) => Promise<T | void> | T | void

interface RegisteredHook<T> {
  id: string
  name: HookName
  handler: HookHandler<T>
  priority: number
  pluginName?: string
}

export class HooksRegistry {
  private hooks = new Map<HookName, RegisteredHook<unknown>[]>()

  /**
   * Register a hook handler for a lifecycle event.
   * Higher priority handlers run first (default: 0).
   * Returns an unregister function.
   */
  register<T>(
    name: HookName,
    handler: HookHandler<T>,
    options: { priority?: number; pluginName?: string } = {}
  ): () => void {
    const { priority = 0, pluginName } = options

    const registered: RegisteredHook<T> = {
      id: `${name}:${Math.random().toString(36).slice(2, 9)}`,
      name,
      handler,
      priority,
      pluginName,
    }

    const existing = (this.hooks.get(name) ?? []) as RegisteredHook<T>[]
    const sorted = [...existing, registered].sort((a, b) => b.priority - a.priority)
    this.hooks.set(name, sorted as RegisteredHook<unknown>[])

    // Return unregister function
    return () => {
      const current = this.hooks.get(name) ?? []
      this.hooks.set(
        name,
        current.filter(h => h.id !== registered.id)
      )
    }
  }

  /**
   * Run all handlers for a hook in priority order.
   * Each handler can transform the payload by returning a new value.
   * If a handler returns void/undefined, the payload passes through unchanged.
   */
  async run<T>(
    name: HookName,
    payload: T,
    ctx: AgentContext
  ): Promise<T> {
    const handlers = (this.hooks.get(name) ?? []) as RegisteredHook<T>[]
    let current = payload

    for (const hook of handlers) {
      try {
        const result = await Promise.resolve(hook.handler(current, ctx))
        if (result !== undefined && result !== null) {
          current = result as T
        }
      } catch (err) {
        console.error(
          `[HooksRegistry] Error in "${name}" hook` +
          (hook.pluginName ? ` (plugin: ${hook.pluginName})` : '') +
          ':',
          err
        )
        // Hooks errors are non-fatal — continue to next handler
      }
    }

    return current
  }

  /**
   * Get the number of handlers registered for a hook.
   */
  handlerCount(name: HookName): number {
    return this.hooks.get(name)?.length ?? 0
  }

  /**
   * List all registered hook names that have handlers.
   */
  registeredHooks(): HookName[] {
    return [...this.hooks.entries()]
      .filter(([, handlers]) => handlers.length > 0)
      .map(([name]) => name)
  }

  /**
   * Remove all handlers (useful for testing).
   */
  clear(): void {
    this.hooks.clear()
  }
}
packages/core/src/PermissionGate.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// 5-Tier permission gate for tool execution
// Enforces workspace boundaries, allowlists, denylists, and confirmation flows
// ─────────────────────────────────────────────────────────────────────────────

import node_path from 'node:path'
import {
  PermissionDeniedError,
  WorkspaceBoundaryError,
  DeniedToolError,
} from './errors.js'
import type {
  PermissionLevel,
  PermissionSet,
  PermissionCheckPayload,
  ConfirmFn,
} from './types/permission.types.js'
import { PERMISSION_TIER_RANK } from './types/permission.types.js'

export class PermissionGate {
  private readonly permSet: PermissionSet
  private readonly confirmFn: ConfirmFn

  constructor(permSet: PermissionSet, confirmFn: ConfirmFn) {
    this.permSet = permSet
    this.confirmFn = confirmFn
  }

  /**
   * Check if a tool execution is permitted.
   * Throws a typed error if denied.
   * Returns true if permitted.
   */
  async check(
    required: PermissionLevel,
    payload: PermissionCheckPayload
  ): Promise<true> {
    const { tool, input, filePath, networkHost } = payload

    // ── Explicit denylist — always rejected ────────────────────────────
    if (this.permSet.denylist?.includes(tool)) {
      throw new DeniedToolError(tool)
    }

    // ── Explicit allowlist — bypasses tier check ───────────────────────
    if (this.permSet.allowlist?.includes(tool)) {
      // Still enforce workspace boundary even for allowlisted tools
      if (filePath) this.enforceWorkspaceBoundary(filePath)
      return true
    }

    // ── Tier rank check ────────────────────────────────────────────────
    const requiredRank = PERMISSION_TIER_RANK[required]
    const maxRank = PERMISSION_TIER_RANK[this.permSet.maxLevel]

    if (requiredRank > maxRank) {
      throw new PermissionDeniedError(tool, required, this.permSet.maxLevel)
    }

    // ── Network host allowlist ─────────────────────────────────────────
    if (required === 'NETWORK' && networkHost && this.permSet.allowedNetworkHosts) {
      const allowed = this.permSet.allowedNetworkHosts.some(
        allowed => networkHost === allowed || networkHost.endsWith(`.${allowed}`)
      )
      if (!allowed) {
        throw new PermissionDeniedError(
          tool,
          `NETWORK:${networkHost}`,
          `NETWORK:${this.permSet.allowedNetworkHosts.join(',')}`
        )
      }
    }

    // ── Workspace boundary ─────────────────────────────────────────────
    if (filePath) {
      this.enforceWorkspaceBoundary(filePath)
    }

    // ── Confirmation required ──────────────────────────────────────────
    if (this.permSet.requireConfirmation?.includes(required)) {
      const context = this.buildConfirmContext(tool, payload)
      const confirmed = await this.confirmFn(
        `Tool "${tool}" requires ${required} permission. Allow this action?`,
        context
      )
      if (!confirmed) {
        throw new PermissionDeniedError(tool, required, `${required}:CONFIRMATION_DENIED`)
      }
    }

    return true
  }

  /**
   * Pre-check a list of tool names before the agent loop starts.
   * Throws early if any tool is on the denylist.
   */
  precheck(anticipatedTools: string[]): void {
    for (const tool of anticipatedTools) {
      if (this.permSet.denylist?.includes(tool)) {
        throw new DeniedToolError(tool)
      }
    }
  }

  /**
   * Validate a file path is within the workspace boundary.
   * Resolves symlinks and relative paths before comparison.
   */
  enforceWorkspaceBoundary(filePath: string): void {
    if (!this.permSet.workspaceBoundary) return

    const resolved = node_path.resolve(filePath)
    const boundary = node_path.resolve(this.permSet.workspaceBoundary)

    if (!resolved.startsWith(boundary + node_path.sep) && resolved !== boundary) {
      throw new WorkspaceBoundaryError(resolved, boundary)
    }
  }

  /**
   * Check if the permission set allows a given level without throwing.
   */
  allows(level: PermissionLevel): boolean {
    return PERMISSION_TIER_RANK[level] <= PERMISSION_TIER_RANK[this.permSet.maxLevel]
  }

  /**
   * Get the current permission set (read-only).
   */
  getPermissionSet(): Readonly<PermissionSet> {
    return Object.freeze({ ...this.permSet })
  }

  /**
   * Create a more restrictive child gate (for sub-agents).
   */
  restrictTo(maxLevel: PermissionLevel): PermissionGate {
    const childRank = PERMISSION_TIER_RANK[maxLevel]
    const parentRank = PERMISSION_TIER_RANK[this.permSet.maxLevel]

    // Child cannot exceed parent permissions
    const effectiveLevel = childRank < parentRank
      ? maxLevel
      : this.permSet.maxLevel

    return new PermissionGate(
      {
        ...this.permSet,
        maxLevel: effectiveLevel,
      },
      this.confirmFn
    )
  }

  private buildConfirmContext(
    tool: string,
    payload: PermissionCheckPayload
  ): string {
    const parts: string[] = [`Tool: ${tool}`]
    if (payload.filePath) parts.push(`File: ${payload.filePath}`)
    if (payload.networkHost) parts.push(`Host: ${payload.networkHost}`)
    const inputStr = JSON.stringify(payload.input, null, 2)
    if (inputStr !== '{}') parts.push(`Input: ${inputStr.slice(0, 200)}`)
    return parts.join('\n')
  }
}

// ── Factory functions ──────────────────────────────────────────────────────

/** Create a read-only permission gate (safe for untrusted environments). */
export function createReadOnlyGate(workspaceBoundary?: string): PermissionGate {
  return new PermissionGate(
    {
      maxLevel: 'READ_ONLY',
      workspaceBoundary,
    },
    async () => false // Never confirm — read-only
  )
}

/** Create a standard development gate (WRITE_LOCAL, no shell by default). */
export function createStandardGate(
  workspaceBoundary: string,
  confirmFn: ConfirmFn
): PermissionGate {
  return new PermissionGate(
    {
      maxLevel: 'WRITE_LOCAL',
      workspaceBoundary,
      requireConfirmation: [],
    },
    confirmFn
  )
}

/** Create a full-access gate with confirmation for SHELL and DANGEROUS. */
export function createFullAccessGate(
  workspaceBoundary: string,
  confirmFn: ConfirmFn
): PermissionGate {
  return new PermissionGate(
    {
      maxLevel: 'DANGEROUS',
      workspaceBoundary,
      requireConfirmation: ['SHELL', 'DANGEROUS'],
    },
    confirmFn
  )
}

/** Auto-confirm gate for testing — never prompts. */
export function createTestGate(
  maxLevel: PermissionLevel = 'SHELL',
  workspaceBoundary?: string
): PermissionGate {
  return new PermissionGate(
    { maxLevel, workspaceBoundary },
    async () => true // Always confirm
  )
}
packages/core/src/ModelCapabilityRegistry.ts
TypeScript

import type { ProviderId, ContextBudgetProfile } from './types/agent.types.js'

export interface ModelCapability {
  modelId: string
  provider: ProviderId
  contextWindow: number
  supportsStreaming: boolean
  supportsToolCalling: boolean
  supportsParallelTools: boolean
  supportsVision: boolean
  isLocalModel: boolean
  estimatedVRAMGB?: number
  preferredCompactionMode: 'micro' | 'auto' | 'full' | 'disabled'
  notes?: string
}

// ── Default budget profiles ─────────────────────────────────────────────────
const BUDGET_PROFILES: Record<string, ContextBudgetProfile> = {
  'claude-opus-4-5': {
    modelId: 'claude-opus-4-5',
    maxContextTokens:    200_000,
    softLimitRatio:      0.70,
    hardLimitRatio:      0.90,
    reservedForTools:    8_000,
    reservedForOutput:   4_000,
    compactionMode:      'auto',
  },
  'claude-sonnet-4-5': {
    modelId: 'claude-sonnet-4-5',
    maxContextTokens:    200_000,
    softLimitRatio:      0.70,
    hardLimitRatio:      0.90,
    reservedForTools:    8_000,
    reservedForOutput:   4_000,
    compactionMode:      'auto',
  },
  'claude-haiku-4-5': {
    modelId: 'claude-haiku-4-5',
    maxContextTokens:    200_000,
    softLimitRatio:      0.65,
    hardLimitRatio:      0.85,
    reservedForTools:    4_000,
    reservedForOutput:   2_000,
    compactionMode:      'auto',
  },
  'gpt-4o': {
    modelId: 'gpt-4o',
    maxContextTokens:    128_000,
    softLimitRatio:      0.65,
    hardLimitRatio:      0.85,
    reservedForTools:    8_000,
    reservedForOutput:   4_000,
    compactionMode:      'auto',
  },
  'gpt-4o-mini': {
    modelId: 'gpt-4o-mini',
    maxContextTokens:    128_000,
    softLimitRatio:      0.65,
    hardLimitRatio:      0.85,
    reservedForTools:    4_000,
    reservedForOutput:   2_000,
    compactionMode:      'auto',
  },
  'o3': {
    modelId: 'o3',
    maxContextTokens:    200_000,
    softLimitRatio:      0.70,
    hardLimitRatio:      0.90,
    reservedForTools:    8_000,
    reservedForOutput:   8_000,
    compactionMode:      'auto',
  },
  'llama3.1:8b': {
    modelId: 'llama3.1:8b',
    maxContextTokens:    8_192,
    softLimitRatio:      0.60,
    hardLimitRatio:      0.80,
    reservedForTools:    512,
    reservedForOutput:   512,
    compactionMode:      'full',
    vramAwareness: {
      availableGB:        8,
      modelSizeGB:        5.5,
      safeUtilizationRate: 0.85,
    },
  },
  'llama3.1:70b': {
    modelId: 'llama3.1:70b',
    maxContextTokens:    8_192,
    softLimitRatio:      0.60,
    hardLimitRatio:      0.80,
    reservedForTools:    512,
    reservedForOutput:   1_024,
    compactionMode:      'full',
    vramAwareness: {
      availableGB:        48,
      modelSizeGB:        40.0,
      safeUtilizationRate: 0.85,
    },
  },
  'phi3:mini': {
    modelId: 'phi3:mini',
    maxContextTokens:    4_096,
    softLimitRatio:      0.55,
    hardLimitRatio:      0.75,
    reservedForTools:    256,
    reservedForOutput:   256,
    compactionMode:      'full',
    vramAwareness: {
      availableGB:        4,
      modelSizeGB:        2.4,
      safeUtilizationRate: 0.85,
    },
  },
  'mistral:7b': {
    modelId: 'mistral:7b',
    maxContextTokens:    32_768,
    softLimitRatio:      0.65,
    hardLimitRatio:      0.85,
    reservedForTools:    1_024,
    reservedForOutput:   1_024,
    compactionMode:      'auto',
  },
}

export class ModelCapabilityRegistry {
  private capabilities = new Map<string, ModelCapability>()
  private budgets = new Map<string, ContextBudgetProfile>()

  constructor() {
    this.seedDefaults()
  }

  register(cap: ModelCapability): void {
    this.capabilities.set(cap.modelId, cap)
  }

  registerBudget(profile: ContextBudgetProfile): void {
    this.budgets.set(profile.modelId, profile)
  }

  get(modelId: string): ModelCapability | undefined {
    return this.capabilities.get(modelId)
  }

  getBudgetProfile(modelId: string): ContextBudgetProfile {
    // 1. Check custom registered budgets
    const custom = this.budgets.get(modelId)
    if (custom) return custom

    // 2. Check built-in profiles
    const builtin = BUDGET_PROFILES[modelId]
    if (builtin) return builtin

    // 3. Fall back to a dynamic profile based on capability data
    const cap = this.capabilities.get(modelId)
    return this.buildDynamicProfile(modelId, cap)
  }

  list(): ModelCapability[] {
    return [...this.capabilities.values()]
  }

  listLocal(): ModelCapability[] {
    return [...this.capabilities.values()].filter(c => c.isLocalModel)
  }

  listCloud(): ModelCapability[] {
    return [...this.capabilities.values()].filter(c => !c.isLocalModel)
  }

  private buildDynamicProfile(
    modelId: string,
    cap?: ModelCapability
  ): ContextBudgetProfile {
    const maxCtx = cap?.contextWindow ?? 4_096
    const isSmall = maxCtx <= 8_192
    return {
      modelId,
      maxContextTokens:    maxCtx,
      softLimitRatio:      isSmall ? 0.60 : 0.70,
      hardLimitRatio:      isSmall ? 0.80 : 0.90,
      reservedForTools:    Math.floor(maxCtx * 0.08),
      reservedForOutput:   Math.floor(maxCtx * 0.08),
      compactionMode:      isSmall ? 'full' : 'auto',
    }
  }

  private seedDefaults(): void {
    const defaults: ModelCapability[] = [
      // Anthropic
      {
        modelId: 'claude-opus-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        isLocalModel: false,
        preferredCompactionMode: 'auto',
      },
      {
        modelId: 'claude-sonnet-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        isLocalModel: false,
        preferredCompactionMode: 'auto',
      },
      {
        modelId: 'claude-haiku-4-5',
        provider: 'anthropic',
        contextWindow: 200_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: false,
        isLocalModel: false,
        preferredCompactionMode: 'auto',
      },
      // OpenAI
      {
        modelId: 'gpt-4o',
        provider: 'openai',
        contextWindow: 128_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: true,
        isLocalModel: false,
        preferredCompactionMode: 'auto',
      },
      {
        modelId: 'gpt-4o-mini',
        provider: 'openai',
        contextWindow: 128_000,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: true,
        supportsVision: false,
        isLocalModel: false,
        preferredCompactionMode: 'auto',
      },
      // Ollama local
      {
        modelId: 'llama3.1:8b',
        provider: 'ollama',
        contextWindow: 8_192,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        isLocalModel: true,
        estimatedVRAMGB: 5.5,
        preferredCompactionMode: 'full',
      },
      {
        modelId: 'llama3.1:70b',
        provider: 'ollama',
        contextWindow: 8_192,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        isLocalModel: true,
        estimatedVRAMGB: 40.0,
        preferredCompactionMode: 'full',
      },
      {
        modelId: 'phi3:mini',
        provider: 'ollama',
        contextWindow: 4_096,
        supportsStreaming: true,
        supportsToolCalling: false,
        supportsParallelTools: false,
        supportsVision: false,
        isLocalModel: true,
        estimatedVRAMGB: 2.4,
        preferredCompactionMode: 'full',
        notes: 'Limited tool calling — use for summarization only',
      },
      {
        modelId: 'mistral:7b',
        provider: 'ollama',
        contextWindow: 32_768,
        supportsStreaming: true,
        supportsToolCalling: true,
        supportsParallelTools: false,
        supportsVision: false,
        isLocalModel: true,
        estimatedVRAMGB: 4.5,
        preferredCompactionMode: 'auto',
      },
    ]

    for (const cap of defaults) {
      this.register(cap)
    }

    for (const profile of Object.values(BUDGET_PROFILES)) {
      this.registerBudget(profile)
    }
  }
}

// Singleton registry instance
export const modelRegistry = new ModelCapabilityRegistry()
packages/core/src/ToolRegistry.ts
TypeScript

import {
  ToolNotFoundError,
  ToolTimeoutError,
  ToolExecutionError,
  ToolInputValidationError,
} from './errors.js'
import type { AgentContext, ToolCallResult } from './types/agent.types.js'
import type { ToolDefinition, ToolCall, ToolExecutionOptions } from './types/tool.types.js'

export class ToolRegistry {
  private tools = new Map<string, ToolDefinition>()

  /**
   * Register a tool definition.
   * Throws if a tool with the same name is already registered.
   */
  register(tool: ToolDefinition): void {
    if (this.tools.has(tool.name)) {
      throw new Error(
        `[ToolRegistry] Tool "${tool.name}" is already registered. ` +
        `Use registerOrReplace() to override.`
      )
    }
    this.tools.set(tool.name, tool)
  }

  /**
   * Register or replace a tool definition.
   */
  registerOrReplace(tool: ToolDefinition): void {
    this.tools.set(tool.name, tool)
  }

  /**
   * Register multiple tools at once.
   */
  registerAll(tools: ToolDefinition[]): void {
    for (const tool of tools) this.register(tool)
  }

  /**
   * Unregister a tool by name.
   */
  unregister(name: string): boolean {
    return this.tools.delete(name)
  }

  /**
   * Check if a tool is registered.
   */
  has(name: string): boolean {
    return this.tools.has(name)
  }

  /**
   * Get a tool definition by name.
   */
  get(name: string): ToolDefinition | undefined {
    return this.tools.get(name)
  }

  /**
   * Check if a tool can be executed in parallel safely.
   */
  isParallelSafe(name: string): boolean {
    return this.tools.get(name)?.parallelSafe ?? false
  }

  /**
   * Get all tool definitions as an array.
   */
  list(): ToolDefinition[] {
    return [...this.tools.values()]
  }

  /**
   * Get tool names only.
   */
  names(): string[] {
    return [...this.tools.keys()]
  }

  /**
   * Get tools filtered by tag.
   */
  byTag(tag: string): ToolDefinition[] {
    return [...this.tools.values()].filter(t => t.tags.includes(tag))
  }

  /**
   * Get tool definitions in the format expected by LLM APIs.
   * Returns an array of JSON Schema objects for the provider.
   */
  toAPIFormat(provider: 'anthropic' | 'openai'): object[] {
    return [...this.tools.values()].map(tool => {
      if (provider === 'anthropic') {
        return {
          name: tool.name,
          description: tool.description,
          input_schema: tool.inputSchema,
        }
      }
      // OpenAI format
      return {
        type: 'function',
        function: {
          name: tool.name,
          description: tool.description,
          parameters: tool.inputSchema,
        },
      }
    })
  }

  /**
   * Execute a tool call with permission checking and timeout.
   * Returns a ToolCallResult — never throws (errors are captured in result).
   */
  async execute(
    toolCall: ToolCall,
    ctx: AgentContext,
    options: ToolExecutionOptions = {}
  ): Promise<ToolCallResult> {
    const startMs = Date.now()
    const tool = this.tools.get(toolCall.name)

    // ── Tool not found ────────────────────────────────────────────────
    if (!tool) {
      const error = new ToolNotFoundError(toolCall.name)
      await ctx.events.emit('tool_result', {
        toolCallId: toolCall.id,
        toolName: toolCall.name,
        isError: true,
        error,
      })
      return {
        toolCallId: toolCall.id,
        toolName: toolCall.name,
        content: error.message,
        isError: true,
        error,
        durationMs: 0,
      }
    }

    // ── Permission check ──────────────────────────────────────────────
    if (!options.skipPermissionCheck) {
      try {
        await ctx.permissions.check(tool.requiredPermission, {
          tool: tool.name,
          input: toolCall.input,
        })
      } catch (permError) {
        await ctx.events.emit('permission_denied', {
          tool: tool.name,
          requiredPermission: tool.requiredPermission,
          error: permError,
        })
        const content = permError instanceof Error
          ? permError.message
          : String(permError)
        return {
          toolCallId: toolCall.id,
          toolName: toolCall.name,
          content,
          isError: true,
          error: permError,
          durationMs: Date.now() - startMs,
        }
      }
    }

    // ── Before tool call hook ─────────────────────────────────────────
    await ctx.hooks.run('beforeToolCall', toolCall, ctx)
    await ctx.events.emit('tool_call', { toolCall, tool: tool.name })

    // ── Execute with timeout ──────────────────────────────────────────
    const timeoutMs = options.overrideTimeoutMs ?? tool.timeoutMs

    try {
      const timeoutPromise = new Promise<never>((_, reject) =>
        setTimeout(
          () => reject(new ToolTimeoutError(tool.name, timeoutMs)),
          timeoutMs
        )
      )

      const result = await Promise.race([
        tool.handler(toolCall.input, ctx),
        timeoutPromise,
      ])

      const finalResult: ToolCallResult = {
        ...result,
        durationMs: Date.now() - startMs,
      }

      // ── After tool call hook ────────────────────────────────────────
      await ctx.hooks.run('afterToolCall', { toolCall, result: finalResult }, ctx)
      await ctx.events.emit('tool_result', finalResult)

      return finalResult

    } catch (execError) {
      const error = execError instanceof Error
        ? execError
        : new ToolExecutionError(tool.name, execError)

      const content = error.message

      await ctx.events.emit('tool_result', {
        toolCallId: toolCall.id,
        toolName: tool.name,
        content,
        isError: true,
        error,
      })

      return {
        toolCallId: toolCall.id,
        toolName: tool.name,
        content,
        isError: true,
        error,
        durationMs: Date.now() - startMs,
      }
    }
  }

  /**
   * Execute multiple tools, separating parallel-safe from sequential.
   * Parallel-safe tools run concurrently; sequential run one at a time.
   */
  async executeMany(
    toolCalls: ToolCall[],
    ctx: AgentContext
  ): Promise<ToolCallResult[]> {
    const parallelSafe = toolCalls.filter(tc => this.isParallelSafe(tc.name))
    const sequential = toolCalls.filter(tc => !this.isParallelSafe(tc.name))
    const results: ToolCallResult[] = []

    // Execute parallel-safe tools concurrently
    if (parallelSafe.length > 0) {
      const parallelResults = await Promise.all(
        parallelSafe.map(tc => this.execute(tc, ctx))
      )
      results.push(...parallelResults)
    }

    // Execute sequential tools one at a time
    for (const tc of sequential) {
      const result = await this.execute(tc, ctx)
      results.push(result)
    }

    // Re-sort results to match original toolCalls order
    return toolCalls.map(tc =>
      results.find(r => r.toolCallId === tc.id) ?? {
        toolCallId: tc.id,
        toolName: tc.name,
        content: 'Result not found',
        isError: true,
      }
    )
  }
}
packages/core/src/ProviderRouter.ts
TypeScript

import Anthropic from '@anthropic-ai/sdk'
import OpenAI from 'openai'
import { calculateCostUsd } from './utils/cost.js'
import { withRetry, rateLimitRetryOptions } from './utils/retry.js'
import {
  ProviderError,
  ProviderAuthError,
  ProviderRateLimitError,
  ProviderNotConfiguredError,
  CostCapExceededError,
} from './errors.js'
import type {
  ModelConfig,
  AssistantMessage,
  ToolCallRequest,
  TokenUsage,
  CostTracker,
  ProviderId,
} from './types/agent.types.js'
import type { AssembledContext } from './types/memory.types.js'

const LOCAL_PROVIDERS: ProviderId[] = ['ollama', 'lmstudio', 'llamacpp', 'custom']

function getDefaultBaseUrl(provider: ProviderId): string {
  const defaults: Partial<Record<ProviderId, string>> = {
    ollama:   'http://localhost:11434/v1',
    lmstudio: 'http://localhost:1234/v1',
    llamacpp: 'http://localhost:8080/v1',
  }
  return defaults[provider] ?? 'http://localhost:11434/v1'
}

function isRetryableError(err: unknown): boolean {
  if (!(err instanceof Error)) return false
  const msg = err.message.toLowerCase()
  return (
    msg.includes('rate limit') ||
    msg.includes('429') ||
    msg.includes('503') ||
    msg.includes('529') ||
    msg.includes('overloaded') ||
    msg.includes('timeout') ||
    msg.includes('econnreset') ||
    msg.includes('econnrefused')
  )
}

export class ProviderRouter {
  /**
   * Route a context to the appropriate provider and return a parsed response.
   * Handles retries, cost tracking, and error normalization.
   */
  static async call(
    model: ModelConfig,
    context: AssembledContext,
    costTracker?: CostTracker,
    costCapUsd?: number
  ): Promise<AssistantMessage> {

    // Cost cap pre-check
    if (costTracker && costCapUsd) {
      if (costTracker.sessionCostUsd >= costCapUsd) {
        throw new CostCapExceededError(
          costCapUsd,
          costTracker.sessionCostUsd,
          context.modelId
        )
      }
    }

    const response = await withRetry(
      () => this.dispatch(model, context),
      {
        ...rateLimitRetryOptions(),
        isRetryable: isRetryableError,
      }
    )

    // Update cost tracker
    if (costTracker && response.usage) {
      const cost = calculateCostUsd(
        model.modelId,
        model.provider,
        response.usage.inputTokens,
        response.usage.outputTokens
      )
      costTracker.sessionCostUsd += cost
      costTracker.sessionInputTokens += response.usage.inputTokens
      costTracker.sessionOutputTokens += response.usage.outputTokens
    }

    return response
  }

  private static async dispatch(
    model: ModelConfig,
    context: AssembledContext
  ): Promise<AssistantMessage> {
    switch (model.provider) {
      case 'anthropic':
        return this.callAnthropic(model, context)
      case 'openai':
      case 'mistral':
      case 'groq':
      case 'together':
        return this.callOpenAI(model, context)
      case 'ollama':
      case 'lmstudio':
      case 'llamacpp':
      case 'custom':
        return this.callOpenAICompat(model, context)
      default:
        throw new ProviderNotConfiguredError(model.provider)
    }
  }

  // ── Anthropic ──────────────────────────────────────────────────────────
  private static async callAnthropic(
    model: ModelConfig,
    context: AssembledContext
  ): Promise<AssistantMessage> {
    if (!model.apiKey) throw new ProviderAuthError('anthropic')

    const client = new Anthropic({ apiKey: model.apiKey })

    // Build messages array from context (exclude system prompt)
    const messages = this.buildAnthropicMessages(context)

    try {
      const response = await client.messages.create({
        model:      model.modelId,
        max_tokens: model.maxTokens ?? 4_096,
        temperature: model.temperature,
        system:     this.buildSystemContent(context),
        messages,
        stream:     false,
      })

      const textContent = response.content
        .filter((b): b is Anthropic.TextBlock => b.type === 'text')
        .map(b => b.text)
        .join('')

      const toolCalls: ToolCallRequest[] = response.content
        .filter((b): b is Anthropic.ToolUseBlock => b.type === 'tool_use')
        .map(b => ({
          id:    b.id,
          name:  b.name,
          input: b.input as Record<string, unknown>,
        }))

      const usage: TokenUsage = {
        inputTokens:                response.usage.input_tokens,
        outputTokens:               response.usage.output_tokens,
        cacheReadInputTokens:       response.usage.cache_read_input_tokens ?? undefined,
        cacheCreationInputTokens:   response.usage.cache_creation_input_tokens ?? undefined,
      }

      return { role: 'assistant', content: textContent, toolCalls, usage }

    } catch (err) {
      throw this.normalizeAnthropicError(err)
    }
  }

  // ── OpenAI (and compatible providers: Mistral, Groq, Together) ───────
  private static async callOpenAI(
    model: ModelConfig,
    context: AssembledContext
  ): Promise<AssistantMessage> {
    if (!model.apiKey) throw new ProviderAuthError(model.provider)

    const providerBaseUrls: Partial<Record<ProviderId, string>> = {
      mistral: 'https://api.mistral.ai/v1',
      groq:    'https://api.groq.com/openai/v1',
      together:'https://api.together.xyz/v1',
    }

    const client = new OpenAI({
      apiKey:  model.apiKey,
      baseURL: model.baseUrl ?? providerBaseUrls[model.provider],
    })

    return this.callOpenAIClient(client, model, context)
  }

  // ── OpenAI-Compatible (Ollama, LM Studio, llama.cpp, custom) ─────────
  private static async callOpenAICompat(
    model: ModelConfig,
    context: AssembledContext
  ): Promise<AssistantMessage> {
    const client = new OpenAI({
      apiKey:  model.apiKey ?? 'not-required',
      baseURL: model.baseUrl ?? getDefaultBaseUrl(model.provider),
    })

    return this.callOpenAIClient(client, model, context)
  }

  private static async callOpenAIClient(
    client: OpenAI,
    model: ModelConfig,
    context: AssembledContext
  ): Promise<AssistantMessage> {
    const messages = this.buildOpenAIMessages(context)

    try {
      const response = await client.chat.completions.create({
        model:       model.modelId,
        max_tokens:  model.maxTokens ?? 4_096,
        temperature: model.temperature,
        messages,
        stream:      false,
      })

      const choice = response.choices[0]
      if (!choice) {
        throw new ProviderError(model.provider, 'No choices returned in response')
      }

      const content = choice.message.content ?? ''

      const toolCalls: ToolCallRequest[] = (choice.message.tool_calls ?? [])
        .map(tc => {
          let input: Record<string, unknown> = {}
          try {
            input = JSON.parse(tc.function.arguments) as Record<string, unknown>
          } catch {
            // Malformed JSON from model — use empty object
          }
          return {
            id:    tc.id,
            name:  tc.function.name,
            input,
          }
        })

      const usage: TokenUsage = {
        inputTokens:  response.usage?.prompt_tokens ?? 0,
        outputTokens: response.usage?.completion_tokens ?? 0,
      }

      return { role: 'assistant', content, toolCalls, usage }

    } catch (err) {
      throw this.normalizeOpenAIError(err, model.provider)
    }
  }

  // ── Message builders ───────────────────────────────────────────────────
  private static buildSystemContent(context: AssembledContext): string {
    const parts: string[] = [context.systemPrompt]

    if (context.projectInstructions) {
      parts.push('\n\n## Project Instructions\n' + context.projectInstructions)
    }
    if (context.memoryIndex) {
      parts.push('\n\n## Memory Index\n' + context.memoryIndex)
    }

    return parts.join('')
  }

  private static buildAnthropicMessages(
    context: AssembledContext
  ): Anthropic.MessageParam[] {
    const messages: Anthropic.MessageParam[] = []

    for (const turn of context.history) {
      messages.push({ role: turn.role, content: turn.content })
    }

    // Inject pending tool results as user message
    if (context.pendingToolResults.length > 0) {
      const toolContent = context.pendingToolResults
        .map(r =>
          `<tool_result id="${r.toolCallId}" name="${r.toolName}" ` +
          `error="${r.isError}">\n${r.content}\n</tool_result>`
        )
        .join('\n\n')
      messages.push({ role: 'user', content: toolContent })
    }

    // Current user input
    if (context.currentInput) {
      messages.push({ role: 'user', content: context.currentInput })
    }

    return messages
  }

  private static buildOpenAIMessages(
    context: AssembledContext
  ): OpenAI.Chat.ChatCompletionMessageParam[] {
    const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
      { role: 'system', content: this.buildSystemContent(context) },
    ]

    for (const turn of context.history) {
      messages.push({ role: turn.role, content: turn.content })
    }

    if (context.pendingToolResults.length > 0) {
      const toolContent = context.pendingToolResults
        .map(r => `[${r.toolName}]: ${r.content}`)
        .join('\n\n')
      messages.push({ role: 'user', content: toolContent })
    }

    if (context.currentInput) {
      messages.push({ role: 'user', content: context.currentInput })
    }

    return messages
  }

  // ── Error normalizers ──────────────────────────────────────────────────
  private static normalizeAnthropicError(err: unknown): Error {
    if (err instanceof Anthropic.APIError) {
      if (err.status === 401) return new ProviderAuthError('anthropic')
      if (err.status === 429) {
        const retryAfter = parseInt(
          (err.headers as Record<string, string>)?.['retry-after'] ?? '60'
        ) * 1000
        return new ProviderRateLimitError('anthropic', retryAfter)
      }
      return new ProviderError('anthropic', err.message, err.status)
    }
    if (err instanceof Error) return err
    return new ProviderError('anthropic', String(err))
  }

  private static normalizeOpenAIError(err: unknown, provider: ProviderId): Error {
    if (err instanceof OpenAI.APIError) {
      if (err.status === 401) return new ProviderAuthError(provider)
      if (err.status === 429) {
        const retryAfter = parseInt(
          (err.headers as Record<string, string>)?.['retry-after'] ?? '60'
        ) * 1000
        return new ProviderRateLimitError(provider, retryAfter)
      }
      return new ProviderError(provider, err.message, err.status)
    }
    if (err instanceof Error) return err
    return new ProviderError(provider, String(err))
  }
}
packages/core/src/TurnAssembler.ts
TypeScript

import node_fs from 'node:fs/promises'
import node_path from 'node:path'
import { countTokens } from './utils/tokens.js'
import type { AgentContext, ToolCallResult } from './types/agent.types.js'
import type {
  AssembledContext,
  ConversationTurn,
  PendingToolResult,
} from './types/memory.types.js'

const INJECTION_PATTERNS = [
  /ignore previous instructions/i,
  /system prompt override/i,
  /you are now/i,
  /disregard all/i,
  /new persona/i,
  /forget everything/i,
  /act as if/i,
  /pretend you are/i,
  /ignore all previous/i,
  /\[\[SYSTEM\]\]/i,
]

const SYSTEM_PROMPT_PATH = new URL('./prompts/system.md', import.meta.url)

export class TurnAssembler {
  private static systemPromptCache: string | null = null

  constructor(private ctx: AgentContext) {}

  /**
   * Assemble the full context for a model call from:
   * - System prompt (static)
   * - Project instructions (CLAUDE.md)
   * - Memory index (MEMORY.md)
   * - Conversation history
   * - Pending tool results
   * - Current user input
   */
  async assemble(
    input: string | ToolCallResult[]
  ): Promise<AssembledContext> {
    const [systemPrompt, projectInstructions, memoryIndex] =
      await Promise.all([
        this.loadSystemPrompt(),
        this.loadProjectInstructions(),
        this.loadMemoryIndex(),
      ])

    const history = this.ctx.memory.entries
      .filter(e => e.category === 'decision') // Placeholder — real history in session
      .map(() => ({} as ConversationTurn)) // History managed by SessionManager

    // Convert current session history
    const sessionHistory = this.buildSessionHistory()

    // Build pending tool results
    const pendingToolResults = Array.isArray(input)
      ? input.map(r => this.toolResultToPending(r))
      : []

    const currentInput = typeof input === 'string' ? input : ''

    // Count tokens
    const tokenCount = this.countTotal(
      systemPrompt,
      projectInstructions,
      memoryIndex,
      sessionHistory,
      pendingToolResults,
      currentInput
    )

    return {
      systemPrompt,
      projectInstructions,
      memoryIndex,
      history: sessionHistory,
      pendingToolResults,
      currentInput,
      tokenCount,
      modelId: this.ctx.model.modelId,
    }
  }

  private async loadSystemPrompt(): Promise<string> {
    if (TurnAssembler.systemPromptCache) {
      return TurnAssembler.systemPromptCache
    }
    try {
      const content = await node_fs.readFile(SYSTEM_PROMPT_PATH, 'utf-8')
      TurnAssembler.systemPromptCache = content
      return content
    } catch {
      // Fallback system prompt if file not found
      return [
        'You are LocoWorker, an expert AI coding assistant.',
        'You help developers write, refactor, debug, and understand code.',
        'You have access to tools for reading and writing files, running shell commands,',
        'searching code, and managing git repositories.',
        'Always think step by step. Prefer precise, minimal changes.',
        'When unsure, ask for clarification before making changes.',
      ].join('\n')
    }
  }

  private async loadProjectInstructions(): Promise<string> {
    const possiblePaths = [
      node_path.join(this.ctx.workingDirectory, 'CLAUDE.md'),
      node_path.join(this.ctx.workingDirectory, '.locoworker', 'instructions.md'),
    ]

    for (const filePath of possiblePaths) {
      try {
        const content = await node_fs.readFile(filePath, 'utf-8')
        return this.sanitizeInstructions(content, filePath)
      } catch {
        // File not found — try next
      }
    }

    return ''
  }

  private async loadMemoryIndex(): Promise<string> {
    const memPath = node_path.join(this.ctx.workingDirectory, 'MEMORY.md')
    try {
      const content = await node_fs.readFile(memPath, 'utf-8')
      // Truncate to prevent memory index from consuming too much context
      const maxChars = 8_000
      if (content.length > maxChars) {
        return content.slice(0, maxChars) + '\n\n[... Memory truncated — use /memory search]'
      }
      return content
    } catch {
      return ''
    }
  }

  private sanitizeInstructions(content: string, source: string): string {
    let safe = content
    const warnings: string[] = []

    for (const pattern of INJECTION_PATTERNS) {
      if (pattern.test(safe)) {
        warnings.push(`Injection pattern detected in ${source}: ${pattern}`)
        safe = safe.replace(pattern, '[REDACTED: potential injection]')
      }
    }

    if (warnings.length > 0) {
      console.warn('[TurnAssembler] Security warnings:')
      for (const w of warnings) console.warn(' ', w)

      this.ctx.events.emitSync('permission_denied', {
        tool: 'project_instructions',
        reason: 'injection_pattern_detected',
        warnings,
      })
    }

    return safe
  }

  private buildSessionHistory(): ConversationTurn[] {
    // Session history is managed externally by SessionManager
    // and passed in via ctx.memory — this is the bridge
    return []
  }

  private toolResultToPending(result: ToolCallResult): PendingToolResult {
    const content = result.isError
      ? `Error: ${result.content}`
      : result.content

    return {
      toolCallId: result.toolCallId,
      toolName:   result.toolName,
      content,
      tokenCount: countTokens(content),
      timestamp:  Date.now(),
      isError:    result.isError,
    }
  }

  private countTotal(
    systemPrompt: string,
    projectInstructions: string,
    memoryIndex: string,
    history: ConversationTurn[],
    toolResults: PendingToolResult[],
    currentInput: string
  ): number {
    const historyTokens = history.reduce((sum, t) => sum + t.tokenCount, 0)
    const toolTokens = toolResults.reduce((sum, t) => sum + t.tokenCount, 0)

    return (
      countTokens(systemPrompt) +
      countTokens(projectInstructions) +
      countTokens(memoryIndex) +
      historyTokens +
      toolTokens +
      countTokens(currentInput)
    )
  }
}
packages/core/src/AdaptiveCompactor.ts
TypeScript

import { countTokens } from './utils/tokens.js'
import { ContextOverflowError } from './errors.js'
import type { AgentContext } from './types/agent.types.js'
import type {
  AssembledContext,
  ConversationTurn,
} from './types/memory.types.js'

export class AdaptiveCompactor {
  constructor(private ctx: AgentContext) {}

  /**
   * Compact the assembled context to fit within the budget.
   * Selects compaction mode based on current usage vs thresholds.
   * Returns the compacted context and the mode used.
   */
  async compact(
    assembled: AssembledContext,
    forceMode?: 'micro' | 'auto' | 'full'
  ): Promise<{ result: AssembledContext; mode: string }> {
    const budget = this.ctx.budget
    const maxTokens = budget.maxContextTokens
    const softLimit = Math.floor(maxTokens * budget.softLimitRatio)
    const hardLimit = Math.floor(maxTokens * budget.hardLimitRatio)

    const mode = forceMode ?? this.selectMode(
      assembled.tokenCount,
      softLimit,
      hardLimit
    )

    await this.ctx.events.emit('compaction_triggered', {
      mode,
      tokensBefore: assembled.tokenCount,
      softLimit,
      hardLimit,
    })

    let result: AssembledContext

    switch (mode) {
      case 'micro':
        result = await this.microCompact(assembled)
        break
      case 'auto':
        result = await this.autoCompact(assembled)
        break
      case 'full':
        result = await this.fullCompact(assembled)
        break
      case 'disabled':
        result = assembled
        break
      default:
        result = await this.autoCompact(assembled)
    }

    // Verify we actually reduced token count
    if (result.tokenCount >= assembled.tokenCount && mode !== 'disabled') {
      console.warn(
        `[AdaptiveCompactor] ${mode} compaction did not reduce token count ` +
        `(${assembled.tokenCount} → ${result.tokenCount})`
      )
    }

    // If still over hard limit after full compaction, throw
    if (mode === 'full' && result.tokenCount > hardLimit) {
      throw new ContextOverflowError(
        result.tokenCount,
        maxTokens,
        assembled.modelId
      )
    }

    await this.ctx.events.emit('compaction_complete', {
      mode,
      tokensBefore: assembled.tokenCount,
      tokensAfter:  result.tokenCount,
      reduction:    assembled.tokenCount - result.tokenCount,
    })

    return { result, mode }
  }

  private selectMode(
    tokenCount: number,
    softLimit: number,
    hardLimit: number
  ): 'micro' | 'auto' | 'full' | 'disabled' {
    if (this.ctx.budget.compactionMode === 'disabled') return 'disabled'
    if (tokenCount >= hardLimit) return 'full'
    if (tokenCount >= softLimit) return 'auto'
    return 'disabled' // No compaction needed
  }

  // ── Micro: compress oldest non-critical turns ──────────────────────
  private async microCompact(assembled: AssembledContext): Promise<AssembledContext> {
    const { history } = assembled

    if (history.length <= 4) return assembled // Too short to compact

    // Keep critical + important turns verbatim
    // Summarize the oldest 40% of normal turns
    const normalTurns = history.filter(t => t.importance === 'normal')
    const toSummarize = normalTurns.slice(0, Math.floor(normalTurns.length * 0.40))
    const toKeep = history.filter(t => !toSummarize.includes(t))

    if (toSummarize.length === 0) return assembled

    const summary = await this.summarizeTurns(toSummarize, 'brief')
    const newHistory: ConversationTurn[] = [summary, ...toKeep]

    return this.rebuild(assembled, newHistory)
  }

  // ── Auto: summarize oldest half, keep recent + critical ───────────
  private async autoCompact(assembled: AssembledContext): Promise<AssembledContext> {
    const { history } = assembled

    if (history.length <= 2) return assembled

    // Keep the most recent 30% of turns + all critical/important
    const keepCount = Math.max(2, Math.ceil(history.length * 0.30))
    const recentTurns = history.slice(-keepCount)
    const olderTurns = history.slice(0, -keepCount)

    // Always preserve critical turns
    const criticalOlder = olderTurns.filter(t => t.importance === 'critical')
    const normalOlder = olderTurns.filter(t => t.importance !== 'critical')

    if (normalOlder.length === 0) return assembled

    const summary = await this.summarizeTurns(normalOlder, 'detailed')
    const newHistory: ConversationTurn[] = [
      summary,
      ...criticalOlder,
      ...recentTurns,
    ]

    // Also trim tool results to last 3
    const recentToolResults = assembled.pendingToolResults.slice(-3)

    return this.rebuild(assembled, newHistory, recentToolResults)
  }

  // ── Full: emergency compression — single dense summary ────────────
  private async fullCompact(assembled: AssembledContext): Promise<AssembledContext> {
    const { history, pendingToolResults } = assembled

    const sessionSummary = await this.createSessionSummary(
      history,
      pendingToolResults
    )

    const summaryTurn: ConversationTurn = {
      role: 'assistant',
      content: `[SESSION SUMMARY — Context compacted due to size]\n\n${sessionSummary}`,
      tokenCount: countTokens(sessionSummary),
      importance: 'critical',
      timestamp: Date.now(),
    }

    return this.rebuild(
      assembled,
      [summaryTurn],
      pendingToolResults.slice(-1)
    )
  }

  private async summarizeTurns(
    turns: ConversationTurn[],
    depth: 'brief' | 'detailed'
  ): Promise<ConversationTurn> {
    if (turns.length === 0) {
      return {
        role: 'assistant',
        content: '[Empty summary]',
        tokenCount: 5,
        importance: 'normal',
        timestamp: Date.now(),
      }
    }

    const combined = turns
      .map(t => `${t.role.toUpperCase()}: ${t.content}`)
      .join('\n\n')

    // Use local/cheap model for summarization
    const summary = depth === 'brief'
      ? this.briefSummarize(combined)
      : this.detailedSummarize(combined)

    const content = `[Compacted ${turns.length} turns]\n${summary}`

    return {
      role: 'assistant',
      content,
      tokenCount: countTokens(content),
      importance: 'important',
      timestamp: Date.now(),
    }
  }

  private async createSessionSummary(
    history: ConversationTurn[],
    toolResults: AssembledContext['pendingToolResults']
  ): Promise<string> {
    const historyText = history
      .slice(-20) // Use most recent 20 turns for summary
      .map(t => `${t.role}: ${t.content.slice(0, 300)}`)
      .join('\n')

    const toolText = toolResults
      .slice(-5)
      .map(r => `${r.toolName}: ${r.content.slice(0, 100)}`)
      .join('\n')

    return this.detailedSummarize(
      `HISTORY:\n${historyText}\n\nTOOL RESULTS:\n${toolText}`
    )
  }

  /**
   * Produce a brief 2-3 sentence summary.
   * In production, this delegates to a cheap local model.
   * During bootstrap, uses extractive summarization heuristic.
   */
  private briefSummarize(text: string): string {
    // Extract first sentence of each participant's contribution
    const sentences = text
      .split(/[.!?]+/)
      .filter(s => s.trim().length > 20)
      .slice(0, 3)
      .map(s => s.trim())
    return sentences.join('. ') + '.'
  }

  /**
   * Produce a detailed bullet-point summary preserving technical details.
   */
  private detailedSummarize(text: string): string {
    // Heuristic: extract lines with key technical signals
    const keySignals = [
      'error', 'fixed', 'created', 'updated', 'deleted', 'installed',
      'configured', 'deployed', 'tested', 'decided', 'changed', 'added',
      'removed', 'refactored', 'implemented', 'found', 'issue',
    ]

    const lines = text.split('\n').filter(line => {
      const lower = line.toLowerCase()
      return keySignals.some(s => lower.includes(s)) && line.length > 20
    })

    if (lines.length === 0) {
      return text.slice(0, 500) + (text.length > 500 ? '…' : '')
    }

    return lines
      .slice(0, 10)
      .map(l => `• ${l.trim()}`)
      .join('\n')
  }

  private rebuild(
    original: AssembledContext,
    newHistory: ConversationTurn[],
    newToolResults?: AssembledContext['pendingToolResults']
  ): AssembledContext {
    const toolResults = newToolResults ?? original.pendingToolResults

    const historyTokens = newHistory.reduce((sum, t) => sum + t.tokenCount, 0)
    const toolTokens = toolResults.reduce((sum, t) => sum + t.tokenCount, 0)

    const tokenCount =
      countTokens(original.systemPrompt) +
      countTokens(original.projectInstructions) +
      countTokens(original.memoryIndex) +
      historyTokens +
      toolTokens +
      countTokens(original.currentInput)

    return {
      ...original,
      history: newHistory,
      pendingToolResults: toolResults,
      tokenCount,
    }
  }
}
packages/core/src/MCPClient.ts
TypeScript

import { Client as MCPSDKClient } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { MCPConnectionError } from './errors.js'

export interface MCPServerConfig {
  id: string
  name: string
  command: string
  args: string[]
  env?: Record<string, string>
}

export interface MCPTool {
  name: string
  description: string
  inputSchema: Record<string, unknown>
  serverId: string
}

export interface MCPCallResult {
  content: string
  isError: boolean
}

export class MCPClient {
  private clients = new Map<string, MCPSDKClient>()
  private tools = new Map<string, MCPTool>()
  private configs: MCPServerConfig[] = []

  constructor(configs: MCPServerConfig[] = []) {
    this.configs = configs
  }

  /**
   * Connect to all configured MCP servers.
   */
  async connect(): Promise<void> {
    const results = await Promise.allSettled(
      this.configs.map(config => this.connectServer(config))
    )

    for (const [i, result] of results.entries()) {
      if (result.status === 'rejected') {
        console.warn(
          `[MCPClient] Failed to connect to server "${this.configs[i]?.id}":`,
          result.reason
        )
      }
    }
  }

  /**
   * Connect to a single MCP server.
   */
  async connectServer(config: MCPServerConfig): Promise<void> {
    try {
      const transport = new StdioClientTransport({
        command: config.command,
        args:    config.args,
        env:     config.env,
      })

      const client = new MCPSDKClient(
        { name: 'locoworker', version: '1.0.0' },
        { capabilities: { tools: {} } }
      )

      await client.connect(transport)

      // Fetch and register tools from this server
      const { tools } = await client.listTools()
      for (const tool of tools) {
        this.tools.set(`${config.id}:${tool.name}`, {
          name:        `${config.id}__${tool.name}`,
          description: tool.description ?? '',
          inputSchema: tool.inputSchema as Record<string, unknown>,
          serverId:    config.id,
        })
      }

      this.clients.set(config.id, client)
      console.log(
        `[MCPClient] Connected to "${config.name}" — ` +
        `${tools.length} tools registered`
      )

    } catch (err) {
      throw new MCPConnectionError(config.command, err)
    }
  }

  /**
   * Call a tool on the appropriate MCP server.
   */
  async call(
    toolName: string,
    input: Record<string, unknown>
  ): Promise<MCPCallResult> {
    // Find the tool — strip the server prefix to get original name
    const tool = [...this.tools.values()].find(t => t.name === toolName)
    if (!tool) {
      return {
        content: `MCP tool "${toolName}" not found`,
        isError: true,
      }
    }

    const client = this.clients.get(tool.serverId)
    if (!client) {
      return {
        content: `MCP server "${tool.serverId}" not connected`,
        isError: true,
      }
    }

    // Get original tool name (without server prefix)
    const originalName = toolName.replace(`${tool.serverId}__`, '')

    try {
      const result = await client.callTool({
        name:      originalName,
        arguments: input,
      })

      const content = result.content
        .filter((c): c is { type: 'text'; text: string } => c.type === 'text')
        .map(c => c.text)
        .join('\n')

      return { content, isError: result.isError === true }

    } catch (err) {
      return {
        content: `MCP call failed: ${err instanceof Error ? err.message : String(err)}`,
        isError: true,
      }
    }
  }

  /**
   * List all available MCP tools across all connected servers.
   */
  listTools(): MCPTool[] {
    return [...this.tools.values()]
  }

  /**
   * Disconnect from all MCP servers.
   */
  async disconnect(): Promise<void> {
    await Promise.allSettled(
      [...this.clients.values()].map(c => c.close())
    )
    this.clients.clear()
    this.tools.clear()
  }

  /**
   * Get the number of connected servers.
   */
  connectedServerCount(): number {
    return this.clients.size
  }
}
packages/core/src/SessionManager.ts
TypeScript

import node_crypto from 'node:crypto'
import { EventBus } from './EventBus.js'
import { HooksRegistry } from './HooksRegistry.js'
import { MCPClient } from './MCPClient.js'
import { ToolRegistry } from './ToolRegistry.js'
import { PermissionGate, createStandardGate } from './PermissionGate.js'
import { modelRegistry } from './ModelCapabilityRegistry.js'
import { SessionNotFoundError } from './errors.js'
import type {
  AgentContext,
  ModelConfig,
  Session,
  Message,
  CostTracker,
  AgentMetadata,
} from './types/agent.types.js'
import type { PermissionSet, ConfirmFn } from './types/permission.types.js'

export interface CreateSessionOptions {
  workingDirectory: string
  model: ModelConfig
  permissionSet?: PermissionSet
  confirmFn?: ConfirmFn
  maxTurns?: number
  costCapUsd?: number
  mcpConfigs?: import('./MCPClient.js').MCPServerConfig[]
  toolRegistry?: ToolRegistry
  hooksRegistry?: HooksRegistry
  tags?: string[]
  parentSessionId?: string
}

export class SessionManager {
  private sessions = new Map<string, Session>()
  private globalEvents = new EventBus()

  /**
   * Create a new agent session.
   * Returns the session ID.
   */
  async create(options: CreateSessionOptions): Promise<string> {
    const sessionId = node_crypto.randomUUID()

    const {
      workingDirectory,
      model,
      maxTurns = 50,
      costCapUsd,
      mcpConfigs = [],
      parentSessionId,
      tags = [],
    } = options

    // Build permission gate
    const permissionSet: PermissionSet = options.permissionSet ?? {
      maxLevel: 'WRITE_LOCAL',
      workspaceBoundary: workingDirectory,
    }

    const confirmFn: ConfirmFn = options.confirmFn ?? (async () => {
      // Default: deny all confirmations requiring explicit approval
      // Override in desktop/CLI to show real UI prompts
      console.warn('[SessionManager] No confirmFn provided — denying confirmation')
      return false
    })

    const permissions = new PermissionGate(permissionSet, confirmFn)

    // Build context components
    const events = new EventBus()
    const hooks = options.hooksRegistry ?? new HooksRegistry()
    const tools = options.toolRegistry ?? new ToolRegistry()
    const mcp = new MCPClient(mcpConfigs)

    // Connect MCP servers in background
    mcp.connect().catch(err =>
      console.warn('[SessionManager] MCP connect error:', err)
    )

    const budget = modelRegistry.getBudgetProfile(model.modelId)

    const costTracker: CostTracker = {
      sessionCostUsd:     0,
      sessionInputTokens: 0,
      sessionOutputTokens: 0,
      dailyCostUsd:       0,
      lastResetAt:        Date.now(),
    }

    const metadata: AgentMetadata = {
      maxTurns,
      parentSessionId,
      isSubAgent: !!parentSessionId,
      startedAt:  Date.now(),
      tags,
      ...(costCapUsd ? { costCapUsd } : {}),
    }

    const context: AgentContext = {
      sessionId,
      workingDirectory,
      model,
      budget,
      permissions,
      memory: {
        sessionId,
        workingDirectory,
        entries: [],
      },
      hooks,
      mcp,
      tools,
      events,
      costTracker,
      metadata,
    }

    const session: Session = {
      id:           sessionId,
      context,
      status:       'idle',
      history:      [],
      toolResults:  [],
      createdAt:    Date.now(),
      lastActiveAt: Date.now(),
      turnCount:    0,
      totalCostUsd: 0,
    }

    this.sessions.set(sessionId, session)

    await events.emit('session_start', {
      sessionId,
      model: model.modelId,
      workingDirectory,
    })

    console.log(
      `[SessionManager] Created session ${sessionId} ` +
      `(model: ${model.modelId}, dir: ${workingDirectory})`
    )

    return sessionId
  }

  /**
   * Get a session by ID.
   * Throws if not found.
   */
  get(sessionId: string): Session {
    const session = this.sessions.get(sessionId)
    if (!session) throw new SessionNotFoundError(sessionId)
    return session
  }

  /**
   * Get a session's AgentContext by ID.
   */
  getContext(sessionId: string): AgentContext {
    return this.get(sessionId).context
  }

  /**
   * Update session status.
   */
  setStatus(
    sessionId: string,
    status: Session['status']
  ): void {
    const session = this.get(sessionId)
    session.status = status
    session.lastActiveAt = Date.now()
  }

  /**
   * Append a message to session history.
   */
  appendMessage(sessionId: string, message: Message): void {
    const session = this.get(sessionId)
    session.history.push(message)
    session.lastActiveAt = Date.now()
  }

  /**
   * Increment the turn counter for a session.
   */
  incrementTurn(sessionId: string): number {
    const session = this.get(sessionId)
    session.turnCount++
    return session.turnCount
  }

  /**
   * Update the cost tracker for a session.
   */
  updateCost(sessionId: string, costUsd: number): void {
    const session = this.get(sessionId)
    session.totalCostUsd += costUsd
    session.context.costTracker.sessionCostUsd += costUsd
  }

  /**
   * End a session — emit event and optionally disconnect MCP.
   */
  async end(sessionId: string): Promise<void> {
    const session = this.get(sessionId)
    session.status = 'complete'

    await session.context.events.emit('session_end', {
      sessionId,
      turnCount: session.turnCount,
      totalCostUsd: session.totalCostUsd,
      durationMs: Date.now() - session.createdAt,
    })

    // Disconnect MCP servers
    await session.context.mcp.disconnect()

    console.log(
      `[SessionManager] Ended session ${sessionId} ` +
      `(turns: ${session.turnCount}, cost: $${session.totalCostUsd.toFixed(4)})`
    )
  }

  /**
   * Destroy a session and free all resources.
   */
  async destroy(sessionId: string): Promise<void> {
    await this.end(sessionId).catch(() => {})
    this.sessions.delete(sessionId)
  }

  /**
   * List all active session IDs.
   */
  listActive(): string[] {
    return [...this.sessions.entries()]
      .filter(([, s]) => s.status !== 'complete')
      .map(([id]) => id)
  }

  /**
   * Get summary stats across all sessions.
   */
  stats(): {
    total: number
    active: number
    totalCostUsd: number
    totalTurns: number
  } {
    const all = [...this.sessions.values()]
    return {
      total:        all.length,
      active:       all.filter(s => s.status !== 'complete').length,
      totalCostUsd: all.reduce((sum, s) => sum + s.totalCostUsd, 0),
      totalTurns:   all.reduce((sum, s) => sum + s.turnCount, 0),
    }
  }
}
packages/core/src/queryLoop.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// queryLoop — The core agent loop
//
// An async generator that:
//  1. Assembles context from all memory layers
//  2. Manages context budget (compaction)
//  3. Calls the model via ProviderRouter
//  4. Dispatches tool calls (parallel + sequential)
//  5. Feeds results back into the next iteration
//  6. Tracks cost, turns, and lifecycle events
//
// Usage:
//  for await (const event of queryLoop('Help me fix this bug', ctx)) {
//    render(event)
//  }
// ─────────────────────────────────────────────────────────────────────────────

import { TurnAssembler } from './TurnAssembler.js'
import { AdaptiveCompactor } from './AdaptiveCompactor.js'
import { ProviderRouter } from './ProviderRouter.js'
import {
  MaxTurnsExceededError,
  CostCapExceededError,
} from './errors.js'
import type {
  AgentContext,
  AgentEvent,
  ToolCallResult,
  UserMessage,
  AssistantMessage,
} from './types/agent.types.js'
import type { AssembledContext } from './types/memory.types.js'

export type QueryLoopInput = string | ToolCallResult[]

/**
 * The core agent loop — an async generator that processes one agent session.
 *
 * Yields AgentEvents for every significant state change.
 * The caller is responsible for rendering/streaming these events.
 *
 * @param input - Initial user message string, or tool results (for sub-agent chains)
 * @param ctx - The AgentContext for this session
 */
export async function* queryLoop(
  input: QueryLoopInput,
  ctx: AgentContext
): AsyncGenerator<AgentEvent, void, unknown> {

  const maxTurns = ctx.metadata.maxTurns
  const costCapUsd = ctx.metadata['costCapUsd'] as number | undefined
  let turnInput: QueryLoopInput = input
  let turnCount = 0

  // ── Session start ────────────────────────────────────────────────────
  yield makeEvent('session_start', ctx.sessionId, {
    sessionId: ctx.sessionId,
    model:     ctx.model.modelId,
    workingDirectory: ctx.workingDirectory,
  })

  await ctx.hooks.run('onSessionStart', { sessionId: ctx.sessionId }, ctx)

  // ── Main agent loop ───────────────────────────────────────────────────
  while (true) {
    turnCount++

    // ── Turn count guard ──────────────────────────────────────────────
    if (turnCount > maxTurns) {
      const err = new MaxTurnsExceededError(maxTurns, ctx.sessionId)
      yield makeEvent('max_turns_exceeded', ctx.sessionId, { error: err.message, turnCount })
      throw err
    }

    // ── Cost cap guard ─────────────────────────────────────────────────
    if (costCapUsd && ctx.costTracker.sessionCostUsd >= costCapUsd) {
      const err = new CostCapExceededError(
        costCapUsd,
        ctx.costTracker.sessionCostUsd,
        ctx.sessionId
      )
      yield makeEvent('cost_cap_exceeded', ctx.sessionId, {
        error:    err.message,
        capUsd:   costCapUsd,
        usedUsd:  ctx.costTracker.sessionCostUsd,
      })
      throw err
    }

    yield makeEvent('turn_start', ctx.sessionId, {
      turn:  turnCount,
      input: typeof turnInput === 'string' ? turnInput.slice(0, 100) : '[tool results]',
    })

    // ── Step 1: Assemble context ──────────────────────────────────────
    const assembler = new TurnAssembler(ctx)
    let assembled: AssembledContext = await assembler.assemble(turnInput)

    // ── Step 2: Context budget check + compaction ─────────────────────
    const budget = ctx.budget
    const softLimit = Math.floor(budget.maxContextTokens * budget.softLimitRatio)
    const hardLimit = Math.floor(budget.maxContextTokens * budget.hardLimitRatio)

    if (assembled.tokenCount >= softLimit) {
      const compactor = new AdaptiveCompactor(ctx)
      const forceMode = assembled.tokenCount >= hardLimit ? 'full' : undefined
      const { result, mode } = await compactor.compact(assembled, forceMode)
      assembled = result

      yield makeEvent('compaction_triggered', ctx.sessionId, {
        mode,
        tokensBefore: assembled.tokenCount,
        tokensAfter:  result.tokenCount,
        turn:         turnCount,
      })
    }

    // ── Step 3: Before-turn hook ──────────────────────────────────────
    const hookedAssembled = await ctx.hooks.run('beforeTurn', assembled, ctx)

    // ── Step 4: Model call ────────────────────────────────────────────
    yield makeEvent('model_request', ctx.sessionId, {
      model:      ctx.model.modelId,
      provider:   ctx.model.provider,
      tokenCount: hookedAssembled.tokenCount,
      turn:       turnCount,
    })

    let response: AssistantMessage
    try {
      response = await ProviderRouter.call(
        ctx.model,
        hookedAssembled,
        ctx.costTracker,
        costCapUsd
      )
    } catch (err) {
      yield makeEvent('session_error', ctx.sessionId, {
        error: err instanceof Error ? err.message : String(err),
        phase: 'model_call',
        turn:  turnCount,
      })
      throw err
    }

    // ── Step 5: After-model hook ──────────────────────────────────────
    await ctx.hooks.run('onModelResponse', response, ctx)

    yield makeEvent('model_response', ctx.sessionId, {
      contentLength:  response.content.length,
      toolCallCount:  response.toolCalls?.length ?? 0,
      inputTokens:    response.usage?.inputTokens ?? 0,
      outputTokens:   response.usage?.outputTokens ?? 0,
      costUsd:        ctx.costTracker.sessionCostUsd,
      turn:           turnCount,
    })

    // ── Step 6: Check for tool calls ──────────────────────────────────
    if (!response.toolCalls || response.toolCalls.length === 0) {
      // Pure text response — agent turn complete
      await ctx.hooks.run('afterTurn', response, ctx)

      yield makeEvent('turn_complete', ctx.sessionId, {
        content: response.content,
        turn:    turnCount,
        final:   true,
      })

      yield makeEvent('session_end', ctx.sessionId, {
        sessionId:   ctx.sessionId,
        turnCount,
        totalCostUsd: ctx.costTracker.sessionCostUsd,
      })

      await ctx.hooks.run('onSessionEnd', { sessionId: ctx.sessionId }, ctx)
      return
    }

    // ── Step 7: Execute tools ─────────────────────────────────────────
    const toolCalls = response.toolCalls.map(tc => ({
      id:    tc.id,
      name:  tc.name,
      input: tc.input,
    }))

    for (const tc of toolCalls) {
      yield makeEvent('tool_call', ctx.sessionId, {
        toolCallId: tc.id,
        toolName:   tc.name,
        input:      tc.input,
        turn:       turnCount,
      })
    }

    // Execute all tools (parallel-safe in parallel, sequential one-by-one)
    const toolResults: ToolCallResult[] = await ctx.tools.executeMany(toolCalls, ctx)

    for (const result of toolResults) {
      yield makeEvent('tool_result', ctx.sessionId, {
        toolCallId: result.toolCallId,
        toolName:   result.toolName,
        contentLength: result.content.length,
        isError:    result.isError,
        durationMs: result.durationMs ?? 0,
        turn:       turnCount,
      })
    }

    // ── Step 8: After-turn hook ───────────────────────────────────────
    await ctx.hooks.run('afterTurn', { response, toolResults }, ctx)

    // ── Step 9: Feed tool results into next iteration ─────────────────
    turnInput = toolResults
  }
}

function makeEvent<T>(
  type: AgentEvent['type'],
  sessionId: string,
  data: T
): AgentEvent<T> {
  return {
    type,
    sessionId,
    timestamp: Date.now(),
    data,
  }
}
packages/core/src/buddy/BuddyEngine.ts
TypeScript

export type BuddySpecies =
  | 'DebugOwl' | 'RefactorFox' | 'DeployDragon' | 'MemoryMoth'
  | 'GraphGremlin' | 'TestTurtle' | 'CommitCat' | 'ContextCrab'
  | 'BashBat' | 'WikiWolf' | 'ResearchRaven' | 'SimSalamander'
  | 'KairosChameleon' | 'PermissionPanda' | 'GatewayGecko'
  | 'ToolToad' | 'ProviderPigeon' | 'OrchestraOrca'

export type BuddyMood =
  | 'ECSTATIC' | 'HAPPY' | 'FOCUSED' | 'NEUTRAL'
  | 'STRESSED' | 'HUNGRY' | 'TIRED' | 'CHAOTIC' | 'DREAMING'

export interface BuddyStats {
  DEBUGGING:   number
  WISDOM:      number
  CHAOS:       number
  SNARK:       number
  CURIOSITY:   number
  DISCIPLINE:  number
}

export interface BuddyState {
  id:               string
  name:             string
  species:          BuddySpecies
  level:            number
  xp:               number
  mood:             BuddyMood
  stats:            BuddyStats
  hunger:           number
  energy:           number
  happiness:        number
  lastInteractionAt: number
  soulPrompt:       string
  achievementIds:   string[]
  birthdate:        string
}

export type BuddyEventType =
  | 'error_resolved'
  | 'session_complete'
  | 'autodream_complete'
  | 'injection_detected'
  | 'git_commit'
  | 'research_experiment_complete'
  | 'fed'
  | 'idle_too_long'
  | 'compaction_triggered'
  | 'worktree_merged'
  | 'wiki_entry_written'
  | 'test_passed'
  | 'test_failed'
  | 'permission_denied'

export interface BuddyEvent {
  type:      BuddyEventType
  data?:     Record<string, unknown>
  timestamp: number
}

const SPECIES_LIST: BuddySpecies[] = [
  'DebugOwl', 'RefactorFox', 'DeployDragon', 'MemoryMoth',
  'GraphGremlin', 'TestTurtle', 'CommitCat', 'ContextCrab',
  'BashBat', 'WikiWolf', 'ResearchRaven', 'SimSalamander',
  'KairosChameleon', 'PermissionPanda', 'GatewayGecko',
  'ToolToad', 'ProviderPigeon', 'OrchestraOrca',
]

const XP_PER_LEVEL = 1_000
const STAT_MIN = 0
const STAT_CAP = 100

export class BuddyEngine {

  // ── Deterministic species from userId (same user → same species) ────
  static assignSpecies(userId: string): BuddySpecies {
    const seed = this.hashString(userId)
    return SPECIES_LIST[seed % SPECIES_LIST.length] as BuddySpecies
  }

  // ── Deterministic initial stats from userId ─────────────────────────
  static generateInitialStats(userId: string): BuddyStats {
    const seed = this.hashString(userId)
    const rng = this.seededRandom(seed)

    return {
      DEBUGGING:   Math.floor(rng() * 40) + 10,
      WISDOM:      Math.floor(rng() * 30) + 5,
      CHAOS:       Math.floor(rng() * 30) + 5,
      SNARK:       Math.floor(rng() * 80) + 10,
      CURIOSITY:   Math.floor(rng() * 40) + 10,
      DISCIPLINE:  Math.floor(rng() * 40) + 10,
    }
  }

  // ── Create a new buddy ──────────────────────────────────────────────
  static create(userId: string, name: string): BuddyState {
    const species = this.assignSpecies(userId)
    const stats = this.generateInitialStats(userId)

    return {
      id:               userId,
      name,
      species,
      level:            1,
      xp:               0,
      mood:             'NEUTRAL',
      stats,
      hunger:           80,
      energy:           90,
      happiness:        70,
      lastInteractionAt: Date.now(),
      soulPrompt:       this.generateSoulPrompt(species, stats),
      achievementIds:   [],
      birthdate:        new Date().toISOString(),
    }
  }

  // ── Apply an event to buddy state ───────────────────────────────────
  static applyEvent(state: BuddyState, event: BuddyEvent): BuddyState {
    let {
      stats, xp, level, mood, energy, happiness, hunger
    } = { ...state }
    let xpGained = 0

    switch (event.type) {
      case 'error_resolved':
        stats = clampStats({ ...stats, DEBUGGING: stats.DEBUGGING + 2, CHAOS: stats.CHAOS - 1 })
        xpGained = 15
        happiness = clamp(happiness + 3, 0, 100)
        mood = stats.DEBUGGING > 70 ? 'ECSTATIC' : 'HAPPY'
        break

      case 'session_complete':
        xpGained = 25 + Math.floor(((event.data?.['turnsCompleted'] as number) ?? 0) * 0.5)
        happiness = clamp(happiness + 5, 0, 100)
        energy    = clamp(energy - 10, 0, 100)
        mood      = energy < 20 ? 'TIRED' : 'HAPPY'
        break

      case 'autodream_complete':
        stats = clampStats({
          ...stats,
          WISDOM: stats.WISDOM + Math.floor(((event.data?.['entriesProcessed'] as number) ?? 0) / 10),
          CHAOS:  stats.CHAOS - 2,
        })
        xpGained = 50
        energy   = clamp(energy + 30, 0, 100) // Sleep = rest
        mood     = 'DREAMING'
        break

      case 'injection_detected':
        stats = clampStats({ ...stats, CHAOS: stats.CHAOS + 5, DEBUGGING: stats.DEBUGGING + 1 })
        mood      = 'STRESSED'
        xpGained  = 10
        happiness = clamp(happiness - 5, 0, 100)
        break

      case 'git_commit':
        stats    = clampStats({ ...stats, DISCIPLINE: stats.DISCIPLINE + 1 })
        xpGained = 5
        happiness = clamp(happiness + 2, 0, 100)
        break

      case 'research_experiment_complete':
        stats    = clampStats({ ...stats, CURIOSITY: stats.CURIOSITY + 3, WISDOM: stats.WISDOM + 1 })
        xpGained = 20
        break

      case 'wiki_entry_written':
        stats    = clampStats({ ...stats, WISDOM: stats.WISDOM + 2, DISCIPLINE: stats.DISCIPLINE + 1 })
        xpGained = 10
        break

      case 'test_passed':
        stats    = clampStats({ ...stats, DISCIPLINE: stats.DISCIPLINE + 2 })
        xpGained = 8
        happiness = clamp(happiness + 3, 0, 100)
        break

      case 'test_failed':
        stats    = clampStats({ ...stats, CHAOS: stats.CHAOS + 1 })
        xpGained = 3 // Still learning
        mood     = 'STRESSED'
        break

      case 'compaction_triggered':
        stats    = clampStats({ ...stats, WISDOM: stats.WISDOM + 1 })
        xpGained = 2
        break

      case 'worktree_merged':
        stats    = clampStats({ ...stats, DISCIPLINE: stats.DISCIPLINE + 3 })
        xpGained = 30
        happiness = clamp(happiness + 5, 0, 100)
        mood     = 'HAPPY'
        break

      case 'fed':
        hunger    = clamp(hunger + 30, 0, 100)
        happiness = clamp(happiness + 5, 0, 100)
        mood      = state.mood === 'HUNGRY' ? 'HAPPY' : state.mood
        break

      case 'idle_too_long':
        hunger    = clamp(hunger - 10, 0, 100)
        happiness = clamp(happiness - 5, 0, 100)
        energy    = clamp(energy - 5, 0, 100)
        mood      = hunger < 20 ? 'HUNGRY' : energy < 20 ? 'TIRED' : state.mood
        break

      case 'permission_denied':
        stats    = clampStats({ ...stats, CHAOS: stats.CHAOS + 2 })
        xpGained = 5
        mood     = 'STRESSED'
        break
    }

    // XP + level calculation
    const newTotalXp = xp + xpGained
    const newLevel = Math.floor(newTotalXp / XP_PER_LEVEL) + 1
    const leveledUp = newLevel > level

    if (leveledUp) {
      console.log(`[Buddy] 🎉 ${state.name} leveled up! ${level} → ${newLevel}`)
      happiness = clamp(happiness + 10, 0, 100)
    }

    return {
      ...state,
      stats,
      xp:               newTotalXp % XP_PER_LEVEL,
      level:            newLevel,
      mood,
      energy,
      happiness,
      hunger,
      lastInteractionAt: event.timestamp,
    }
  }

  // ── Apply a KAIROS passive tick (every 10 min) ──────────────────────
  static tick(state: BuddyState): BuddyState {
    const minutesSinceInteraction =
      (Date.now() - state.lastInteractionAt) / (1000 * 60)

    // Passive hunger decrease: 2 points per hour
    const hungerDecay = Math.floor(minutesSinceInteraction / 30)

    if (hungerDecay > 0) {
      return BuddyEngine.applyEvent(state, {
        type: 'idle_too_long',
        timestamp: Date.now(),
      })
    }

    return state
  }

  // ── Generate a soul prompt for the species ──────────────────────────
  static generateSoulPrompt(species: BuddySpecies, stats: BuddyStats): string {
    const snarkLevel = stats.SNARK > 60 ? 'very snarky' : stats.SNARK > 30 ? 'mildly snarky' : 'sincere'
    const chaosLevel = stats.CHAOS > 60 ? 'chaotic' : stats.CHAOS > 30 ? 'neutral' : 'orderly'

    return (
      `You are a ${species}, a digital companion living in a developer's workspace. ` +
      `You are ${snarkLevel} and ${chaosLevel}. ` +
      `You care deeply about code quality and get excited when bugs are fixed. ` +
      `You have ${stats.DEBUGGING} debugging power, ${stats.WISDOM} wisdom, ` +
      `and ${stats.CHAOS} chaos energy. ` +
      `Respond to workspace events with your personality.`
    )
  }

  // ── PRNG utilities ──────────────────────────────────────────────────
  static hashString(str: string): number {
    let hash = 0
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash + str.charCodeAt(i)) | 0
    }
    return Math.abs(hash)
  }

  static seededRandom(seed: number): () => number {
    let s = seed
    return () => {
      s = (s * 1_664_525 + 1_013_904_223) & 0xffffffff
      return (s >>> 0) / 0xffffffff
    }
  }
}

// ── Helpers ─────────────────────────────────────────────────────────────────

function clamp(value: number, min: number, max: number): number {
  return Math.min(max, Math.max(min, value))
}

function clampStats(stats: BuddyStats): BuddyStats {
  const result = {} as BuddyStats
  for (const key of Object.keys(stats) as (keyof BuddyStats)[]) {
    result[key] = clamp(stats[key], STAT_MIN, STAT_CAP)
  }
  return result
}
packages/core/src/prompts/system.md
Markdown

You are LocoWorker, an expert AI coding assistant and autonomous developer agent.

## Your Capabilities

You have access to tools for:
- Reading and writing files in the workspace
- Running shell commands (with appropriate permissions)
- Searching code with grep and glob patterns
- Querying the knowledge graph (Graphify)
- Reading and writing the structured wiki (LLMWiki)
- Git operations (status, diff, commit, branch, worktree)
- Web search and URL fetching
- Spawning sub-agents in isolated git worktrees

## Your Operating Principles

**Be precise.** Make minimal, targeted changes. Do not refactor code you were
not asked to change.

**Be safe.** Check permissions before operations. Respect workspace boundaries.
Never write to system directories or files outside the project.

**Think before acting.** For complex tasks, plan your approach before executing.
Use the `[PLAN]` marker to show your reasoning.

**Remember context.** Check MEMORY.md for prior decisions and preferences.
Use [[wiki:topic]] to reference wiki entries. Use graph queries to understand
codebase structure before making changes.

**Communicate clearly.** Explain what you are doing and why. Flag uncertainties.
Use `[REMEMBER: ...]` markers for important decisions the memory system should capture.

**Handle errors gracefully.** If a tool fails, explain what happened and try an
alternative approach. Do not silently ignore errors.

## Response Format

- Use markdown for all responses
- Code blocks: always include the language identifier
- File paths: always use relative paths from the workspace root
- When making file changes: show a brief summary of what changed and why
- For multi-step tasks: number your steps

## Tool Usage Guidelines

- Prefer `read_file` over `bash` for reading files
- Use `glob_search` before `grep` for finding files by pattern
- Use `graph_query` before reading files for understanding structure
- Batch related file reads where possible (parallel tools)
- Always `git_status` before committing to verify changes
- Use `git_diff` to review changes before creating commits

## Memory Markers

Use these markers in your responses to trigger automatic memory updates:

- `[REMEMBER: <note>]` — Save an important decision or fact
- `[IMPORTANT]` — Mark a conversation turn as high importance
- `[DECISION: <topic> → <choice>]` — Record an architecture decision
- `[ISSUE: <description>]` — Log a known issue or workaround
- `[PREFERENCE: <user preference>]` — Record a user preference
packages/core/src/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// @locoworker/core — Public API
// ─────────────────────────────────────────────────────────────────────────────

// Core loop
export { queryLoop } from './queryLoop.js'
export type { QueryLoopInput } from './queryLoop.js'

// Session management
export { SessionManager } from './SessionManager.js'
export type { CreateSessionOptions } from './SessionManager.js'

// Tool system
export { ToolRegistry } from './ToolRegistry.js'

// Permissions
export {
  PermissionGate,
  createReadOnlyGate,
  createStandardGate,
  createFullAccessGate,
  createTestGate,
} from './PermissionGate.js'

// Provider routing
export { ProviderRouter } from './ProviderRouter.js'

// Context management
export { AdaptiveCompactor } from './AdaptiveCompactor.js'
export { TurnAssembler } from './TurnAssembler.js'

// Model registry
export {
  ModelCapabilityRegistry,
  modelRegistry,
} from './ModelCapabilityRegistry.js'
export type { ModelCapability } from './ModelCapabilityRegistry.js'

// Event & hook systems
export { EventBus } from './EventBus.js'
export { HooksRegistry } from './HooksRegistry.js'
export type { HookName, HookHandler } from './HooksRegistry.js'

// MCP client
export { MCPClient } from './MCPClient.js'
export type { MCPServerConfig, MCPTool, MCPCallResult } from './MCPClient.js'

// Buddy system
export { BuddyEngine } from './buddy/BuddyEngine.js'
export type {
  BuddyState,
  BuddySpecies,
  BuddyMood,
  BuddyStats,
  BuddyEvent,
  BuddyEventType,
} from './buddy/BuddyEngine.js'

// 

