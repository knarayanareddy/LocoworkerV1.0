Pass 17 — Part 1: packages/providers (Provider Registry Abstraction)
Division rationale:

Part 1 (this pass): packages/providers — the foundational registry abstraction, all provider adapter interfaces, routing strategies, capability catalog, cost tracking, health checking, and wiring points into packages/core. This comes first because tools-mcp (Part 2) registers itself through this registry, so the contracts must exist before the MCP package touches them.
Part 2 (next pass): packages/tools-mcp (full MCP client + tool registration skeleton) + openclaw/ placeholder package.
Rule: Complete final files. Later pass wins. No merging required.

Philosophy: This is a Pass 17 scaffolding pass — types and contracts are production-grade, adapter implementations are real but non-breaking stubs that the current packages/core ProviderRouter can delegate to incrementally. Pass 18 does the full cutover.

File tree (Pass 17 Part 1 — files created/touched)
text

locoworker/
├── packages/
│   └── providers/                              ← NEW package
│       ├── package.json                        ← NEW
│       ├── tsconfig.json                       ← NEW
│       ├── vitest.config.ts                    ← NEW
│       └── src/
│           ├── index.ts                        ← NEW (barrel export)
│           ├── types.ts                        ← NEW (all core contracts)
│           ├── capabilities.ts                 ← NEW (model capability catalog)
│           ├── cost.ts                         ← NEW (cost estimation + per-session tracker)
│           ├── health.ts                       ← NEW (provider health checker)
│           ├── registry.ts                     ← NEW (ProviderRegistry — main class)
│           ├── adapters/
│           │   ├── index.ts                    ← NEW
│           │   ├── base.ts                     ← NEW (BaseProviderAdapter abstract)
│           │   ├── anthropic.ts                ← NEW (AnthropicAdapter)
│           │   ├── openai.ts                   ← NEW (OpenAICompatibleAdapter — covers OAI + Groq + Mistral)
│           │   ├── gemini.ts                   ← NEW (GeminiAdapter)
│           │   ├── ollama.ts                   ← NEW (OllamaAdapter)
│           │   └── local.ts                    ← NEW (LocalAdapter — LM Studio + llama.cpp)
│           ├── routing/
│           │   ├── index.ts                    ← NEW
│           │   ├── types.ts                    ← NEW (routing strategy contracts)
│           │   ├── strategies.ts               ← NEW (single / round-robin / least-cost / least-latency)
│           │   └── router.ts                   ← NEW (ProviderRouter — canonical implementation)
│           └── __tests__/
│               ├── registry.test.ts            ← NEW
│               ├── routing.test.ts             ← NEW
│               └── cost.test.ts                ← NEW
└── packages/
    └── core/
        └── src/
            └── providers/
                └── bridge.ts                   ← NEW (wiring point — how core delegates to @locoworker/providers)
packages/providers/package.json
JSON

{
  "name": "@locoworker/providers",
  "version": "0.1.0",
  "description": "LocoWorker provider registry — adapters, routing strategies, capability catalog, cost tracking",
  "license": "MIT",
  "private": false,
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./adapters": {
      "import": "./dist/adapters/index.js",
      "types": "./dist/adapters/index.d.ts"
    },
    "./routing": {
      "import": "./dist/routing/index.js",
      "types": "./dist/routing/index.d.ts"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "typecheck": "tsc --noEmit",
    "dev": "tsc --watch --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome check src",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@locoworker/shared": "workspace:*"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "rimraf": "^6.0.1",
    "typescript": "^5.5.4",
    "vitest": "^2.0.5"
  },
  "peerDependencies": {
    "typescript": ">=5.5.0"
  }
}
packages/providers/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "composite": true,
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "src/**/__tests__/**"],
  "references": [
    { "path": "../shared" }
  ]
}
packages/providers/vitest.config.ts
TypeScript

import { defineConfig } from 'vitest/config';
import { resolve } from 'node:path';

export default defineConfig({
  test: {
    name: '@locoworker/providers',
    environment: 'node',
    globals: false,
    include: ['src/**/__tests__/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/__tests__/**', 'src/index.ts', 'src/adapters/index.ts', 'src/routing/index.ts'],
    },
  },
  resolve: {
    alias: {
      '@locoworker/shared': resolve(__dirname, '../shared/src/index.ts'),
    },
  },
});
packages/providers/src/types.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/types.ts
// Pass 17 Part 1 — Core provider contracts and types
//
// This file is the single source of truth for all provider-related types
// across the monorepo. packages/core's ProviderRouter delegates to these
// contracts. Pass 18 will wire core fully; for now, bridge.ts provides the
// wiring points.
// ─────────────────────────────────────────────────────────────────────────────

// ── Provider identity ─────────────────────────────────────────────────────────

/**
 * All supported provider IDs. Adding a new provider means adding it here
 * and implementing a corresponding ProviderAdapter.
 */
export type ProviderId =
  | 'anthropic'
  | 'openai'
  | 'gemini'
  | 'mistral'
  | 'groq'
  | 'cohere'
  | 'ollama'
  | 'lmstudio'
  | 'llamacpp'
  | 'mcp'           // MCP-sourced model endpoints (Pass 17 Part 2)
  | 'custom';       // Any OpenAI-compatible custom endpoint

/** Provider categories for routing and capability inference */
export type ProviderCategory = 'cloud' | 'local' | 'custom';

// ── Model identity ────────────────────────────────────────────────────────────

export interface ModelId {
  readonly provider: ProviderId;
  readonly model: string;
}

/** Canonical string form: "anthropic:claude-opus-4-5" */
export type ModelKey = `${ProviderId}:${string}`;

export function toModelKey(id: ModelId): ModelKey {
  return `${id.provider}:${id.model}` as ModelKey;
}

export function parseModelKey(key: ModelKey): ModelId {
  const colonIdx = key.indexOf(':');
  if (colonIdx === -1) throw new Error(`Invalid ModelKey: "${key}"`);
  return {
    provider: key.slice(0, colonIdx) as ProviderId,
    model: key.slice(colonIdx + 1),
  };
}

// ── Provider configuration ────────────────────────────────────────────────────

export interface ProviderConfig {
  readonly id: ProviderId;
  readonly category: ProviderCategory;
  /** Display name for UIs and logs */
  readonly displayName: string;
  /** API key (cloud providers) */
  readonly apiKey?: string;
  /** Base URL override (local providers or custom endpoints) */
  readonly baseUrl?: string;
  /** Default model to use when none specified */
  readonly defaultModel: string;
  /** Optional list of models to expose (subset of provider's catalog) */
  readonly models?: readonly string[];
  /** Whether this provider is currently enabled */
  readonly enabled: boolean;
  /** Request timeout in milliseconds */
  readonly timeoutMs: number;
  /** Max retry attempts on transient errors */
  readonly maxRetries: number;
  /** Additional provider-specific options */
  readonly options?: Record<string, unknown>;
}

/** Partial config used for registration/updates */
export type ProviderConfigInput = Partial<ProviderConfig> &
  Pick<ProviderConfig, 'id'>;

// ── Message and content types ─────────────────────────────────────────────────

export type MessageRole = 'user' | 'assistant' | 'system' | 'tool';

export interface TextContent {
  readonly type: 'text';
  readonly text: string;
}

export interface ImageContent {
  readonly type: 'image';
  readonly source:
    | { type: 'base64'; mediaType: string; data: string }
    | { type: 'url'; url: string };
}

export interface ToolUseContent {
  readonly type: 'tool_use';
  readonly id: string;
  readonly name: string;
  readonly input: Record<string, unknown>;
}

export interface ToolResultContent {
  readonly type: 'tool_result';
  readonly toolUseId: string;
  readonly content: string | readonly (TextContent | ImageContent)[];
  readonly isError?: boolean;
}

export type ContentBlock =
  | TextContent
  | ImageContent
  | ToolUseContent
  | ToolResultContent;

export interface Message {
  readonly role: MessageRole;
  readonly content: string | readonly ContentBlock[];
}

// ── Tool definitions (provider-facing) ───────────────────────────────────────

export interface ProviderToolDefinition {
  readonly name: string;
  readonly description: string;
  readonly inputSchema: Record<string, unknown>; // JSON Schema object
}

// ── Request / response contracts ─────────────────────────────────────────────

export interface ProviderRequestOptions {
  readonly model: string;
  readonly messages: readonly Message[];
  readonly system?: string;
  readonly tools?: readonly ProviderToolDefinition[];
  readonly maxTokens?: number;
  readonly temperature?: number;
  readonly topP?: number;
  readonly topK?: number;
  readonly stopSequences?: readonly string[];
  readonly stream?: boolean;
  /** Extra provider-specific params passed through verbatim */
  readonly extraParams?: Record<string, unknown>;
}

export interface UsageStats {
  readonly inputTokens: number;
  readonly outputTokens: number;
  readonly cacheReadTokens?: number;
  readonly cacheWriteTokens?: number;
}

export interface ProviderResponse {
  readonly id: string;
  readonly provider: ProviderId;
  readonly model: string;
  readonly content: readonly ContentBlock[];
  readonly stopReason:
    | 'end_turn'
    | 'tool_use'
    | 'max_tokens'
    | 'stop_sequence'
    | 'error'
    | string;
  readonly usage: UsageStats;
  readonly latencyMs: number;
  readonly rawResponse?: unknown;
}

// ── Streaming ─────────────────────────────────────────────────────────────────

export type StreamEvent =
  | { type: 'message_start'; message: Pick<ProviderResponse, 'id' | 'provider' | 'model'> }
  | { type: 'content_block_start'; index: number; contentBlock: ContentBlock }
  | { type: 'content_block_delta'; index: number; delta: TextContent | { type: 'input_json_delta'; partialJson: string } }
  | { type: 'content_block_stop'; index: number }
  | { type: 'message_delta'; stopReason: ProviderResponse['stopReason']; usage: UsageStats }
  | { type: 'message_stop' }
  | { type: 'error'; error: ProviderError };

// ── Health + availability ─────────────────────────────────────────────────────

export type ProviderHealthStatus = 'healthy' | 'degraded' | 'unhealthy' | 'unknown';

export interface ProviderHealth {
  readonly providerId: ProviderId;
  readonly status: ProviderHealthStatus;
  readonly latencyMs?: number;
  readonly lastCheckedAt: number;
  readonly error?: string;
}

// ── Error types ───────────────────────────────────────────────────────────────

export type ProviderErrorCode =
  | 'authentication_failed'
  | 'rate_limited'
  | 'quota_exceeded'
  | 'model_not_found'
  | 'context_too_long'
  | 'invalid_request'
  | 'provider_unavailable'
  | 'timeout'
  | 'stream_error'
  | 'unknown';

export class ProviderError extends Error {
  constructor(
    public readonly code: ProviderErrorCode,
    message: string,
    public readonly providerId: ProviderId,
    public readonly retryable: boolean,
    public readonly cause?: unknown,
    public readonly statusCode?: number,
  ) {
    super(message);
    this.name = 'ProviderError';
  }
}

// ── Registry events ───────────────────────────────────────────────────────────

export type ProviderRegistryEvent =
  | { type: 'provider_registered'; providerId: ProviderId }
  | { type: 'provider_unregistered'; providerId: ProviderId }
  | { type: 'provider_enabled'; providerId: ProviderId }
  | { type: 'provider_disabled'; providerId: ProviderId }
  | { type: 'health_updated'; health: ProviderHealth }
  | { type: 'request_completed'; providerId: ProviderId; model: string; usage: UsageStats; latencyMs: number }
  | { type: 'request_failed'; providerId: ProviderId; model: string; error: ProviderError };

export type ProviderRegistryListener = (event: ProviderRegistryEvent) => void;
packages/providers/src/capabilities.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/capabilities.ts
// Pass 17 Part 1 — Model capability catalog
//
// Describes what each known model can do so the router and core engine
// can make informed decisions (e.g. which model supports vision, extended
// context, tool use, caching, etc.) without making live API calls.
// ─────────────────────────────────────────────────────────────────────────────

import type { ModelKey, ProviderId } from './types.js';
import { toModelKey } from './types.js';

// ── Capability flags ──────────────────────────────────────────────────────────

export interface ModelCapabilities {
  /** Supports tool/function calling */
  readonly toolUse: boolean;
  /** Supports streaming responses */
  readonly streaming: boolean;
  /** Supports vision (image input) */
  readonly vision: boolean;
  /** Supports system prompt injection */
  readonly systemPrompt: boolean;
  /** Supports prompt caching (e.g. Anthropic cache_control) */
  readonly promptCaching: boolean;
  /** Max context window in tokens */
  readonly contextWindowTokens: number;
  /** Max output tokens */
  readonly maxOutputTokens: number;
  /** Whether the model supports JSON mode / structured output */
  readonly jsonMode: boolean;
}

// ── Pricing (per 1M tokens) ───────────────────────────────────────────────────

export interface ModelPricing {
  /** Cost per 1M input tokens (USD) */
  readonly inputCostPer1M: number;
  /** Cost per 1M output tokens (USD) */
  readonly outputCostPer1M: number;
  /** Cost per 1M cache-read tokens (USD), if supported */
  readonly cacheReadCostPer1M?: number;
  /** Cost per 1M cache-write tokens (USD), if supported */
  readonly cacheWriteCostPer1M?: number;
}

// ── Catalog entry ─────────────────────────────────────────────────────────────

export interface ModelCatalogEntry {
  readonly modelKey: ModelKey;
  readonly provider: ProviderId;
  readonly modelId: string;
  /** Human-readable display name */
  readonly displayName: string;
  readonly capabilities: ModelCapabilities;
  readonly pricing: ModelPricing;
  /** Whether this model is the recommended default for its provider */
  readonly isDefault?: boolean;
  /** Deprecation notice if applicable */
  readonly deprecated?: string;
}

// ── Catalog ───────────────────────────────────────────────────────────────────

const CATALOG: readonly ModelCatalogEntry[] = [
  // ── Anthropic ─────────────────────────────────────────────────────────────

  {
    modelKey: toModelKey({ provider: 'anthropic', model: 'claude-opus-4-5' }),
    provider: 'anthropic',
    modelId: 'claude-opus-4-5',
    displayName: 'Claude Opus 4.5',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: true,
      contextWindowTokens: 200_000,
      maxOutputTokens: 32_768,
      jsonMode: false,
    },
    pricing: {
      inputCostPer1M: 15.0,
      outputCostPer1M: 75.0,
      cacheReadCostPer1M: 1.5,
      cacheWriteCostPer1M: 18.75,
    },
  },

  {
    modelKey: toModelKey({ provider: 'anthropic', model: 'claude-sonnet-4-5' }),
    provider: 'anthropic',
    modelId: 'claude-sonnet-4-5',
    displayName: 'Claude Sonnet 4.5',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: true,
      contextWindowTokens: 200_000,
      maxOutputTokens: 16_384,
      jsonMode: false,
    },
    pricing: {
      inputCostPer1M: 3.0,
      outputCostPer1M: 15.0,
      cacheReadCostPer1M: 0.3,
      cacheWriteCostPer1M: 3.75,
    },
  },

  {
    modelKey: toModelKey({ provider: 'anthropic', model: 'claude-haiku-3-5' }),
    provider: 'anthropic',
    modelId: 'claude-haiku-3-5',
    displayName: 'Claude Haiku 3.5',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: true,
      contextWindowTokens: 200_000,
      maxOutputTokens: 8_192,
      jsonMode: false,
    },
    pricing: {
      inputCostPer1M: 0.8,
      outputCostPer1M: 4.0,
      cacheReadCostPer1M: 0.08,
      cacheWriteCostPer1M: 1.0,
    },
  },

  // ── OpenAI ────────────────────────────────────────────────────────────────

  {
    modelKey: toModelKey({ provider: 'openai', model: 'gpt-4o' }),
    provider: 'openai',
    modelId: 'gpt-4o',
    displayName: 'GPT-4o',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 128_000,
      maxOutputTokens: 16_384,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 2.5,
      outputCostPer1M: 10.0,
    },
  },

  {
    modelKey: toModelKey({ provider: 'openai', model: 'gpt-4o-mini' }),
    provider: 'openai',
    modelId: 'gpt-4o-mini',
    displayName: 'GPT-4o Mini',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 128_000,
      maxOutputTokens: 16_384,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 0.15,
      outputCostPer1M: 0.6,
    },
  },

  {
    modelKey: toModelKey({ provider: 'openai', model: 'o3' }),
    provider: 'openai',
    modelId: 'o3',
    displayName: 'OpenAI o3',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: false,
      promptCaching: false,
      contextWindowTokens: 200_000,
      maxOutputTokens: 100_000,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 10.0,
      outputCostPer1M: 40.0,
    },
  },

  // ── Gemini ────────────────────────────────────────────────────────────────

  {
    modelKey: toModelKey({ provider: 'gemini', model: 'gemini-2.5-pro' }),
    provider: 'gemini',
    modelId: 'gemini-2.5-pro',
    displayName: 'Gemini 2.5 Pro',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 1_000_000,
      maxOutputTokens: 65_536,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 1.25,
      outputCostPer1M: 5.0,
    },
  },

  {
    modelKey: toModelKey({ provider: 'gemini', model: 'gemini-2.5-flash' }),
    provider: 'gemini',
    modelId: 'gemini-2.5-flash',
    displayName: 'Gemini 2.5 Flash',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: true,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 1_000_000,
      maxOutputTokens: 65_536,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 0.075,
      outputCostPer1M: 0.3,
    },
  },

  // ── Mistral ───────────────────────────────────────────────────────────────

  {
    modelKey: toModelKey({ provider: 'mistral', model: 'mistral-large-latest' }),
    provider: 'mistral',
    modelId: 'mistral-large-latest',
    displayName: 'Mistral Large',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: false,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 131_072,
      maxOutputTokens: 16_384,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 2.0,
      outputCostPer1M: 6.0,
    },
  },

  // ── Groq ──────────────────────────────────────────────────────────────────

  {
    modelKey: toModelKey({ provider: 'groq', model: 'llama-3.3-70b-versatile' }),
    provider: 'groq',
    modelId: 'llama-3.3-70b-versatile',
    displayName: 'Llama 3.3 70B (Groq)',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: false,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 128_000,
      maxOutputTokens: 32_768,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 0.59,
      outputCostPer1M: 0.79,
    },
  },

  // ── Ollama (local — zero cost, dynamic models) ────────────────────────────
  // Ollama entries are "template" entries; the registry creates real entries
  // dynamically when a model is pulled. These serve as the known defaults.

  {
    modelKey: toModelKey({ provider: 'ollama', model: 'llama3.2' }),
    provider: 'ollama',
    modelId: 'llama3.2',
    displayName: 'Llama 3.2 (Ollama)',
    isDefault: true,
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: false,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 128_000,
      maxOutputTokens: 8_192,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 0,
      outputCostPer1M: 0,
    },
  },

  {
    modelKey: toModelKey({ provider: 'ollama', model: 'qwen2.5-coder:32b' }),
    provider: 'ollama',
    modelId: 'qwen2.5-coder:32b',
    displayName: 'Qwen 2.5 Coder 32B (Ollama)',
    capabilities: {
      toolUse: true,
      streaming: true,
      vision: false,
      systemPrompt: true,
      promptCaching: false,
      contextWindowTokens: 131_072,
      maxOutputTokens: 8_192,
      jsonMode: true,
    },
    pricing: {
      inputCostPer1M: 0,
      outputCostPer1M: 0,
    },
  },
];

// ── CatalogIndex ──────────────────────────────────────────────────────────────

class ModelCatalogIndex {
  private readonly byKey = new Map<ModelKey, ModelCatalogEntry>();
  private readonly byProvider = new Map<ProviderId, ModelCatalogEntry[]>();

  constructor(entries: readonly ModelCatalogEntry[]) {
    for (const entry of entries) {
      this.byKey.set(entry.modelKey, entry);
      const list = this.byProvider.get(entry.provider) ?? [];
      list.push(entry);
      this.byProvider.set(entry.provider, list);
    }
  }

  get(key: ModelKey): ModelCatalogEntry | undefined {
    return this.byKey.get(key);
  }

  getByProvider(provider: ProviderId): readonly ModelCatalogEntry[] {
    return this.byProvider.get(provider) ?? [];
  }

  getDefault(provider: ProviderId): ModelCatalogEntry | undefined {
    return this.getByProvider(provider).find((e) => e.isDefault);
  }

  /**
   * Register a dynamic entry (e.g. a model pulled into Ollama at runtime).
   * Does not mutate the static catalog — only the runtime index.
   */
  register(entry: ModelCatalogEntry): void {
    this.byKey.set(entry.modelKey, entry);
    const list = this.byProvider.get(entry.provider) ?? [];
    if (!list.some((e) => e.modelKey === entry.modelKey)) {
      list.push(entry);
      this.byProvider.set(entry.provider, list);
    }
  }

  all(): readonly ModelCatalogEntry[] {
    return [...this.byKey.values()];
  }
}

/** Singleton catalog index. Import and use directly. */
export const modelCatalog = new ModelCatalogIndex(CATALOG);

/**
 * Estimate cost for a given usage against a known model.
 * Returns 0 if pricing is not found.
 */
export function estimateCostUsd(
  modelKey: ModelKey,
  inputTokens: number,
  outputTokens: number,
  cacheReadTokens = 0,
  cacheWriteTokens = 0,
): number {
  const entry = modelCatalog.get(modelKey);
  if (!entry) return 0;

  const { pricing } = entry;
  const inputCost = (inputTokens / 1_000_000) * pricing.inputCostPer1M;
  const outputCost = (outputTokens / 1_000_000) * pricing.outputCostPer1M;
  const cacheReadCost = cacheReadTokens > 0 && pricing.cacheReadCostPer1M
    ? (cacheReadTokens / 1_000_000) * pricing.cacheReadCostPer1M
    : 0;
  const cacheWriteCost = cacheWriteTokens > 0 && pricing.cacheWriteCostPer1M
    ? (cacheWriteTokens / 1_000_000) * pricing.cacheWriteCostPer1M
    : 0;

  return inputCost + outputCost + cacheReadCost + cacheWriteCost;
}
packages/providers/src/cost.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/cost.ts
// Pass 17 Part 1 — Cost tracking per session / per provider
// ─────────────────────────────────────────────────────────────────────────────

import type { ModelKey, ProviderId, UsageStats } from './types.js';
import { estimateCostUsd } from './capabilities.js';

// ── Per-call record ───────────────────────────────────────────────────────────

export interface CostRecord {
  readonly id: string;
  readonly sessionId: string;
  readonly provider: ProviderId;
  readonly modelKey: ModelKey;
  readonly usage: UsageStats;
  readonly estimatedCostUsd: number;
  readonly recordedAt: number;
}

// ── Session cost tracker ──────────────────────────────────────────────────────

export interface SessionCostSummary {
  readonly sessionId: string;
  readonly totalInputTokens: number;
  readonly totalOutputTokens: number;
  readonly totalCacheReadTokens: number;
  readonly totalCacheWriteTokens: number;
  readonly totalEstimatedCostUsd: number;
  readonly callCount: number;
  readonly byProvider: Record<ProviderId, { calls: number; costUsd: number }>;
  readonly byModel: Record<ModelKey, { calls: number; costUsd: number }>;
}

export class SessionCostTracker {
  private records: CostRecord[] = [];
  private callCounter = 0;

  constructor(
    private readonly sessionId: string,
    private readonly costCapUsd: number,
    private readonly warnThreshold: number = 0.8,
    private readonly onCapExceeded?: (summary: SessionCostSummary) => void,
    private readonly onWarnThreshold?: (summary: SessionCostSummary) => void,
  ) {}

  /**
   * Record a completed API call and return the updated summary.
   * Throws if the cost cap has been exceeded.
   */
  record(
    provider: ProviderId,
    modelKey: ModelKey,
    usage: UsageStats,
  ): { record: CostRecord; summary: SessionCostSummary; overWarnThreshold: boolean } {
    const estimatedCostUsd = estimateCostUsd(
      modelKey,
      usage.inputTokens,
      usage.outputTokens,
      usage.cacheReadTokens ?? 0,
      usage.cacheWriteTokens ?? 0,
    );

    const record: CostRecord = {
      id: `${this.sessionId}-call-${++this.callCounter}`,
      sessionId: this.sessionId,
      provider,
      modelKey,
      usage,
      estimatedCostUsd,
      recordedAt: Date.now(),
    };

    this.records.push(record);
    const summary = this.summarise();

    const overCap = this.costCapUsd > 0 && summary.totalEstimatedCostUsd >= this.costCapUsd;
    const overWarn =
      this.costCapUsd > 0 &&
      summary.totalEstimatedCostUsd >= this.costCapUsd * this.warnThreshold;

    if (overCap) {
      this.onCapExceeded?.(summary);
      throw new Error(
        `Session cost cap exceeded: $${summary.totalEstimatedCostUsd.toFixed(4)} >= $${this.costCapUsd.toFixed(2)}`,
      );
    }

    if (overWarn) {
      this.onWarnThreshold?.(summary);
    }

    return { record, summary, overWarnThreshold: overWarn };
  }

  summarise(): SessionCostSummary {
    let totalInputTokens = 0;
    let totalOutputTokens = 0;
    let totalCacheReadTokens = 0;
    let totalCacheWriteTokens = 0;
    let totalEstimatedCostUsd = 0;
    const byProvider: Record<string, { calls: number; costUsd: number }> = {};
    const byModel: Record<string, { calls: number; costUsd: number }> = {};

    for (const r of this.records) {
      totalInputTokens += r.usage.inputTokens;
      totalOutputTokens += r.usage.outputTokens;
      totalCacheReadTokens += r.usage.cacheReadTokens ?? 0;
      totalCacheWriteTokens += r.usage.cacheWriteTokens ?? 0;
      totalEstimatedCostUsd += r.estimatedCostUsd;

      const p = byProvider[r.provider] ?? { calls: 0, costUsd: 0 };
      byProvider[r.provider] = { calls: p.calls + 1, costUsd: p.costUsd + r.estimatedCostUsd };

      const m = byModel[r.modelKey] ?? { calls: 0, costUsd: 0 };
      byModel[r.modelKey] = { calls: m.calls + 1, costUsd: m.costUsd + r.estimatedCostUsd };
    }

    return {
      sessionId: this.sessionId,
      totalInputTokens,
      totalOutputTokens,
      totalCacheReadTokens,
      totalCacheWriteTokens,
      totalEstimatedCostUsd,
      callCount: this.records.length,
      byProvider: byProvider as SessionCostSummary['byProvider'],
      byModel: byModel as SessionCostSummary['byModel'],
    };
  }

  remainingBudgetUsd(): number {
    if (this.costCapUsd <= 0) return Infinity;
    return Math.max(0, this.costCapUsd - this.summarise().totalEstimatedCostUsd);
  }

  reset(): void {
    this.records = [];
    this.callCounter = 0;
  }

  getRecords(): readonly CostRecord[] {
    return [...this.records];
  }
}

// ── Global daily cost tracker ─────────────────────────────────────────────────
// Lightweight in-memory daily aggregator. Persisted separately by packages
// that need durability (e.g. packages/security audit log or a future
// packages/providers SQLite store).

export class DailyCostTracker {
  private readonly dailyRecords = new Map<
    string, // YYYY-MM-DD
    { totalCostUsd: number; callCount: number }
  >();

  private readonly dailyCapUsd: number;

  constructor(dailyCapUsd = 0) {
    this.dailyCapUsd = dailyCapUsd;
  }

  private today(): string {
    return new Date().toISOString().slice(0, 10);
  }

  add(costUsd: number): void {
    const day = this.today();
    const existing = this.dailyRecords.get(day) ?? { totalCostUsd: 0, callCount: 0 };
    const updated = { totalCostUsd: existing.totalCostUsd + costUsd, callCount: existing.callCount + 1 };
    this.dailyRecords.set(day, updated);

    if (this.dailyCapUsd > 0 && updated.totalCostUsd >= this.dailyCapUsd) {
      throw new Error(
        `Daily cost cap exceeded: $${updated.totalCostUsd.toFixed(4)} >= $${this.dailyCapUsd.toFixed(2)} (${day})`,
      );
    }
  }

  todayCostUsd(): number {
    return this.dailyRecords.get(this.today())?.totalCostUsd ?? 0;
  }

  allDays(): Array<{ date: string; totalCostUsd: number; callCount: number }> {
    return [...this.dailyRecords.entries()].map(([date, data]) => ({ date, ...data }));
  }
}
packages/providers/src/health.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/health.ts
// Pass 17 Part 1 — Provider health checker
// ─────────────────────────────────────────────────────────────────────────────

import type { ProviderId, ProviderHealth, ProviderHealthStatus } from './types.js';

export interface HealthCheckOptions {
  /** Timeout for the health check request in ms */
  timeoutMs?: number;
  /** Number of times to retry a failed check before marking unhealthy */
  retries?: number;
}

export type HealthCheckFn = (
  providerId: ProviderId,
  opts: HealthCheckOptions,
) => Promise<ProviderHealth>;

/**
 * ProviderHealthMonitor tracks the last known health of each registered
 * provider and coordinates periodic re-checks.
 *
 * Each adapter registers its own healthCheckFn during registration.
 * The monitor calls them on a configurable interval and caches results.
 */
export class ProviderHealthMonitor {
  private readonly healthMap = new Map<ProviderId, ProviderHealth>();
  private readonly checkFns = new Map<ProviderId, HealthCheckFn>();
  private intervalHandle: ReturnType<typeof setInterval> | null = null;

  constructor(
    private readonly checkIntervalMs: number = 60_000,
    private readonly opts: HealthCheckOptions = { timeoutMs: 5_000, retries: 1 },
  ) {}

  /**
   * Register a provider with its health check function.
   * Initial status is 'unknown' until the first check completes.
   */
  register(providerId: ProviderId, checkFn: HealthCheckFn): void {
    this.checkFns.set(providerId, checkFn);
    if (!this.healthMap.has(providerId)) {
      this.healthMap.set(providerId, {
        providerId,
        status: 'unknown',
        lastCheckedAt: 0,
      });
    }
  }

  unregister(providerId: ProviderId): void {
    this.checkFns.delete(providerId);
    this.healthMap.delete(providerId);
  }

  getHealth(providerId: ProviderId): ProviderHealth {
    return (
      this.healthMap.get(providerId) ?? {
        providerId,
        status: 'unknown',
        lastCheckedAt: 0,
      }
    );
  }

  getAllHealth(): readonly ProviderHealth[] {
    return [...this.healthMap.values()];
  }

  isHealthy(providerId: ProviderId): boolean {
    const h = this.getHealth(providerId);
    return h.status === 'healthy' || h.status === 'degraded';
  }

  async checkNow(providerId: ProviderId): Promise<ProviderHealth> {
    const fn = this.checkFns.get(providerId);
    if (!fn) {
      const health: ProviderHealth = {
        providerId,
        status: 'unknown',
        lastCheckedAt: Date.now(),
        error: 'No health check function registered.',
      };
      this.healthMap.set(providerId, health);
      return health;
    }

    try {
      const health = await fn(providerId, this.opts);
      this.healthMap.set(providerId, health);
      return health;
    } catch (err) {
      const health: ProviderHealth = {
        providerId,
        status: 'unhealthy',
        lastCheckedAt: Date.now(),
        error: err instanceof Error ? err.message : String(err),
      };
      this.healthMap.set(providerId, health);
      return health;
    }
  }

  async checkAll(): Promise<readonly ProviderHealth[]> {
    const checks = [...this.checkFns.keys()].map((id) => this.checkNow(id));
    return Promise.all(checks);
  }

  /** Override health manually (e.g. after observing an error in a real request) */
  reportObservedHealth(
    providerId: ProviderId,
    status: ProviderHealthStatus,
    error?: string,
    latencyMs?: number,
  ): void {
    this.healthMap.set(providerId, {
      providerId,
      status,
      latencyMs,
      lastCheckedAt: Date.now(),
      error,
    });
  }

  startPeriodicChecks(): void {
    if (this.intervalHandle) return;
    this.intervalHandle = setInterval(() => {
      void this.checkAll();
    }, this.checkIntervalMs);
    // Don't block process exit
    if (typeof this.intervalHandle === 'object' && 'unref' in this.intervalHandle) {
      (this.intervalHandle as NodeJS.Timeout).unref();
    }
  }

  stopPeriodicChecks(): void {
    if (this.intervalHandle) {
      clearInterval(this.intervalHandle);
      this.intervalHandle = null;
    }
  }
}
packages/providers/src/adapters/base.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/base.ts
// Pass 17 Part 1 — BaseProviderAdapter abstract class
// ─────────────────────────────────────────────────────────────────────────────

import type {
  ProviderConfig,
  ProviderHealth,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderId,
  HealthCheckOptions,
} from '../types.js';

/**
 * The contract every provider adapter must fulfill.
 * Core's ProviderRouter delegates to adapters via this interface.
 */
export interface IProviderAdapter {
  readonly providerId: ProviderId;
  readonly config: ProviderConfig;

  /**
   * Complete a single turn. Returns the full response after the model finishes.
   */
  complete(options: ProviderRequestOptions): Promise<ProviderResponse>;

  /**
   * Stream a single turn. Yields StreamEvents as they arrive.
   */
  stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown>;

  /**
   * Perform a lightweight health check (e.g. list models, ping endpoint).
   */
  healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth>;

  /**
   * List models available at this provider (used to populate the dynamic
   * catalog for local providers like Ollama).
   */
  listModels(): Promise<readonly string[]>;

  /**
   * Update runtime configuration (e.g. new API key, different base URL).
   */
  updateConfig(patch: Partial<ProviderConfig>): void;
}

/**
 * BaseProviderAdapter provides shared scaffolding.
 * Concrete adapters extend this class and implement the abstract methods.
 */
export abstract class BaseProviderAdapter implements IProviderAdapter {
  protected _config: ProviderConfig;

  constructor(config: ProviderConfig) {
    this._config = config;
  }

  get providerId(): ProviderId {
    return this._config.id;
  }

  get config(): ProviderConfig {
    return this._config;
  }

  updateConfig(patch: Partial<ProviderConfig>): void {
    this._config = { ...this._config, ...patch };
  }

  abstract complete(options: ProviderRequestOptions): Promise<ProviderResponse>;
  abstract stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown>;
  abstract healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth>;
  abstract listModels(): Promise<readonly string[]>;

  // ── Shared helpers ──────────────────────────────────────────────────────────

  protected assertEnabled(): void {
    if (!this._config.enabled) {
      throw new Error(`Provider "${this._config.id}" is disabled.`);
    }
  }

  protected assertApiKey(): void {
    if (!this._config.apiKey) {
      throw new Error(
        `Provider "${this._config.id}" requires an API key (${this._config.id.toUpperCase()}_API_KEY).`,
      );
    }
  }

  protected buildHeaders(extra: Record<string, string> = {}): Record<string, string> {
    return {
      'Content-Type': 'application/json',
      'User-Agent': 'LocoWorker/0.1.0 (+https://github.com/knarayanareddy/LocoworkerV1.0)',
      ...extra,
    };
  }

  protected async fetchWithTimeout(
    url: string,
    init: RequestInit,
    timeoutMs: number,
  ): Promise<Response> {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);
    try {
      const res = await fetch(url, { ...init, signal: controller.signal });
      return res;
    } finally {
      clearTimeout(timer);
    }
  }

  protected measureLatency<T>(fn: () => Promise<T>): Promise<{ result: T; latencyMs: number }> {
    const start = performance.now();
    return fn().then((result) => ({ result, latencyMs: Math.round(performance.now() - start) }));
  }
}
packages/providers/src/adapters/anthropic.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/anthropic.ts
// Pass 17 Part 1 — AnthropicAdapter
//
// Implements the Anthropic Messages API (claude-* models).
// Pass 18 may replace this with the official @anthropic-ai/sdk if desired;
// for now we use raw fetch to keep the package dependency-free.
// ─────────────────────────────────────────────────────────────────────────────

import { BaseProviderAdapter } from './base.js';
import type {
  ProviderConfig,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderHealth,
  UsageStats,
  ContentBlock,
  HealthCheckOptions,
} from '../types.js';
import { ProviderError } from '../types.js';

const ANTHROPIC_API_BASE = 'https://api.anthropic.com/v1';
const ANTHROPIC_VERSION = '2023-06-01';

export class AnthropicAdapter extends BaseProviderAdapter {
  constructor(config: ProviderConfig) {
    super(config);
  }

  private get baseUrl(): string {
    return this._config.baseUrl ?? ANTHROPIC_API_BASE;
  }

  private buildAnthropicHeaders(): Record<string, string> {
    return this.buildHeaders({
      'x-api-key': this._config.apiKey ?? '',
      'anthropic-version': ANTHROPIC_VERSION,
      'anthropic-beta': 'prompt-caching-2024-07-31',
    });
  }

  private buildAnthropicBody(options: ProviderRequestOptions): Record<string, unknown> {
    return {
      model: options.model,
      max_tokens: options.maxTokens ?? 8096,
      messages: options.messages.map((m) => ({
        role: m.role === 'tool' ? 'user' : m.role,
        content: m.content,
      })),
      ...(options.system ? { system: options.system } : {}),
      ...(options.tools?.length
        ? {
            tools: options.tools.map((t) => ({
              name: t.name,
              description: t.description,
              input_schema: t.inputSchema,
            })),
          }
        : {}),
      ...(options.temperature !== undefined ? { temperature: options.temperature } : {}),
      ...(options.topP !== undefined ? { top_p: options.topP } : {}),
      ...(options.stopSequences?.length ? { stop_sequences: options.stopSequences } : {}),
      ...(options.extraParams ?? {}),
    };
  }

  async complete(options: ProviderRequestOptions): Promise<ProviderResponse> {
    this.assertEnabled();
    this.assertApiKey();

    const body = { ...this.buildAnthropicBody(options), stream: false };
    const { result: raw, latencyMs } = await this.measureLatency(() =>
      this.fetchWithTimeout(
        `${this.baseUrl}/messages`,
        {
          method: 'POST',
          headers: this.buildAnthropicHeaders(),
          body: JSON.stringify(body),
        },
        this._config.timeoutMs,
      ).then(async (res) => {
        if (!res.ok) {
          const text = await res.text().catch(() => '');
          throw new ProviderError(
            this.mapStatusToCode(res.status),
            `Anthropic API error ${res.status}: ${text}`,
            'anthropic',
            this.isRetryable(res.status),
            undefined,
            res.status,
          );
        }
        return res.json() as Promise<Record<string, unknown>>;
      }),
    );

    return this.parseResponse(raw, latencyMs);
  }

  async *stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown> {
    this.assertEnabled();
    this.assertApiKey();

    const body = { ...this.buildAnthropicBody(options), stream: true };
    const res = await this.fetchWithTimeout(
      `${this.baseUrl}/messages`,
      {
        method: 'POST',
        headers: this.buildAnthropicHeaders(),
        body: JSON.stringify(body),
      },
      this._config.timeoutMs,
    );

    if (!res.ok || !res.body) {
      const text = await res.text().catch(() => '');
      throw new ProviderError(
        this.mapStatusToCode(res.status),
        `Anthropic stream error ${res.status}: ${text}`,
        'anthropic',
        this.isRetryable(res.status),
        undefined,
        res.status,
      );
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() ?? '';

        for (const line of lines) {
          const trimmed = line.trim();
          if (!trimmed || trimmed === 'data: [DONE]') continue;
          if (!trimmed.startsWith('data: ')) continue;

          try {
            const event = JSON.parse(trimmed.slice(6)) as Record<string, unknown>;
            const mapped = this.mapStreamEvent(event);
            if (mapped) yield mapped;
          } catch {
            // Malformed SSE line — skip
          }
        }
      }
    } finally {
      reader.releaseLock();
    }
  }

  async healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth> {
    const start = performance.now();
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models`,
        { method: 'GET', headers: this.buildAnthropicHeaders() },
        opts.timeoutMs ?? 5_000,
      );
      const latencyMs = Math.round(performance.now() - start);
      if (res.ok) {
        return { providerId: 'anthropic', status: 'healthy', latencyMs, lastCheckedAt: Date.now() };
      }
      return {
        providerId: 'anthropic',
        status: res.status === 429 ? 'degraded' : 'unhealthy',
        latencyMs,
        lastCheckedAt: Date.now(),
        error: `HTTP ${res.status}`,
      };
    } catch (err) {
      return {
        providerId: 'anthropic',
        status: 'unhealthy',
        lastCheckedAt: Date.now(),
        error: err instanceof Error ? err.message : String(err),
      };
    }
  }

  async listModels(): Promise<readonly string[]> {
    this.assertApiKey();
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models`,
        { method: 'GET', headers: this.buildAnthropicHeaders() },
        this._config.timeoutMs,
      );
      if (!res.ok) return [];
      const data = await res.json() as { data?: Array<{ id: string }> };
      return data.data?.map((m) => m.id) ?? [];
    } catch {
      return [];
    }
  }

  // ── Private helpers ─────────────────────────────────────────────────────────

  private parseResponse(raw: Record<string, unknown>, latencyMs: number): ProviderResponse {
    const usage = raw['usage'] as { input_tokens: number; output_tokens: number; cache_read_input_tokens?: number; cache_creation_input_tokens?: number } | undefined;
    const usageStats: UsageStats = {
      inputTokens: usage?.input_tokens ?? 0,
      outputTokens: usage?.output_tokens ?? 0,
      cacheReadTokens: usage?.cache_read_input_tokens ?? 0,
      cacheWriteTokens: usage?.cache_creation_input_tokens ?? 0,
    };

    return {
      id: raw['id'] as string ?? '',
      provider: 'anthropic',
      model: raw['model'] as string ?? '',
      content: (raw['content'] as ContentBlock[]) ?? [],
      stopReason: this.mapStopReason(raw['stop_reason'] as string),
      usage: usageStats,
      latencyMs,
      rawResponse: raw,
    };
  }

  private mapStopReason(raw?: string): ProviderResponse['stopReason'] {
    switch (raw) {
      case 'end_turn': return 'end_turn';
      case 'tool_use': return 'tool_use';
      case 'max_tokens': return 'max_tokens';
      case 'stop_sequence': return 'stop_sequence';
      default: return raw ?? 'end_turn';
    }
  }

  private mapStatusToCode(status: number): import('../types.js').ProviderErrorCode {
    if (status === 401 || status === 403) return 'authentication_failed';
    if (status === 429) return 'rate_limited';
    if (status === 400) return 'invalid_request';
    if (status === 404) return 'model_not_found';
    if (status >= 500) return 'provider_unavailable';
    return 'unknown';
  }

  private isRetryable(status: number): boolean {
    return status === 429 || status >= 500;
  }

  private mapStreamEvent(event: Record<string, unknown>): StreamEvent | null {
    const type = event['type'] as string;
    switch (type) {
      case 'message_start': {
        const msg = event['message'] as Record<string, unknown>;
        return {
          type: 'message_start',
          message: { id: msg['id'] as string, provider: 'anthropic', model: msg['model'] as string },
        };
      }
      case 'content_block_start':
        return {
          type: 'content_block_start',
          index: event['index'] as number,
          contentBlock: event['content_block'] as ContentBlock,
        };
      case 'content_block_delta': {
        const delta = event['delta'] as Record<string, unknown>;
        const deltaType = delta['type'] as string;
        return {
          type: 'content_block_delta',
          index: event['index'] as number,
          delta:
            deltaType === 'text_delta'
              ? { type: 'text', text: delta['text'] as string }
              : { type: 'input_json_delta', partialJson: delta['partial_json'] as string },
        };
      }
      case 'content_block_stop':
        return { type: 'content_block_stop', index: event['index'] as number };
      case 'message_delta': {
        const delta = event['delta'] as Record<string, unknown>;
        const usage = event['usage'] as { output_tokens: number } | undefined;
        return {
          type: 'message_delta',
          stopReason: this.mapStopReason(delta['stop_reason'] as string),
          usage: { inputTokens: 0, outputTokens: usage?.output_tokens ?? 0 },
        };
      }
      case 'message_stop':
        return { type: 'message_stop' };
      default:
        return null;
    }
  }
}
packages/providers/src/adapters/openai.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/openai.ts
// Pass 17 Part 1 — OpenAICompatibleAdapter
//
// Covers: OpenAI, Groq, Mistral, Cohere (OpenAI-compatible endpoint),
// and any custom OpenAI-compatible endpoint.
// The providerId is set at construction time so the same adapter class
// can be instantiated for different providers.
// ─────────────────────────────────────────────────────────────────────────────

import { BaseProviderAdapter } from './base.js';
import type {
  ProviderConfig,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderHealth,
  UsageStats,
  HealthCheckOptions,
  Message,
} from '../types.js';
import { ProviderError } from '../types.js';

const DEFAULT_BASE_URLS: Record<string, string> = {
  openai: 'https://api.openai.com/v1',
  groq: 'https://api.groq.com/openai/v1',
  mistral: 'https://api.mistral.ai/v1',
  cohere: 'https://api.cohere.com/compatibility/v1',
};

export class OpenAICompatibleAdapter extends BaseProviderAdapter {
  constructor(config: ProviderConfig) {
    super(config);
  }

  private get baseUrl(): string {
    return this._config.baseUrl ?? DEFAULT_BASE_URLS[this._config.id] ?? 'https://api.openai.com/v1';
  }

  private buildOAIHeaders(): Record<string, string> {
    return this.buildHeaders({
      Authorization: `Bearer ${this._config.apiKey ?? ''}`,
    });
  }

  private convertMessages(messages: readonly Message[]): unknown[] {
    return messages.map((m) => ({
      role: m.role,
      content: typeof m.content === 'string'
        ? m.content
        : m.content
            .map((block) => {
              if (block.type === 'text') return { type: 'text', text: block.text };
              if (block.type === 'image') {
                if (block.source.type === 'url') {
                  return { type: 'image_url', image_url: { url: block.source.url } };
                }
                return {
                  type: 'image_url',
                  image_url: { url: `data:${block.source.mediaType};base64,${block.source.data}` },
                };
              }
              return null;
            })
            .filter(Boolean),
    }));
  }

  private buildOAIBody(options: ProviderRequestOptions): Record<string, unknown> {
    const msgs = options.system
      ? [{ role: 'system', content: options.system }, ...this.convertMessages(options.messages)]
      : this.convertMessages(options.messages);

    return {
      model: options.model,
      messages: msgs,
      max_tokens: options.maxTokens ?? 4096,
      ...(options.temperature !== undefined ? { temperature: options.temperature } : {}),
      ...(options.topP !== undefined ? { top_p: options.topP } : {}),
      ...(options.stopSequences?.length ? { stop: options.stopSequences } : {}),
      ...(options.tools?.length
        ? {
            tools: options.tools.map((t) => ({
              type: 'function',
              function: { name: t.name, description: t.description, parameters: t.inputSchema },
            })),
            tool_choice: 'auto',
          }
        : {}),
      ...(options.extraParams ?? {}),
    };
  }

  async complete(options: ProviderRequestOptions): Promise<ProviderResponse> {
    this.assertEnabled();
    this.assertApiKey();

    const { result: raw, latencyMs } = await this.measureLatency(() =>
      this.fetchWithTimeout(
        `${this.baseUrl}/chat/completions`,
        {
          method: 'POST',
          headers: this.buildOAIHeaders(),
          body: JSON.stringify(this.buildOAIBody(options)),
        },
        this._config.timeoutMs,
      ).then(async (res) => {
        if (!res.ok) {
          const text = await res.text().catch(() => '');
          throw new ProviderError(
            this.mapStatusToCode(res.status),
            `${this._config.id} API error ${res.status}: ${text}`,
            this._config.id,
            this.isRetryable(res.status),
            undefined,
            res.status,
          );
        }
        return res.json() as Promise<Record<string, unknown>>;
      }),
    );

    return this.parseOAIResponse(raw, latencyMs);
  }

  async *stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown> {
    this.assertEnabled();
    this.assertApiKey();

    const body = { ...this.buildOAIBody(options), stream: true };
    const res = await this.fetchWithTimeout(
      `${this.baseUrl}/chat/completions`,
      { method: 'POST', headers: this.buildOAIHeaders(), body: JSON.stringify(body) },
      this._config.timeoutMs,
    );

    if (!res.ok || !res.body) {
      const text = await res.text().catch(() => '');
      throw new ProviderError(
        this.mapStatusToCode(res.status),
        `${this._config.id} stream error ${res.status}: ${text}`,
        this._config.id,
        this.isRetryable(res.status),
        undefined,
        res.status,
      );
    }

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';
    let messageId = '';
    let model = options.model;

    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() ?? '';

        for (const line of lines) {
          const trimmed = line.trim();
          if (!trimmed || trimmed === 'data: [DONE]') {
            if (trimmed === 'data: [DONE]') yield { type: 'message_stop' };
            continue;
          }
          if (!trimmed.startsWith('data: ')) continue;

          try {
            const chunk = JSON.parse(trimmed.slice(6)) as Record<string, unknown>;
            if (!messageId && chunk['id']) {
              messageId = chunk['id'] as string;
              model = (chunk['model'] as string) ?? model;
              yield { type: 'message_start', message: { id: messageId, provider: this._config.id, model } };
            }

            const choices = chunk['choices'] as Array<Record<string, unknown>> | undefined;
            if (!choices?.length) continue;

            const choice = choices[0];
            const delta = choice?.['delta'] as Record<string, unknown> | undefined;
            if (!delta) continue;

            if (delta['content'] !== undefined && delta['content'] !== null) {
              yield {
                type: 'content_block_delta',
                index: 0,
                delta: { type: 'text', text: delta['content'] as string },
              };
            }

            const finishReason = choice?.['finish_reason'] as string | null;
            if (finishReason) {
              yield {
                type: 'message_delta',
                stopReason: this.mapFinishReason(finishReason),
                usage: { inputTokens: 0, outputTokens: 0 },
              };
            }
          } catch {
            // Skip malformed lines
          }
        }
      }
    } finally {
      reader.releaseLock();
    }
  }

  async healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth> {
    const start = performance.now();
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models`,
        { method: 'GET', headers: this.buildOAIHeaders() },
        opts.timeoutMs ?? 5_000,
      );
      const latencyMs = Math.round(performance.now() - start);
      return {
        providerId: this._config.id,
        status: res.ok ? 'healthy' : res.status === 429 ? 'degraded' : 'unhealthy',
        latencyMs,
        lastCheckedAt: Date.now(),
        error: res.ok ? undefined : `HTTP ${res.status}`,
      };
    } catch (err) {
      return {
        providerId: this._config.id,
        status: 'unhealthy',
        lastCheckedAt: Date.now(),
        error: err instanceof Error ? err.message : String(err),
      };
    }
  }

  async listModels(): Promise<readonly string[]> {
    if (!this._config.apiKey) return [];
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models`,
        { method: 'GET', headers: this.buildOAIHeaders() },
        this._config.timeoutMs,
      );
      if (!res.ok) return [];
      const data = await res.json() as { data?: Array<{ id: string }> };
      return data.data?.map((m) => m.id) ?? [];
    } catch {
      return [];
    }
  }

  // ── Private helpers ─────────────────────────────────────────────────────────

  private parseOAIResponse(raw: Record<string, unknown>, latencyMs: number): ProviderResponse {
    const choices = raw['choices'] as Array<Record<string, unknown>> | undefined;
    const choice = choices?.[0];
    const message = choice?.['message'] as Record<string, unknown> | undefined;
    const usage = raw['usage'] as { prompt_tokens: number; completion_tokens: number } | undefined;

    const usageStats: UsageStats = {
      inputTokens: usage?.prompt_tokens ?? 0,
      outputTokens: usage?.completion_tokens ?? 0,
    };

    const content = typeof message?.['content'] === 'string'
      ? [{ type: 'text' as const, text: message['content'] as string }]
      : [];

    return {
      id: raw['id'] as string ?? '',
      provider: this._config.id,
      model: raw['model'] as string ?? '',
      content,
      stopReason: this.mapFinishReason(choice?.['finish_reason'] as string ?? ''),
      usage: usageStats,
      latencyMs,
      rawResponse: raw,
    };
  }

  private mapFinishReason(raw: string): ProviderResponse['stopReason'] {
    switch (raw) {
      case 'stop': return 'end_turn';
      case 'tool_calls': return 'tool_use';
      case 'length': return 'max_tokens';
      default: return raw || 'end_turn';
    }
  }

  private mapStatusToCode(status: number): import('../types.js').ProviderErrorCode {
    if (status === 401 || status === 403) return 'authentication_failed';
    if (status === 429) return 'rate_limited';
    if (status === 400) return 'invalid_request';
    if (status === 404) return 'model_not_found';
    if (status >= 500) return 'provider_unavailable';
    return 'unknown';
  }

  private isRetryable(status: number): boolean {
    return status === 429 || status >= 500;
  }
}
packages/providers/src/adapters/gemini.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/gemini.ts
// Pass 17 Part 1 — GeminiAdapter (Google Gemini API)
// ─────────────────────────────────────────────────────────────────────────────

import { BaseProviderAdapter } from './base.js';
import type {
  ProviderConfig,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderHealth,
  UsageStats,
  HealthCheckOptions,
} from '../types.js';
import { ProviderError } from '../types.js';

const GEMINI_API_BASE = 'https://generativelanguage.googleapis.com/v1beta';

export class GeminiAdapter extends BaseProviderAdapter {
  constructor(config: ProviderConfig) {
    super(config);
  }

  private get baseUrl(): string {
    return this._config.baseUrl ?? GEMINI_API_BASE;
  }

  private modelEndpoint(model: string, streaming: boolean): string {
    const method = streaming ? 'streamGenerateContent' : 'generateContent';
    return `${this.baseUrl}/models/${model}:${method}?key=${this._config.apiKey ?? ''}`;
  }

  private buildGeminiBody(options: ProviderRequestOptions): Record<string, unknown> {
    const contents = options.messages
      .filter((m) => m.role !== 'system')
      .map((m) => ({
        role: m.role === 'assistant' ? 'model' : 'user',
        parts: typeof m.content === 'string'
          ? [{ text: m.content }]
          : m.content.map((b) => (b.type === 'text' ? { text: b.text } : null)).filter(Boolean),
      }));

    return {
      contents,
      ...(options.system
        ? { systemInstruction: { parts: [{ text: options.system }] } }
        : {}),
      generationConfig: {
        maxOutputTokens: options.maxTokens ?? 4096,
        ...(options.temperature !== undefined ? { temperature: options.temperature } : {}),
        ...(options.topP !== undefined ? { topP: options.topP } : {}),
        ...(options.topK !== undefined ? { topK: options.topK } : {}),
        ...(options.stopSequences?.length ? { stopSequences: options.stopSequences } : {}),
      },
      ...(options.tools?.length
        ? {
            tools: [
              {
                functionDeclarations: options.tools.map((t) => ({
                  name: t.name,
                  description: t.description,
                  parameters: t.inputSchema,
                })),
              },
            ],
          }
        : {}),
    };
  }

  async complete(options: ProviderRequestOptions): Promise<ProviderResponse> {
    this.assertEnabled();
    this.assertApiKey();

    const { result: raw, latencyMs } = await this.measureLatency(() =>
      this.fetchWithTimeout(
        this.modelEndpoint(options.model, false),
        {
          method: 'POST',
          headers: this.buildHeaders(),
          body: JSON.stringify(this.buildGeminiBody(options)),
        },
        this._config.timeoutMs,
      ).then(async (res) => {
        if (!res.ok) {
          const text = await res.text().catch(() => '');
          throw new ProviderError(
            res.status === 429 ? 'rate_limited' : res.status === 401 ? 'authentication_failed' : 'unknown',
            `Gemini API error ${res.status}: ${text}`,
            'gemini',
            res.status === 429 || res.status >= 500,
            undefined,
            res.status,
          );
        }
        return res.json() as Promise<Record<string, unknown>>;
      }),
    );

    return this.parseGeminiResponse(raw, options.model, latencyMs);
  }

  async *stream(_options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown> {
    // Gemini streaming stub — full implementation in Pass 18
    throw new ProviderError(
      'unknown',
      'Gemini streaming not yet implemented. Use complete() instead.',
      'gemini',
      false,
    );
  }

  async healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth> {
    const start = performance.now();
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models?key=${this._config.apiKey ?? ''}`,
        { method: 'GET', headers: this.buildHeaders() },
        opts.timeoutMs ?? 5_000,
      );
      const latencyMs = Math.round(performance.now() - start);
      return {
        providerId: 'gemini',
        status: res.ok ? 'healthy' : 'unhealthy',
        latencyMs,
        lastCheckedAt: Date.now(),
        error: res.ok ? undefined : `HTTP ${res.status}`,
      };
    } catch (err) {
      return {
        providerId: 'gemini',
        status: 'unhealthy',
        lastCheckedAt: Date.now(),
        error: err instanceof Error ? err.message : String(err),
      };
    }
  }

  async listModels(): Promise<readonly string[]> {
    if (!this._config.apiKey) return [];
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/models?key=${this._config.apiKey}`,
        { method: 'GET', headers: this.buildHeaders() },
        this._config.timeoutMs,
      );
      if (!res.ok) return [];
      const data = await res.json() as { models?: Array<{ name: string }> };
      return data.models?.map((m) => m.name.replace('models/', '')) ?? [];
    } catch {
      return [];
    }
  }

  private parseGeminiResponse(raw: Record<string, unknown>, model: string, latencyMs: number): ProviderResponse {
    const candidates = raw['candidates'] as Array<Record<string, unknown>> | undefined;
    const candidate = candidates?.[0];
    const content = candidate?.['content'] as { parts?: Array<{ text?: string }> } | undefined;
    const text = content?.parts?.map((p) => p.text ?? '').join('') ?? '';

    const usageMeta = raw['usageMetadata'] as {
      promptTokenCount?: number;
      candidatesTokenCount?: number;
    } | undefined;

    const usage: UsageStats = {
      inputTokens: usageMeta?.promptTokenCount ?? 0,
      outputTokens: usageMeta?.candidatesTokenCount ?? 0,
    };

    const finishReason = candidate?.['finishReason'] as string | undefined;

    return {
      id: `gemini-${Date.now()}`,
      provider: 'gemini',
      model,
      content: text ? [{ type: 'text', text }] : [],
      stopReason: finishReason === 'STOP' ? 'end_turn' : finishReason === 'MAX_TOKENS' ? 'max_tokens' : 'end_turn',
      usage,
      latencyMs,
      rawResponse: raw,
    };
  }
}
packages/providers/src/adapters/ollama.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/ollama.ts
// Pass 17 Part 1 — OllamaAdapter (local Ollama server)
//
// Uses Ollama's native /api/chat endpoint (OpenAI-compatible mode is also
// available but we use native to access Ollama-specific features like
// /api/tags for model listing).
// ─────────────────────────────────────────────────────────────────────────────

import { BaseProviderAdapter } from './base.js';
import type {
  ProviderConfig,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderHealth,
  UsageStats,
  HealthCheckOptions,
} from '../types.js';
import { ProviderError } from '../types.js';

export class OllamaAdapter extends BaseProviderAdapter {
  constructor(config: ProviderConfig) {
    super(config);
  }

  private get baseUrl(): string {
    return this._config.baseUrl ?? 'http://localhost:11434';
  }

  private buildOllamaBody(options: ProviderRequestOptions, stream: boolean): Record<string, unknown> {
    const messages = options.system
      ? [{ role: 'system', content: options.system }, ...options.messages]
      : [...options.messages];

    return {
      model: options.model,
      messages: messages.map((m) => ({
        role: m.role,
        content: typeof m.content === 'string'
          ? m.content
          : m.content.map((b) => (b.type === 'text' ? b.text : '')).join(''),
      })),
      stream,
      options: {
        ...(options.temperature !== undefined ? { temperature: options.temperature } : {}),
        ...(options.topP !== undefined ? { top_p: options.topP } : {}),
        ...(options.topK !== undefined ? { top_k: options.topK } : {}),
        num_predict: options.maxTokens ?? -1,
        stop: options.stopSequences ?? [],
      },
      ...(options.tools?.length
        ? {
            tools: options.tools.map((t) => ({
              type: 'function',
              function: { name: t.name, description: t.description, parameters: t.inputSchema },
            })),
          }
        : {}),
    };
  }

  async complete(options: ProviderRequestOptions): Promise<ProviderResponse> {
    this.assertEnabled();

    const { result: raw, latencyMs } = await this.measureLatency(() =>
      this.fetchWithTimeout(
        `${this.baseUrl}/api/chat`,
        {
          method: 'POST',
          headers: this.buildHeaders(),
          body: JSON.stringify(this.buildOllamaBody(options, false)),
        },
        this._config.timeoutMs,
      ).then(async (res) => {
        if (!res.ok) {
          const text = await res.text().catch(() => '');
          throw new ProviderError(
            'provider_unavailable',
            `Ollama error ${res.status}: ${text}`,
            'ollama',
            true,
            undefined,
            res.status,
          );
        }
        return res.json() as Promise<Record<string, unknown>>;
      }),
    );

    return this.parseOllamaResponse(raw, options.model, latencyMs);
  }

  async *stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown> {
    this.assertEnabled();

    const res = await this.fetchWithTimeout(
      `${this.baseUrl}/api/chat`,
      {
        method: 'POST',
        headers: this.buildHeaders(),
        body: JSON.stringify(this.buildOllamaBody(options, true)),
      },
      this._config.timeoutMs,
    );

    if (!res.ok || !res.body) {
      throw new ProviderError('provider_unavailable', `Ollama stream error ${res.status}`, 'ollama', true);
    }

    yield { type: 'message_start', message: { id: `ollama-${Date.now()}`, provider: 'ollama', model: options.model } };

    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() ?? '';

        for (const line of lines) {
          if (!line.trim()) continue;
          try {
            const chunk = JSON.parse(line) as Record<string, unknown>;
            const msg = chunk['message'] as { content?: string } | undefined;
            if (msg?.content) {
              yield { type: 'content_block_delta', index: 0, delta: { type: 'text', text: msg.content } };
            }
            if (chunk['done'] === true) {
              yield {
                type: 'message_delta',
                stopReason: 'end_turn',
                usage: {
                  inputTokens: (chunk['prompt_eval_count'] as number) ?? 0,
                  outputTokens: (chunk['eval_count'] as number) ?? 0,
                },
              };
              yield { type: 'message_stop' };
            }
          } catch {
            // Skip malformed lines
          }
        }
      }
    } finally {
      reader.releaseLock();
    }
  }

  async healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth> {
    const start = performance.now();
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/api/tags`,
        { method: 'GET', headers: this.buildHeaders() },
        opts.timeoutMs ?? 3_000,
      );
      const latencyMs = Math.round(performance.now() - start);
      return {
        providerId: 'ollama',
        status: res.ok ? 'healthy' : 'unhealthy',
        latencyMs,
        lastCheckedAt: Date.now(),
        error: res.ok ? undefined : `HTTP ${res.status}`,
      };
    } catch (err) {
      return {
        providerId: 'ollama',
        status: 'unhealthy',
        lastCheckedAt: Date.now(),
        error: err instanceof Error ? err.message : String(err),
      };
    }
  }

  async listModels(): Promise<readonly string[]> {
    try {
      const res = await this.fetchWithTimeout(
        `${this.baseUrl}/api/tags`,
        { method: 'GET', headers: this.buildHeaders() },
        this._config.timeoutMs,
      );
      if (!res.ok) return [];
      const data = await res.json() as { models?: Array<{ name: string }> };
      return data.models?.map((m) => m.name) ?? [];
    } catch {
      return [];
    }
  }

  private parseOllamaResponse(raw: Record<string, unknown>, model: string, latencyMs: number): ProviderResponse {
    const message = raw['message'] as { content?: string } | undefined;
    const usage: UsageStats = {
      inputTokens: (raw['prompt_eval_count'] as number) ?? 0,
      outputTokens: (raw['eval_count'] as number) ?? 0,
    };

    return {
      id: `ollama-${Date.now()}`,
      provider: 'ollama',
      model,
      content: message?.content ? [{ type: 'text', text: message.content }] : [],
      stopReason: raw['done_reason'] === 'stop' ? 'end_turn' : 'end_turn',
      usage,
      latencyMs,
      rawResponse: raw,
    };
  }
}
packages/providers/src/adapters/local.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/local.ts
// Pass 17 Part 1 — LocalAdapter (LM Studio + llama.cpp)
//
// Both LM Studio and llama.cpp expose OpenAI-compatible endpoints,
// so this adapter extends OpenAICompatibleAdapter with local-specific
// defaults (no API key required, local health check, etc.)
// ─────────────────────────────────────────────────────────────────────────────

import { OpenAICompatibleAdapter } from './openai.js';
import type { ProviderConfig, ProviderHealth, HealthCheckOptions } from '../types.js';

const LOCAL_DEFAULT_URLS: Record<string, string> = {
  lmstudio: 'http://localhost:1234/v1',
  llamacpp: 'http://localhost:8080/v1',
};

export class LocalAdapter extends OpenAICompatibleAdapter {
  constructor(config: ProviderConfig) {
    super({
      ...config,
      // Local providers don't need API key validation
      apiKey: config.apiKey ?? 'local-no-key-required',
      baseUrl: config.baseUrl ?? LOCAL_DEFAULT_URLS[config.id] ?? 'http://localhost:1234/v1',
    });
  }

  /**
   * Override: local providers don't throw on missing API key.
   */
  protected override assertApiKey(): void {
    // Local providers are always keyless — no-op
  }

  /**
   * Override: faster timeout for health check (local should be instant).
   */
  override async healthCheck(opts: HealthCheckOptions): Promise<ProviderHealth> {
    return super.healthCheck({ ...opts, timeoutMs: opts.timeoutMs ?? 2_000 });
  }
}
packages/providers/src/adapters/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/adapters/index.ts
// Pass 17 Part 1
// ─────────────────────────────────────────────────────────────────────────────

export { BaseProviderAdapter } from './base.js';
export type { IProviderAdapter } from './base.js';
export { AnthropicAdapter } from './anthropic.js';
export { OpenAICompatibleAdapter } from './openai.js';
export { GeminiAdapter } from './gemini.js';
export { OllamaAdapter } from './ollama.js';
export { LocalAdapter } from './local.js';
packages/providers/src/routing/types.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/routing/types.ts
// Pass 17 Part 1 — Routing strategy contracts
// ─────────────────────────────────────────────────────────────────────────────

import type { IProviderAdapter } from '../adapters/base.js';
import type { ModelId, ProviderRequestOptions } from '../types.js';

export type RoutingStrategyName =
  | 'single'         // Always use the configured default provider/model
  | 'round-robin'    // Cycle through healthy providers
  | 'least-cost'     // Pick the cheapest model that satisfies requirements
  | 'least-latency'  // Pick the provider with lowest observed p50 latency
  | 'custom';        // Caller-supplied strategy function

export interface RoutingContext {
  /** The request about to be sent */
  readonly request: ProviderRequestOptions;
  /** Explicitly requested model (if any) — overrides strategy */
  readonly preferredModel?: ModelId;
  /** If true, skip unhealthy providers (default: true) */
  readonly skipUnhealthy: boolean;
  /** Capabilities required for this request */
  readonly requireToolUse: boolean;
  readonly requireVision: boolean;
  readonly requireStreaming: boolean;
  /** Max acceptable estimated cost per call (USD) — 0 = no limit */
  readonly maxCostPerCallUsd: number;
  /** Max acceptable context (tokens) needed */
  readonly estimatedInputTokens: number;
}

export interface RoutingDecision {
  readonly adapter: IProviderAdapter;
  readonly model: string;
  readonly rationale: string;
}

export interface RoutingStrategy {
  readonly name: RoutingStrategyName | string;
  /**
   * Select an adapter + model for the given context.
   * Returns null if no suitable adapter is available (caller should error).
   */
  select(
    adapters: ReadonlyMap<string, IProviderAdapter>,
    context: RoutingContext,
  ): RoutingDecision | null;
}
packages/providers/src/routing/strategies.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/routing/strategies.ts
// Pass 17 Part 1 — Built-in routing strategy implementations
// ─────────────────────────────────────────────────────────────────────────────

import type { IProviderAdapter } from '../adapters/base.js';
import type { RoutingContext, RoutingDecision, RoutingStrategy } from './types.js';
import { modelCatalog, estimateCostUsd } from '../capabilities.js';
import { toModelKey } from '../types.js';

// ── Helper: filter adapters to healthy + capable ones ────────────────────────

function filterCandidates(
  adapters: ReadonlyMap<string, IProviderAdapter>,
  ctx: RoutingContext,
): IProviderAdapter[] {
  const candidates: IProviderAdapter[] = [];

  for (const adapter of adapters.values()) {
    if (!adapter.config.enabled) continue;

    const model = adapter.config.defaultModel;
    const key = toModelKey({ provider: adapter.providerId, model });
    const entry = modelCatalog.get(key);

    if (!entry) continue;
    if (ctx.requireToolUse && !entry.capabilities.toolUse) continue;
    if (ctx.requireVision && !entry.capabilities.vision) continue;
    if (ctx.requireStreaming && !entry.capabilities.streaming) continue;
    if (ctx.estimatedInputTokens > entry.capabilities.contextWindowTokens) continue;

    if (ctx.maxCostPerCallUsd > 0) {
      const estCost = estimateCostUsd(key, ctx.estimatedInputTokens, 2048);
      if (estCost > ctx.maxCostPerCallUsd) continue;
    }

    candidates.push(adapter);
  }

  return candidates;
}

// ── SingleStrategy ────────────────────────────────────────────────────────────

export class SingleStrategy implements RoutingStrategy {
  readonly name = 'single';
  private readonly preferredProviderId: string;

  constructor(preferredProviderId: string) {
    this.preferredProviderId = preferredProviderId;
  }

  select(
    adapters: ReadonlyMap<string, IProviderAdapter>,
    ctx: RoutingContext,
  ): RoutingDecision | null {
    // If a preferred model was explicitly specified, honour it
    if (ctx.preferredModel) {
      const adapter = adapters.get(ctx.preferredModel.provider);
      if (adapter?.config.enabled) {
        return {
          adapter,
          model: ctx.preferredModel.model,
          rationale: `Explicit model requested: ${ctx.preferredModel.provider}:${ctx.preferredModel.model}`,
        };
      }
    }

    // Otherwise use configured default provider
    const adapter = adapters.get(this.preferredProviderId);
    if (!adapter?.config.enabled) return null;
    return {
      adapter,
      model: ctx.request.model || adapter.config.defaultModel,
      rationale: `Single strategy: using default provider "${this.preferredProviderId}"`,
    };
  }
}

// ── RoundRobinStrategy ────────────────────────────────────────────────────────

export class RoundRobinStrategy implements RoutingStrategy {
  readonly name = 'round-robin';
  private cursor = 0;

  select(
    adapters: ReadonlyMap<string, IProviderAdapter>,
    ctx: RoutingContext,
  ): RoutingDecision | null {
    const candidates = filterCandidates(adapters, ctx);
    if (!candidates.length) return null;

    const adapter = candidates[this.cursor % candidates.length]!;
    this.cursor = (this.cursor + 1) % candidates.length;

    return {
      adapter,
      model: adapter.config.defaultModel,
      rationale: `Round-robin: selected "${adapter.providerId}" (cursor ${this.cursor})`,
    };
  }
}

// ── LeastCostStrategy ─────────────────────────────────────────────────────────

export class LeastCostStrategy implements RoutingStrategy {
  readonly name = 'least-cost';

  select(
    adapters: ReadonlyMap<string, IProviderAdapter>,
    ctx: RoutingContext,
  ): RoutingDecision | null {
    const candidates = filterCandidates(adapters, ctx);
    if (!candidates.length) return null;

    let best: { adapter: IProviderAdapter; model: string; cost: number } | null = null;

    for (const adapter of candidates) {
      const model = adapter.config.defaultModel;
      const key = toModelKey({ provider: adapter.providerId, model });
      const cost = estimateCostUsd(key, ctx.estimatedInputTokens || 1000, 2048);

      if (!best || cost < best.cost) {
        best = { adapter, model, cost };
      }
    }

    if (!best) return null;
    return {
      adapter: best.adapter,
      model: best.model,
      rationale: `Least-cost: selected "${best.adapter.providerId}:${best.model}" (~$${best.cost.toFixed(6)}/call)`,
    };
  }
}

// ── LeastLatencyStrategy ──────────────────────────────────────────────────────

export class LeastLatencyStrategy implements RoutingStrategy {
  readonly name = 'least-latency';

  constructor(
    private readonly latencyMap: Map<string, number> = new Map(),
  ) {}

  observeLatency(providerId: string, latencyMs: number): void {
    // Simple EWMA: new = 0.3 * observed + 0.7 * previous
    const prev = this.latencyMap.get(providerId) ?? latencyMs;
    this.latencyMap.set(providerId, 0.3 * latencyMs + 0.7 * prev);
  }

  select(
    adapters: ReadonlyMap<string, IProviderAdapter>,
    ctx: RoutingContext,
  ): RoutingDecision | null {
    const candidates = filterCandidates(adapters, ctx);
    if (!candidates.length) return null;

    const best = candidates.reduce((prev, curr) => {
      const prevLatency = this.latencyMap.get(prev.providerId) ?? Infinity;
      const currLatency = this.latencyMap.get(curr.providerId) ?? Infinity;
      return currLatency < prevLatency ? curr : prev;
    });

    return {
      adapter: best,
      model: best.config.defaultModel,
      rationale: `Least-latency: selected "${best.providerId}" (p50 ~${Math.round(this.latencyMap.get(best.providerId) ?? 0)}ms)`,
    };
  }
}
packages/providers/src/routing/router.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/routing/router.ts
// Pass 17 Part 1 — ProviderRouter (canonical implementation)
//
// This is the canonical ProviderRouter for @locoworker/providers.
// packages/core's existing ProviderRouter will delegate to this in Pass 18.
// The bridge.ts wiring point shows how that delegation works without
// requiring a full core rewrite in this pass.
// ─────────────────────────────────────────────────────────────────────────────

import type { IProviderAdapter } from '../adapters/base.js';
import type {
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  ProviderRegistryEvent,
  ProviderRegistryListener,
  UsageStats,
} from '../types.js';
import { ProviderError } from '../types.js';
import type { RoutingContext, RoutingDecision, RoutingStrategy } from './types.js';
import { SingleStrategy } from './strategies.js';
import { toModelKey } from '../types.js';
import { SessionCostTracker } from '../cost.js';

export interface ProviderRouterOptions {
  strategy: RoutingStrategy;
  /** Provider ID to fall back to if primary selection fails */
  fallbackProviderId?: string;
  /** Max retry attempts on retryable errors */
  maxRetries?: number;
  /** Whether to automatically update LeastLatencyStrategy with observed latencies */
  trackLatency?: boolean;
}

export class ProviderRouter {
  private readonly adapters: Map<string, IProviderAdapter> = new Map();
  private strategy: RoutingStrategy;
  private readonly fallbackProviderId: string | undefined;
  private readonly maxRetries: number;
  private readonly listeners: ProviderRegistryListener[] = [];
  private costTracker: SessionCostTracker | null = null;

  constructor(opts: ProviderRouterOptions) {
    this.strategy = opts.strategy;
    this.fallbackProviderId = opts.fallbackProviderId;
    this.maxRetries = opts.maxRetries ?? 2;
  }

  // ── Adapter management ──────────────────────────────────────────────────────

  registerAdapter(adapter: IProviderAdapter): void {
    this.adapters.set(adapter.providerId, adapter);
    this.emit({ type: 'provider_registered', providerId: adapter.providerId });
  }

  unregisterAdapter(providerId: string): void {
    this.adapters.delete(providerId);
    this.emit({ type: 'provider_unregistered', providerId: providerId as import('../types.js').ProviderId });
  }

  getAdapter(providerId: string): IProviderAdapter | undefined {
    return this.adapters.get(providerId);
  }

  listAdapters(): readonly IProviderAdapter[] {
    return [...this.adapters.values()];
  }

  // ── Strategy management ─────────────────────────────────────────────────────

  setStrategy(strategy: RoutingStrategy): void {
    this.strategy = strategy;
  }

  // ── Cost tracking ───────────────────────────────────────────────────────────

  setCostTracker(tracker: SessionCostTracker): void {
    this.costTracker = tracker;
  }

  // ── Core routing ────────────────────────────────────────────────────────────

  /**
   * Route a complete() call through the strategy with retry + fallback.
   */
  async route(
    options: ProviderRequestOptions,
    ctx: Partial<RoutingContext> = {},
  ): Promise<ProviderResponse> {
    const fullCtx = this.buildContext(options, ctx);
    let lastError: Error | null = null;

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      const decision = this.strategy.select(this.adapters, fullCtx);

      if (!decision) {
        // Try fallback strategy if configured
        const fallbackDecision = this.tryFallback(fullCtx);
        if (!fallbackDecision) break;
        return this.executeDecision(fallbackDecision, options);
      }

      try {
        const response = await this.executeDecision(decision, options);
        this.recordSuccess(decision, response.usage, response.latencyMs);
        return response;
      } catch (err) {
        lastError = err instanceof Error ? err : new Error(String(err));
        const isRetryable = err instanceof ProviderError && err.retryable;
        if (!isRetryable || attempt === this.maxRetries) break;

        // Exponential backoff before retry
        await new Promise((r) => setTimeout(r, Math.min(1000 * 2 ** attempt, 8000)));
      }
    }

    throw lastError ?? new ProviderError('unknown', 'All routing attempts failed.', 'anthropic', false);
  }

  /**
   * Route a stream() call through the strategy (no retry — streams are
   * not safely retryable once started).
   */
  async *routeStream(
    options: ProviderRequestOptions,
    ctx: Partial<RoutingContext> = {},
  ): AsyncGenerator<StreamEvent, void, unknown> {
    const fullCtx = this.buildContext(options, { ...ctx, requireStreaming: true });
    const decision = this.strategy.select(this.adapters, fullCtx);

    if (!decision) {
      throw new ProviderError(
        'provider_unavailable',
        'No streaming-capable provider available.',
        'anthropic',
        false,
      );
    }

    const start = performance.now();
    let outputTokens = 0;

    try {
      for await (const event of decision.adapter.stream({
        ...options,
        model: decision.model,
      })) {
        if (event.type === 'content_block_delta' && event.delta.type === 'text') {
          // Rough token count for cost estimation
          outputTokens += Math.ceil(event.delta.text.length / 4);
        }
        if (event.type === 'message_stop') {
          const latencyMs = Math.round(performance.now() - start);
          this.recordSuccess(decision, { inputTokens: 0, outputTokens }, latencyMs);
        }
        yield event;
      }
    } catch (err) {
      this.emit({
        type: 'request_failed',
        providerId: decision.adapter.providerId,
        model: decision.model,
        error: err instanceof ProviderError ? err : new ProviderError('unknown', String(err), decision.adapter.providerId, false),
      });
      throw err;
    }
  }

  // ── Events ──────────────────────────────────────────────────────────────────

  on(listener: ProviderRegistryListener): () => void {
    this.listeners.push(listener);
    return () => {
      const idx = this.listeners.indexOf(listener);
      if (idx !== -1) this.listeners.splice(idx, 1);
    };
  }

  private emit(event: ProviderRegistryEvent): void {
    for (const l of this.listeners) {
      try { l(event); } catch { /* listener errors never propagate */ }
    }
  }

  // ── Private helpers ─────────────────────────────────────────────────────────

  private buildContext(
    options: ProviderRequestOptions,
    partial: Partial<RoutingContext>,
  ): RoutingContext {
    return {
      request: options,
      preferredModel: partial.preferredModel,
      skipUnhealthy: partial.skipUnhealthy ?? true,
      requireToolUse: partial.requireToolUse ?? (options.tools?.length ?? 0) > 0,
      requireVision: partial.requireVision ?? false,
      requireStreaming: partial.requireStreaming ?? (options.stream ?? false),
      maxCostPerCallUsd: partial.maxCostPerCallUsd ?? 0,
      estimatedInputTokens: partial.estimatedInputTokens ?? 0,
    };
  }

  private tryFallback(ctx: RoutingContext): RoutingDecision | null {
    if (!this.fallbackProviderId) return null;
    const fallbackStrategy = new SingleStrategy(this.fallbackProviderId);
    return fallbackStrategy.select(this.adapters, ctx);
  }

  private async executeDecision(
    decision: RoutingDecision,
    options: ProviderRequestOptions,
  ): Promise<ProviderResponse> {
    return decision.adapter.complete({
      ...options,
      model: decision.model,
    });
  }

  private recordSuccess(decision: RoutingDecision, usage: UsageStats, latencyMs: number): void {
    const modelKey = toModelKey({ provider: decision.adapter.providerId, model: decision.model });

    if (this.costTracker) {
      try {
        this.costTracker.record(decision.adapter.providerId, modelKey, usage);
      } catch (err) {
        // Re-throw cost cap errors — they are intentional hard stops
        throw err;
      }
    }

    this.emit({
      type: 'request_completed',
      providerId: decision.adapter.providerId,
      model: decision.model,
      usage,
      latencyMs,
    });
  }
}
packages/providers/src/routing/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/routing/index.ts
// Pass 17 Part 1
// ─────────────────────────────────────────────────────────────────────────────

export type { RoutingStrategyName, RoutingContext, RoutingDecision, RoutingStrategy } from './types.js';
export { SingleStrategy, RoundRobinStrategy, LeastCostStrategy, LeastLatencyStrategy } from './strategies.js';
export { ProviderRouter } from './router.js';
export type { ProviderRouterOptions } from './router.js';
packages/providers/src/registry.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/registry.ts
// Pass 17 Part 1 — ProviderRegistry (main class)
//
// The ProviderRegistry is the top-level entry point for the providers package.
// It owns all adapters, the router, health monitor, and cost tracking.
// packages/core will instantiate one ProviderRegistry per process and share it
// via dependency injection (bridge.ts shows the wiring).
// ─────────────────────────────────────────────────────────────────────────────

import type { IProviderAdapter } from './adapters/base.js';
import {
  AnthropicAdapter,
  GeminiAdapter,
  OllamaAdapter,
  LocalAdapter,
  OpenAICompatibleAdapter,
} from './adapters/index.js';
import type {
  ProviderConfig,
  ProviderConfigInput,
  ProviderId,
  ProviderHealth,
  ProviderRegistryEvent,
  ProviderRegistryListener,
} from './types.js';
import { ProviderHealthMonitor } from './health.js';
import { SessionCostTracker, DailyCostTracker } from './cost.js';
import { ProviderRouter } from './routing/router.js';
import {
  SingleStrategy,
  RoundRobinStrategy,
  LeastCostStrategy,
  LeastLatencyStrategy,
} from './routing/strategies.js';
import type { RoutingStrategyName } from './routing/types.js';
import { modelCatalog } from './capabilities.js';
import type { ModelCatalogEntry } from './capabilities.js';

export interface ProviderRegistryConfig {
  /** Default provider to use when none specified */
  defaultProvider: ProviderId;
  /** Fallback provider if default is unavailable */
  fallbackProvider?: ProviderId;
  /** Routing strategy name */
  routingStrategy: RoutingStrategyName;
  /** Session cost cap in USD (0 = no limit) */
  sessionCostCapUsd: number;
  /** Daily cost cap in USD (0 = no limit) */
  dailyCostCapUsd: number;
  /** Cost warn threshold fraction (0–1) */
  costWarnThreshold: number;
  /** Whether to run periodic health checks */
  enableHealthChecks: boolean;
  /** Health check interval in ms */
  healthCheckIntervalMs: number;
  /** Individual provider configs */
  providers: readonly ProviderConfigInput[];
}

export class ProviderRegistry {
  private readonly adapters = new Map<ProviderId, IProviderAdapter>();
  private readonly router: ProviderRouter;
  private readonly healthMonitor: ProviderHealthMonitor;
  private readonly dailyCostTracker: DailyCostTracker;
  private readonly listeners: ProviderRegistryListener[] = [];

  constructor(private readonly config: ProviderRegistryConfig) {
    // Build routing strategy
    const strategy = this.buildStrategy(config.routingStrategy, config.defaultProvider);

    this.router = new ProviderRouter({
      strategy,
      fallbackProviderId: config.fallbackProvider,
      maxRetries: 2,
    });

    this.healthMonitor = new ProviderHealthMonitor(config.healthCheckIntervalMs);
    this.dailyCostTracker = new DailyCostTracker(config.dailyCostCapUsd);

    // Register each provider
    for (const providerInput of config.providers) {
      if (providerInput.enabled === false) continue;
      this.registerProvider(providerInput);
    }

    if (config.enableHealthChecks) {
      this.healthMonitor.startPeriodicChecks();
    }

    // Forward router events to registry listeners
    this.router.on((event) => this.emit(event));
  }

  // ── Provider management ─────────────────────────────────────────────────────

  registerProvider(input: ProviderConfigInput): void {
    const config = this.buildFullConfig(input);
    const adapter = this.createAdapter(config);

    this.adapters.set(config.id, adapter);
    this.router.registerAdapter(adapter);
    this.healthMonitor.register(config.id, (id, opts) => adapter.healthCheck(opts));
  }

  unregisterProvider(providerId: ProviderId): void {
    this.adapters.delete(providerId);
    this.router.unregisterAdapter(providerId);
    this.healthMonitor.unregister(providerId);
    this.emit({ type: 'provider_unregistered', providerId });
  }

  updateProvider(input: ProviderConfigInput): void {
    const adapter = this.adapters.get(input.id as ProviderId);
    if (!adapter) throw new Error(`Provider "${input.id}" is not registered.`);
    adapter.updateConfig(input as Partial<ProviderConfig>);
    this.emit({ type: 'provider_enabled', providerId: input.id as ProviderId });
  }

  getAdapter(providerId: ProviderId): IProviderAdapter | undefined {
    return this.adapters.get(providerId);
  }

  listAdapters(): readonly IProviderAdapter[] {
    return [...this.adapters.values()];
  }

  // ── Router access ───────────────────────────────────────────────────────────

  getRouter(): ProviderRouter {
    return this.router;
  }

  /**
   * Create a session-scoped cost tracker and attach it to the router.
   * Returns the tracker so callers can read summaries.
   */
  createSessionCostTracker(
    sessionId: string,
    costCapUsd?: number,
    onCapExceeded?: (summary: import('./cost.js').SessionCostSummary) => void,
  ): SessionCostTracker {
    const tracker = new SessionCostTracker(
      sessionId,
      costCapUsd ?? this.config.sessionCostCapUsd,
      this.config.costWarnThreshold,
      onCapExceeded,
    );
    this.router.setCostTracker(tracker);
    return tracker;
  }

  // ── Health ──────────────────────────────────────────────────────────────────

  async checkHealth(providerId?: ProviderId): Promise<readonly ProviderHealth[]> {
    if (providerId) {
      const h = await this.healthMonitor.checkNow(providerId);
      return [h];
    }
    return this.healthMonitor.checkAll();
  }

  getAllHealth(): readonly ProviderHealth[] {
    return this.healthMonitor.getAllHealth();
  }

  // ── Catalog ─────────────────────────────────────────────────────────────────

  listModels(providerId?: ProviderId): readonly ModelCatalogEntry[] {
    if (providerId) return modelCatalog.getByProvider(providerId);
    return modelCatalog.all();
  }

  /**
   * Dynamically refresh model list from a local provider (e.g. Ollama)
   * and register new models in the catalog.
   */
  async refreshLocalModels(providerId: ProviderId): Promise<readonly string[]> {
    const adapter = this.adapters.get(providerId);
    if (!adapter) return [];
    const models = await adapter.listModels();

    for (const model of models) {
      const key = `${providerId}:${model}` as import('./types.js').ModelKey;
      if (!modelCatalog.get(key)) {
        modelCatalog.register({
          modelKey: key,
          provider: providerId,
          modelId: model,
          displayName: `${model} (${providerId})`,
          capabilities: {
            toolUse: false,
            streaming: true,
            vision: false,
            systemPrompt: true,
            promptCaching: false,
            contextWindowTokens: 128_000,
            maxOutputTokens: 8_192,
            jsonMode: false,
          },
          pricing: { inputCostPer1M: 0, outputCostPer1M: 0 },
        });
      }
    }

    return models;
  }

  // ── Events ──────────────────────────────────────────────────────────────────

  on(listener: ProviderRegistryListener): () => void {
    this.listeners.push(listener);
    return () => {
      const idx = this.listeners.indexOf(listener);
      if (idx !== -1) this.listeners.splice(idx, 1);
    };
  }

  private emit(event: ProviderRegistryEvent): void {
    for (const l of this.listeners) {
      try { l(event); } catch { /* never propagate */ }
    }
  }

  // ── Factory helpers ─────────────────────────────────────────────────────────

  private buildFullConfig(input: ProviderConfigInput): ProviderConfig {
    const defaults: ProviderConfig = {
      id: input.id as ProviderId,
      category: this.inferCategory(input.id as ProviderId),
      displayName: input.id,
      enabled: true,
      timeoutMs: 60_000,
      maxRetries: 2,
      defaultModel: modelCatalog.getDefault(input.id as ProviderId)?.modelId ?? 'unknown',
    };
    return { ...defaults, ...input } as ProviderConfig;
  }

  private inferCategory(id: ProviderId): import('./types.js').ProviderCategory {
    if (['ollama', 'lmstudio', 'llamacpp'].includes(id)) return 'local';
    if (id === 'custom') return 'custom';
    return 'cloud';
  }

  private createAdapter(config: ProviderConfig): IProviderAdapter {
    switch (config.id) {
      case 'anthropic': return new AnthropicAdapter(config);
      case 'gemini': return new GeminiAdapter(config);
      case 'ollama': return new OllamaAdapter(config);
      case 'lmstudio':
      case 'llamacpp': return new LocalAdapter(config);
      case 'openai':
      case 'groq':
      case 'mistral':
      case 'cohere':
      case 'custom':
      case 'mcp':
      default: return new OpenAICompatibleAdapter(config);
    }
  }

  private buildStrategy(
    name: RoutingStrategyName,
    defaultProvider: ProviderId,
  ) {
    switch (name) {
      case 'round-robin': return new RoundRobinStrategy();
      case 'least-cost': return new LeastCostStrategy();
      case 'least-latency': return new LeastLatencyStrategy();
      case 'single':
      default: return new SingleStrategy(defaultProvider);
    }
  }

  /**
   * Construct a ProviderRegistry from environment variables.
   * This is the factory used by packages/core's bridge.ts.
   */
  static fromEnv(): ProviderRegistry {
    const env = process.env;

    const providerInputs: ProviderConfigInput[] = [
      { id: 'anthropic', apiKey: env['ANTHROPIC_API_KEY'], enabled: !!env['ANTHROPIC_API_KEY'] },
      { id: 'openai', apiKey: env['OPENAI_API_KEY'], enabled: !!env['OPENAI_API_KEY'] },
      { id: 'gemini', apiKey: env['GEMINI_API_KEY'], enabled: !!env['GEMINI_API_KEY'] },
      { id: 'mistral', apiKey: env['MISTRAL_API_KEY'], enabled: !!env['MISTRAL_API_KEY'] },
      { id: 'groq', apiKey: env['GROQ_API_KEY'], enabled: !!env['GROQ_API_KEY'] },
      { id: 'cohere', apiKey: env['COHERE_API_KEY'], enabled: !!env['COHERE_API_KEY'] },
      { id: 'ollama', baseUrl: env['OLLAMA_BASE_URL'], enabled: !!env['OLLAMA_BASE_URL'] },
      { id: 'lmstudio', baseUrl: env['LMSTUDIO_BASE_URL'], enabled: !!env['LMSTUDIO_BASE_URL'] },
      { id: 'llamacpp', baseUrl: env['LLAMACPP_BASE_URL'], enabled: !!env['LLAMACPP_BASE_URL'] },
    ];

    return new ProviderRegistry({
      defaultProvider: (env['DEFAULT_PROVIDER'] as ProviderId) ?? 'anthropic',
      fallbackProvider: (env['FALLBACK_PROVIDER'] as ProviderId) ?? undefined,
      routingStrategy: (env['PROVIDER_ROUTING_STRATEGY'] as RoutingStrategyName) ?? 'single',
      sessionCostCapUsd: parseFloat(env['LOCOWORKER_COST_CAP_USD'] ?? '5.0'),
      dailyCostCapUsd: parseFloat(env['LOCOWORKER_DAILY_COST_CAP_USD'] ?? '50.0'),
      costWarnThreshold: parseFloat(env['LOCOWORKER_COST_WARN_THRESHOLD'] ?? '0.8'),
      enableHealthChecks: false, // off by default; callers opt in
      healthCheckIntervalMs: 60_000,
      providers: providerInputs,
    });
  }

  destroy(): void {
    this.healthMonitor.stopPeriodicChecks();
  }
}
packages/providers/src/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/index.ts
// Pass 17 Part 1 — Barrel export
// ─────────────────────────────────────────────────────────────────────────────

// ── Core types ────────────────────────────────────────────────────────────────
export type {
  ProviderId,
  ProviderCategory,
  ModelId,
  ModelKey,
  ProviderConfig,
  ProviderConfigInput,
  MessageRole,
  TextContent,
  ImageContent,
  ToolUseContent,
  ToolResultContent,
  ContentBlock,
  Message,
  ProviderToolDefinition,
  ProviderRequestOptions,
  UsageStats,
  ProviderResponse,
  StreamEvent,
  ProviderHealthStatus,
  ProviderHealth,
  ProviderErrorCode,
  ProviderRegistryEvent,
  ProviderRegistryListener,
} from './types.js';

export { ProviderError, toModelKey, parseModelKey } from './types.js';

// ── Capabilities + cost ───────────────────────────────────────────────────────
export type { ModelCapabilities, ModelPricing, ModelCatalogEntry } from './capabilities.js';
export { modelCatalog, estimateCostUsd } from './capabilities.js';

export type { CostRecord, SessionCostSummary } from './cost.js';
export { SessionCostTracker, DailyCostTracker } from './cost.js';

// ── Health ────────────────────────────────────────────────────────────────────
export type { HealthCheckOptions, HealthCheckFn } from './health.js';
export { ProviderHealthMonitor } from './health.js';

// ── Adapters ──────────────────────────────────────────────────────────────────
export type { IProviderAdapter } from './adapters/base.js';
export { BaseProviderAdapter } from './adapters/base.js';
export { AnthropicAdapter } from './adapters/anthropic.js';
export { OpenAICompatibleAdapter } from './adapters/openai.js';
export { GeminiAdapter } from './adapters/gemini.js';
export { OllamaAdapter } from './adapters/ollama.js';
export { LocalAdapter } from './adapters/local.js';

// ── Routing ───────────────────────────────────────────────────────────────────
export type {
  RoutingStrategyName,
  RoutingContext,
  RoutingDecision,
  RoutingStrategy,
} from './routing/types.js';
export {
  SingleStrategy,
  RoundRobinStrategy,
  LeastCostStrategy,
  LeastLatencyStrategy,
} from './routing/strategies.js';
export { ProviderRouter } from './routing/router.js';
export type { ProviderRouterOptions } from './routing/router.js';

// ── Registry (top-level entry point) ─────────────────────────────────────────
export { ProviderRegistry } from './registry.js';
export type { ProviderRegistryConfig } from './registry.js';
packages/providers/src/__tests__/registry.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/__tests__/registry.test.ts
// Pass 17 Part 1
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, beforeEach, vi } from 'vitest';
import { ProviderRegistry } from '../registry.js';
import type { ProviderRegistryConfig } from '../registry.js';

const BASE_CONFIG: ProviderRegistryConfig = {
  defaultProvider: 'anthropic',
  routingStrategy: 'single',
  sessionCostCapUsd: 5.0,
  dailyCostCapUsd: 50.0,
  costWarnThreshold: 0.8,
  enableHealthChecks: false,
  healthCheckIntervalMs: 60_000,
  providers: [
    { id: 'anthropic', apiKey: 'test-key-anthropic', enabled: true },
    { id: 'openai', apiKey: 'test-key-openai', enabled: true },
    { id: 'ollama', baseUrl: 'http://localhost:11434', enabled: true },
  ],
};

describe('ProviderRegistry', () => {
  let registry: ProviderRegistry;

  beforeEach(() => {
    registry = new ProviderRegistry(BASE_CONFIG);
  });

  it('registers providers from config', () => {
    const adapters = registry.listAdapters();
    expect(adapters).toHaveLength(3);
    expect(adapters.map((a) => a.providerId)).toContain('anthropic');
    expect(adapters.map((a) => a.providerId)).toContain('openai');
    expect(adapters.map((a) => a.providerId)).toContain('ollama');
  });

  it('skips disabled providers', () => {
    const reg = new ProviderRegistry({
      ...BASE_CONFIG,
      providers: [
        { id: 'anthropic', apiKey: 'key', enabled: true },
        { id: 'openai', apiKey: 'key', enabled: false },
      ],
    });
    expect(reg.listAdapters()).toHaveLength(1);
  });

  it('getAdapter returns registered adapter', () => {
    const adapter = registry.getAdapter('anthropic');
    expect(adapter).toBeDefined();
    expect(adapter?.providerId).toBe('anthropic');
  });

  it('getAdapter returns undefined for unknown provider', () => {
    expect(registry.getAdapter('unknown' as never)).toBeUndefined();
  });

  it('emits provider_registered event on registerProvider', () => {
    const events: string[] = [];
    registry.on((e) => events.push(e.type));
    registry.registerProvider({ id: 'groq', apiKey: 'test', enabled: true });
    expect(events).toContain('provider_registered');
  });

  it('emits provider_unregistered event on unregisterProvider', () => {
    const events: string[] = [];
    registry.on((e) => events.push(e.type));
    registry.unregisterProvider('ollama');
    expect(events).toContain('provider_unregistered');
    expect(registry.getAdapter('ollama')).toBeUndefined();
  });

  it('creates session cost tracker with configured cap', () => {
    const tracker = registry.createSessionCostTracker('session-1');
    expect(tracker).toBeDefined();
    expect(tracker.remainingBudgetUsd()).toBe(5.0);
  });

  it('creates session cost tracker with custom cap', () => {
    const tracker = registry.createSessionCostTracker('session-2', 2.5);
    expect(tracker.remainingBudgetUsd()).toBe(2.5);
  });

  it('listModels returns catalog entries for provider', () => {
    const models = registry.listModels('anthropic');
    expect(models.length).toBeGreaterThan(0);
    expect(models.every((m) => m.provider === 'anthropic')).toBe(true);
  });

  it('listModels returns all models when no provider specified', () => {
    const all = registry.listModels();
    const anthropic = registry.listModels('anthropic');
    expect(all.length).toBeGreaterThanOrEqual(anthropic.length);
  });

  it('fromEnv constructs registry from environment', () => {
    vi.stubEnv('ANTHROPIC_API_KEY', 'env-key');
    vi.stubEnv('DEFAULT_PROVIDER', 'anthropic');
    const reg = ProviderRegistry.fromEnv();
    expect(reg.getAdapter('anthropic')).toBeDefined();
    vi.unstubAllEnvs();
  });
});
packages/providers/src/__tests__/routing.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/__tests__/routing.test.ts
// Pass 17 Part 1
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, beforeEach } from 'vitest';
import {
  SingleStrategy,
  RoundRobinStrategy,
  LeastCostStrategy,
  LeastLatencyStrategy,
} from '../routing/strategies.js';
import type { IProviderAdapter } from '../adapters/base.js';
import type { RoutingContext, RoutingDecision } from '../routing/types.js';
import type { ProviderConfig, ProviderRequestOptions, ProviderResponse, StreamEvent, ProviderHealth, HealthCheckOptions } from '../types.js';

// ── Minimal mock adapter ──────────────────────────────────────────────────────

function makeAdapter(
  providerId: string,
  defaultModel: string,
  enabled = true,
): IProviderAdapter {
  const config: ProviderConfig = {
    id: providerId as never,
    category: 'cloud',
    displayName: providerId,
    enabled,
    timeoutMs: 30_000,
    maxRetries: 2,
    defaultModel,
  };

  return {
    providerId: providerId as never,
    config,
    complete: async (_o: ProviderRequestOptions): Promise<ProviderResponse> => { throw new Error('mock'); },
    stream: async function* (_o: ProviderRequestOptions): AsyncGenerator<StreamEvent> {},
    healthCheck: async (_o: HealthCheckOptions): Promise<ProviderHealth> => ({
      providerId: providerId as never,
      status: 'healthy',
      lastCheckedAt: Date.now(),
    }),
    listModels: async () => [defaultModel],
    updateConfig: (_p: Partial<ProviderConfig>) => {},
  };
}

function makeContext(partial: Partial<RoutingContext> = {}): RoutingContext {
  return {
    request: { model: '', messages: [] },
    skipUnhealthy: true,
    requireToolUse: false,
    requireVision: false,
    requireStreaming: false,
    maxCostPerCallUsd: 0,
    estimatedInputTokens: 1000,
    ...partial,
  };
}

describe('SingleStrategy', () => {
  it('selects the configured default provider', () => {
    const strategy = new SingleStrategy('anthropic');
    const adapters = new Map([
      ['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')],
      ['openai', makeAdapter('openai', 'gpt-4o')],
    ]);
    const decision = strategy.select(adapters, makeContext());
    expect(decision).not.toBeNull();
    expect(decision!.adapter.providerId).toBe('anthropic');
  });

  it('returns null when default provider is not registered', () => {
    const strategy = new SingleStrategy('gemini');
    const adapters = new Map([['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')]]);
    expect(strategy.select(adapters, makeContext())).toBeNull();
  });

  it('returns null when default provider is disabled', () => {
    const strategy = new SingleStrategy('anthropic');
    const adapters = new Map([['anthropic', makeAdapter('anthropic', 'claude-opus-4-5', false)]]);
    expect(strategy.select(adapters, makeContext())).toBeNull();
  });

  it('honours explicit preferredModel over default', () => {
    const strategy = new SingleStrategy('anthropic');
    const adapters = new Map([
      ['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')],
      ['openai', makeAdapter('openai', 'gpt-4o')],
    ]);
    const ctx = makeContext({ preferredModel: { provider: 'openai', model: 'gpt-4o-mini' } });
    const decision = strategy.select(adapters, ctx);
    expect(decision?.adapter.providerId).toBe('openai');
    expect(decision?.model).toBe('gpt-4o-mini');
  });
});

describe('RoundRobinStrategy', () => {
  it('cycles through candidates', () => {
    const strategy = new RoundRobinStrategy();
    const adapters = new Map([
      ['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')],
      ['openai', makeAdapter('openai', 'gpt-4o')],
    ]);

    const results: string[] = [];
    for (let i = 0; i < 4; i++) {
      const d = strategy.select(adapters, makeContext());
      if (d) results.push(d.adapter.providerId);
    }

    // Should alternate between the two providers
    expect(new Set(results).size).toBe(2);
  });
});

describe('LeastCostStrategy', () => {
  it('selects the lower-cost provider', () => {
    const strategy = new LeastCostStrategy();
    // groq is cheaper than anthropic
    const adapters = new Map([
      ['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')],
      ['groq', makeAdapter('groq', 'llama-3.3-70b-versatile')],
    ]);
    const decision = strategy.select(adapters, makeContext());
    expect(decision).not.toBeNull();
    // Groq is significantly cheaper, so it should win
    expect(decision!.adapter.providerId).toBe('groq');
  });
});

describe('LeastLatencyStrategy', () => {
  it('selects provider with lowest observed latency', () => {
    const strategy = new LeastLatencyStrategy();
    const adapters = new Map([
      ['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')],
      ['groq', makeAdapter('groq', 'llama-3.3-70b-versatile')],
    ]);

    strategy.observeLatency('anthropic', 800);
    strategy.observeLatency('groq', 200);

    const decision = strategy.select(adapters, makeContext());
    expect(decision?.adapter.providerId).toBe('groq');
  });

  it('returns any candidate when no latency data available', () => {
    const strategy = new LeastLatencyStrategy();
    const adapters = new Map([['anthropic', makeAdapter('anthropic', 'claude-opus-4-5')]]);
    const decision = strategy.select(adapters, makeContext());
    expect(decision).not.toBeNull();
  });
});
packages/providers/src/__tests__/cost.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/providers/src/__tests__/cost.test.ts
// Pass 17 Part 1
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, vi } from 'vitest';
import { SessionCostTracker, DailyCostTracker } from '../cost.js';
import { estimateCostUsd } from '../capabilities.js';
import { toModelKey } from '../types.js';

describe('estimateCostUsd', () => {
  it('returns 0 for unknown model', () => {
    const cost = estimateCostUsd('unknown:unknown-model' as never, 1000, 500);
    expect(cost).toBe(0);
  });

  it('calculates cost for claude-opus-4-5', () => {
    const key = toModelKey({ provider: 'anthropic', model: 'claude-opus-4-5' });
    const cost = estimateCostUsd(key, 1_000_000, 1_000_000);
    // Input: $15, Output: $75 = $90
    expect(cost).toBeCloseTo(90, 2);
  });

  it('calculates cost for groq llama (cheap)', () => {
    const key = toModelKey({ provider: 'groq', model: 'llama-3.3-70b-versatile' });
    const cost = estimateCostUsd(key, 1_000_000, 1_000_000);
    // Input: $0.59, Output: $0.79 = $1.38
    expect(cost).toBeCloseTo(1.38, 2);
  });

  it('returns 0 for local models (ollama)', () => {
    const key = toModelKey({ provider: 'ollama', model: 'llama3.2' });
    const cost = estimateCostUsd(key, 1_000_000, 1_000_000);
    expect(cost).toBe(0);
  });

  it('includes cache costs when provided', () => {
    const key = toModelKey({ provider: 'anthropic', model: 'claude-opus-4-5' });
    const withCache = estimateCostUsd(key, 1_000_000, 1_000_000, 1_000_000, 1_000_000);
    const withoutCache = estimateCostUsd(key, 1_000_000, 1_000_000, 0, 0);
    expect(withCache).toBeGreaterThan(withoutCache);
  });
});

describe('SessionCostTracker', () => {
  const modelKey = toModelKey({ provider: 'anthropic', model: 'claude-opus-4-5' });

  it('records a call and returns a summary', () => {
    const tracker = new SessionCostTracker('s-1', 10);
    const { summary } = tracker.record('anthropic', modelKey, {
      inputTokens: 1000,
      outputTokens: 500,
    });
    expect(summary.callCount).toBe(1);
    expect(summary.totalInputTokens).toBe(1000);
    expect(summary.totalOutputTokens).toBe(500);
    expect(summary.totalEstimatedCostUsd).toBeGreaterThan(0);
  });

  it('accumulates multiple calls', () => {
    const tracker = new SessionCostTracker('s-2', 100);
    tracker.record('anthropic', modelKey, { inputTokens: 500, outputTokens: 250 });
    tracker.record('anthropic', modelKey, { inputTokens: 500, outputTokens: 250 });
    expect(tracker.summarise().callCount).toBe(2);
    expect(tracker.summarise().totalInputTokens).toBe(1000);
  });

  it('throws when cost cap is exceeded', () => {
    const tracker = new SessionCostTracker('s-3', 0.000001); // impossibly low cap
    expect(() =>
      tracker.record('anthropic', modelKey, { inputTokens: 100_000, outputTokens: 50_000 }),
    ).toThrow(/cost cap exceeded/i);
  });

  it('calls onCapExceeded callback', () => {
    const onCap = vi.fn();
    const tracker = new SessionCostTracker('s-4', 0.000001, 0.8, onCap);
    try {
      tracker.record('anthropic', modelKey, { inputTokens: 100_000, outputTokens: 50_000 });
    } catch { /* expected */ }
    expect(onCap).toHaveBeenCalledOnce();
  });

  it('returns Infinity remaining budget when no cap set', () => {
    const tracker = new SessionCostTracker('s-5', 0);
    expect(tracker.remainingBudgetUsd()).toBe(Infinity);
  });

  it('decreases remaining budget after a call', () => {
    const tracker = new SessionCostTracker('s-6', 10);
    const before = tracker.remainingBudgetUsd();
    tracker.record('anthropic', modelKey, { inputTokens: 1000, outputTokens: 500 });
    expect(tracker.remainingBudgetUsd()).toBeLessThan(before);
  });

  it('resets cleanly', () => {
    const tracker = new SessionCostTracker('s-7', 10);
    tracker.record('anthropic', modelKey, { inputTokens: 1000, outputTokens: 500 });
    tracker.reset();
    expect(tracker.summarise().callCount).toBe(0);
    expect(tracker.remainingBudgetUsd()).toBe(10);
  });
});

describe('DailyCostTracker', () => {
  it('accumulates daily costs', () => {
    const tracker = new DailyCostTracker(100);
    tracker.add(5.0);
    tracker.add(3.0);
    expect(tracker.todayCostUsd()).toBeCloseTo(8.0, 5);
  });

  it('throws when daily cap exceeded', () => {
    const tracker = new DailyCostTracker(5.0);
    tracker.add(4.0);
    expect(() => tracker.add(2.0)).toThrow(/daily cost cap exceeded/i);
  });

  it('does not throw when cap is 0 (no limit)', () => {
    const tracker = new DailyCostTracker(0);
    expect(() => tracker.add(1000)).not.toThrow();
  });
});
packages/core/src/providers/bridge.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/core/src/providers/bridge.ts
// Pass 17 Part 1 — Wiring point: how core delegates to @locoworker/providers
//
// ── What this file is ───────────────────────────────────────────────────────
// This is NOT a full replacement of core's existing ProviderRouter.
// It is a WIRING POINT — a bridge that shows exactly how core should
// consume @locoworker/providers in Pass 18.
//
// In Pass 18, core's existing ProviderRouter will be replaced/updated to
// use ProviderRegistry.fromEnv() and ProviderRouter from this package.
// For now, this file:
//   1. Exports a singleton factory (createProvidersFromEnv) that core's
//      bootstrap can call to get a fully configured registry.
//   2. Re-exports the types core needs from @locoworker/providers so
//      core does not need to import from the providers package directly
//      until Pass 18 makes the cutover.
//   3. Documents the exact ProviderRouter method signatures core needs
//      to fulfil — acting as a test-at-compile-time contract.
//
// ── Dependency direction ───────────────────────────────────────────────────
//   packages/providers   (no dependency on core)
//       ↑
//   packages/core/src/providers/bridge.ts  (core depends on providers)
//       ↑
//   packages/core/src/ProviderRouter.ts    (currently self-contained)
//       ↑ (Pass 18: replace with delegation to bridge)
//
// ─────────────────────────────────────────────────────────────────────────────

// ── Re-export types core needs ────────────────────────────────────────────────
export type {
  ProviderId,
  ProviderConfig,
  ProviderRequestOptions,
  ProviderResponse,
  StreamEvent,
  UsageStats,
  Message,
  ContentBlock,
  ProviderToolDefinition,
  ProviderHealth,
  ProviderError,
  ModelId,
  ModelKey,
} from '@locoworker/providers';

export {
  ProviderRegistry,
  ProviderRouter,
  SingleStrategy,
  RoundRobinStrategy,
  LeastCostStrategy,
  LeastLatencyStrategy,
  modelCatalog,
  estimateCostUsd,
  SessionCostTracker,
  DailyCostTracker,
  toModelKey,
  parseModelKey,
} from '@locoworker/providers';

// ── Singleton factory for core bootstrap ─────────────────────────────────────

import { ProviderRegistry } from '@locoworker/providers';

let _registryInstance: ProviderRegistry | null = null;

/**
 * Returns a singleton ProviderRegistry configured from environment variables.
 *
 * Call this once during app bootstrap (CLI, Gateway, Desktop embedded server).
 * Subsequent calls return the same instance.
 *
 * Pass 18 action: Replace core's ProviderRouter instantiation with a call
 * to this function, then delegate complete()/stream() calls through
 * registry.getRouter().route() and registry.getRouter().routeStream().
 *
 * @example
 * ```ts
 * // In core's bootstrap / AgentEngine constructor (Pass 18 change):
 * import { getOrCreateProviderRegistry } from './providers/bridge.js';
 *
 * const registry = getOrCreateProviderRegistry();
 * const router = registry.getRouter();
 *
 * // Replace: await this.providerRouter.complete(options)
 * // With:    await router.route(options)
 *
 * // Replace: this.providerRouter.stream(options)
 * // With:    router.routeStream(options)
 * ```
 */
export function getOrCreateProviderRegistry(): ProviderRegistry {
  if (!_registryInstance) {
    _registryInstance = ProviderRegistry.fromEnv();
  }
  return _registryInstance;
}

/**
 * Replace the singleton (useful in tests or when config changes at runtime).
 */
export function setProviderRegistry(registry: ProviderRegistry): void {
  _registryInstance?.destroy();
  _registryInstance = registry;
}

/**
 * Destroy and clear the singleton (call during app shutdown).
 */
export function destroyProviderRegistry(): void {
  _registryInstance?.destroy();
  _registryInstance = null;
}

// ── Pass 18 migration checklist (compiler-checked) ───────────────────────────
//
// The interface below documents exactly what core's existing ProviderRouter
// must expose for the bridge to work. If core's ProviderRouter already
// matches this, the Pass 18 migration is a drop-in. If not, the TypeScript
// compiler will flag the mismatch when Pass 18 wires it up.

import type { ProviderRequestOptions, ProviderResponse, StreamEvent } from '@locoworker/providers';

export interface ICoreProviderRouter {
  /**
   * Execute a non-streaming provider call.
   * Pass 18: delegate to ProviderRouter.route()
   */
  complete(options: ProviderRequestOptions): Promise<ProviderResponse>;

  /**
   * Execute a streaming provider call.
   * Pass 18: delegate to ProviderRouter.routeStream()
   */
  stream(options: ProviderRequestOptions): AsyncGenerator<StreamEvent, void, unknown>;
}
State after Pass 17 Part 1
Area	Status
packages/providers	✅ New package — fully scaffolded with production-grade types, adapters, routing, cost tracking, health monitoring, and registry
All cloud adapters (Anthropic, OpenAI-compat, Gemini)	✅ Real implementations with fetch, streaming, health check, listModels
All local adapters (Ollama, LM Studio, llama.cpp)	✅ Real implementations
Routing strategies (single / round-robin / least-cost / least-latency)	✅ All four implemented
Model capability catalog	✅ All major models catalogued with pricing + capability flags
Cost tracking (session + daily)	✅ SessionCostTracker + DailyCostTracker with cap enforcement
Health monitoring	✅ ProviderHealthMonitor with periodic checks + manual override
Core wiring point	✅ bridge.ts documents exactly what Pass 18 must change in core
Tests	✅ 3 test files covering registry, routing strategies, and cost tracking
Pass 17 Part 2 next	🔜 packages/tools-mcp (MCP client + tool registration) + openclaw/ placeholder



Pass 17 — Part 2: packages/tools-mcp + openclaw/
Scope of Part 2 (this pass):

packages/tools-mcp — full MCP (Model Context Protocol) client with stdio + HTTP/SSE transports, server registry, and bridge into packages/core's ToolRegistry
openclaw/ — root-level placeholder package with package.json, interfaces, and contracts; seeds the architecture for Pass 18 full implementation
Rule: Complete final files. Later pass wins. No merging required.

Dependency note: packages/tools-mcp depends on @locoworker/shared, @locoworker/core (for ToolRegistry types), and @locoworker/providers (for ProviderToolDefinition type alignment). openclaw/ has no internal deps in this scaffolding pass — it is intentionally self-contained.

File tree (Pass 17 Part 2 — files created/touched)
text

locoworker/
├── packages/
│   └── tools-mcp/                              ← NEW package
│       ├── package.json                        ← NEW
│       ├── tsconfig.json                       ← NEW
│       ├── vitest.config.ts                    ← NEW
│       └── src/
│           ├── index.ts                        ← NEW (barrel export)
│           ├── types.ts                        ← NEW (MCP protocol types)
│           ├── errors.ts                       ← NEW (typed MCP error hierarchy)
│           ├── client.ts                       ← NEW (McpClient — manages one server connection)
│           ├── registry.ts                     ← NEW (McpServerRegistry — manages many servers)
│           ├── bridge.ts                       ← NEW (bridges MCP tools → LocoWorker ToolRegistry)
│           ├── catalog.ts                      ← NEW (McpToolCatalog — merged view across servers)
│           ├── transport/
│           │   ├── index.ts                    ← NEW
│           │   ├── types.ts                    ← NEW (transport contract)
│           │   ├── stdio.ts                    ← NEW (StdioTransport)
│           │   └── sse.ts                      ← NEW (HttpSseTransport)
│           └── __tests__/
│               ├── client.test.ts              ← NEW
│               ├── bridge.test.ts              ← NEW
│               └── catalog.test.ts             ← NEW
└── openclaw/                                   ← NEW root-level package
    ├── package.json                            ← NEW
    ├── tsconfig.json                           ← NEW
    ├── README.md                               ← NEW
    └── src/
        ├── index.ts                            ← NEW (barrel + re-exports)
        ├── types.ts                            ← NEW (all interface contracts)
        ├── constants.ts                        ← NEW
        └── stubs/
            ├── index.ts                        ← NEW
            ├── analyzer.stub.ts                ← NEW (AnalyzerStub — throws NotImplemented)
            ├── scanner.stub.ts                 ← NEW (ScannerStub)
            └── reporter.stub.ts                ← NEW (ReporterStub)
packages/tools-mcp/package.json
JSON

{
  "name": "@locoworker/tools-mcp",
  "version": "0.1.0",
  "description": "LocoWorker MCP (Model Context Protocol) client + ToolRegistry bridge",
  "license": "MIT",
  "private": false,
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./transport": {
      "import": "./dist/transport/index.js",
      "types": "./dist/transport/index.d.ts"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "typecheck": "tsc --noEmit",
    "dev": "tsc --watch --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome check src",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@locoworker/shared": "workspace:*",
    "@locoworker/providers": "workspace:*"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "rimraf": "^6.0.1",
    "typescript": "^5.5.4",
    "vitest": "^2.0.5"
  },
  "peerDependencies": {
    "typescript": ">=5.5.0"
  }
}
packages/tools-mcp/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "composite": true,
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "src/**/__tests__/**"],
  "references": [
    { "path": "../shared" },
    { "path": "../providers" }
  ]
}
packages/tools-mcp/vitest.config.ts
TypeScript

import { defineConfig } from 'vitest/config';
import { resolve } from 'node:path';

export default defineConfig({
  test: {
    name: '@locoworker/tools-mcp',
    environment: 'node',
    globals: false,
    include: ['src/**/__tests__/**/*.test.ts'],
    testTimeout: 10_000,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['src/**/*.ts'],
      exclude: [
        'src/**/__tests__/**',
        'src/index.ts',
        'src/transport/index.ts',
      ],
    },
  },
  resolve: {
    alias: {
      '@locoworker/shared': resolve(__dirname, '../shared/src/index.ts'),
      '@locoworker/providers': resolve(__dirname, '../providers/src/index.ts'),
    },
  },
});
packages/tools-mcp/src/types.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/types.ts
// Pass 17 Part 2 — MCP protocol types
//
// Based on the Model Context Protocol specification (modelcontextprotocol.io).
// These types model the JSON-RPC 2.0 message layer and the MCP domain objects
// (tools, resources, prompts, sampling) that LocoWorker cares about.
//
// Design note: we do NOT depend on the official @modelcontextprotocol/sdk here.
// We use our own typed subset so this package has zero runtime npm deps beyond
// @locoworker/shared. Pass 18 can swap in the SDK if desired.
// ─────────────────────────────────────────────────────────────────────────────

// ── JSON-RPC 2.0 base types ───────────────────────────────────────────────────

export type JsonRpcId = string | number | null;

export interface JsonRpcRequest {
  readonly jsonrpc: '2.0';
  readonly id: JsonRpcId;
  readonly method: string;
  readonly params?: Record<string, unknown>;
}

export interface JsonRpcNotification {
  readonly jsonrpc: '2.0';
  readonly method: string;
  readonly params?: Record<string, unknown>;
}

export interface JsonRpcSuccessResponse {
  readonly jsonrpc: '2.0';
  readonly id: JsonRpcId;
  readonly result: unknown;
}

export interface JsonRpcErrorObject {
  readonly code: number;
  readonly message: string;
  readonly data?: unknown;
}

export interface JsonRpcErrorResponse {
  readonly jsonrpc: '2.0';
  readonly id: JsonRpcId;
  readonly error: JsonRpcErrorObject;
}

export type JsonRpcMessage =
  | JsonRpcRequest
  | JsonRpcNotification
  | JsonRpcSuccessResponse
  | JsonRpcErrorResponse;

// ── MCP server identity ───────────────────────────────────────────────────────

/** Unique string ID for an MCP server within a registry */
export type McpServerId = string;

export type McpTransportKind = 'stdio' | 'sse' | 'websocket';

/** Discriminated union config for each transport type */
export type McpServerTransportConfig =
  | {
      kind: 'stdio';
      /** Executable to spawn */
      command: string;
      /** Arguments to the executable */
      args?: readonly string[];
      /** Additional environment variables */
      env?: Record<string, string>;
      /** Working directory for the subprocess */
      cwd?: string;
    }
  | {
      kind: 'sse';
      /** HTTP endpoint exposing the MCP SSE stream */
      url: string;
      /** Optional auth headers */
      headers?: Record<string, string>;
    }
  | {
      kind: 'websocket';
      /** WebSocket URL */
      url: string;
      /** Optional auth headers (passed as sub-protocol or query param) */
      headers?: Record<string, string>;
    };

export interface McpServerConfig {
  /** Unique ID within the registry */
  readonly id: McpServerId;
  /** Human-readable name shown in logs / UI */
  readonly displayName: string;
  /** Transport configuration */
  readonly transport: McpServerTransportConfig;
  /** Whether this server is active */
  readonly enabled: boolean;
  /** Connection timeout in ms */
  readonly connectTimeoutMs: number;
  /** Request timeout for individual tool calls in ms */
  readonly requestTimeoutMs: number;
  /** Number of connection retries on transient failure */
  readonly maxRetries: number;
  /** Optional list of tool names to expose (allowlist). Empty = expose all. */
  readonly allowedTools?: readonly string[];
  /** Tool names to hide even if the server exposes them (denylist) */
  readonly deniedTools?: readonly string[];
}

// ── MCP protocol objects ──────────────────────────────────────────────────────

export interface McpImplementationInfo {
  readonly name: string;
  readonly version: string;
}

export interface McpCapabilities {
  readonly tools?: Record<string, unknown>;
  readonly resources?: Record<string, unknown>;
  readonly prompts?: Record<string, unknown>;
  readonly logging?: Record<string, unknown>;
  readonly sampling?: Record<string, unknown>;
}

export interface McpInitializeResult {
  readonly protocolVersion: string;
  readonly serverInfo: McpImplementationInfo;
  readonly capabilities: McpCapabilities;
}

// ── Tool types ────────────────────────────────────────────────────────────────

export interface McpToolInputSchema {
  readonly type: 'object';
  readonly properties?: Record<string, unknown>;
  readonly required?: readonly string[];
  readonly [key: string]: unknown;
}

export interface McpTool {
  readonly name: string;
  readonly description?: string;
  readonly inputSchema: McpToolInputSchema;
}

export interface McpToolCallParams {
  readonly name: string;
  readonly arguments?: Record<string, unknown>;
}

export type McpToolContentItem =
  | { readonly type: 'text'; readonly text: string }
  | { readonly type: 'image'; readonly data: string; readonly mimeType: string }
  | { readonly type: 'resource'; readonly resource: McpResourceContent };

export interface McpToolCallResult {
  readonly content: readonly McpToolContentItem[];
  readonly isError?: boolean;
}

// ── Resource types ────────────────────────────────────────────────────────────

export interface McpResource {
  readonly uri: string;
  readonly name: string;
  readonly description?: string;
  readonly mimeType?: string;
}

export interface McpResourceContent {
  readonly uri: string;
  readonly mimeType?: string;
  readonly text?: string;
  readonly blob?: string;
}

// ── Prompt types ──────────────────────────────────────────────────────────────

export interface McpPromptArgument {
  readonly name: string;
  readonly description?: string;
  readonly required?: boolean;
}

export interface McpPrompt {
  readonly name: string;
  readonly description?: string;
  readonly arguments?: readonly McpPromptArgument[];
}

// ── Connection state ──────────────────────────────────────────────────────────

export type McpConnectionState =
  | 'disconnected'
  | 'connecting'
  | 'connected'
  | 'reconnecting'
  | 'failed';

export interface McpServerState {
  readonly serverId: McpServerId;
  readonly connectionState: McpConnectionState;
  readonly serverInfo?: McpImplementationInfo;
  readonly capabilities?: McpCapabilities;
  readonly tools: readonly McpTool[];
  readonly resources: readonly McpResource[];
  readonly prompts: readonly McpPrompt[];
  readonly connectedAt?: number;
  readonly lastError?: string;
}

// ── Catalog entry ─────────────────────────────────────────────────────────────

/** A tool as seen by LocoWorker after being bridged from an MCP server */
export interface McpBridgedTool {
  /** Globally unique name: `mcp__<serverId>__<toolName>` */
  readonly qualifiedName: string;
  /** Original tool name on the MCP server */
  readonly originalName: string;
  /** MCP server this tool came from */
  readonly serverId: McpServerId;
  /** Server display name (for user-facing descriptions) */
  readonly serverDisplayName: string;
  readonly tool: McpTool;
}

// ── Events ────────────────────────────────────────────────────────────────────

export type McpRegistryEvent =
  | { type: 'server_connected'; serverId: McpServerId; info: McpImplementationInfo }
  | { type: 'server_disconnected'; serverId: McpServerId; reason?: string }
  | { type: 'server_failed'; serverId: McpServerId; error: string }
  | { type: 'tools_updated'; serverId: McpServerId; tools: readonly McpTool[] }
  | { type: 'tool_called'; serverId: McpServerId; toolName: string; durationMs: number }
  | { type: 'tool_error'; serverId: McpServerId; toolName: string; error: string };

export type McpRegistryListener = (event: McpRegistryEvent) => void;
packages/tools-mcp/src/errors.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/errors.ts
// Pass 17 Part 2 — Typed MCP error hierarchy
// ─────────────────────────────────────────────────────────────────────────────

import type { McpServerId } from './types.js';

export type McpErrorCode =
  | 'connection_failed'
  | 'connection_timeout'
  | 'protocol_error'
  | 'tool_not_found'
  | 'tool_call_failed'
  | 'tool_call_timeout'
  | 'server_not_found'
  | 'server_not_connected'
  | 'server_capability_missing'
  | 'transport_closed'
  | 'initialize_failed'
  | 'unknown';

export class McpError extends Error {
  constructor(
    public readonly code: McpErrorCode,
    message: string,
    public readonly serverId?: McpServerId,
    public readonly retryable: boolean = false,
    public readonly cause?: unknown,
  ) {
    super(message);
    this.name = 'McpError';
  }
}

export class McpConnectionError extends McpError {
  constructor(serverId: McpServerId, message: string, retryable = true, cause?: unknown) {
    super('connection_failed', message, serverId, retryable, cause);
    this.name = 'McpConnectionError';
  }
}

export class McpToolNotFoundError extends McpError {
  constructor(serverId: McpServerId, toolName: string) {
    super('tool_not_found', `Tool "${toolName}" not found on server "${serverId}".`, serverId, false);
    this.name = 'McpToolNotFoundError';
  }
}

export class McpToolCallError extends McpError {
  constructor(serverId: McpServerId, toolName: string, message: string, cause?: unknown) {
    super('tool_call_failed', `Tool call "${toolName}" on "${serverId}" failed: ${message}`, serverId, false, cause);
    this.name = 'McpToolCallError';
  }
}

export class McpTimeoutError extends McpError {
  constructor(serverId: McpServerId, operationName: string, timeoutMs: number) {
    super(
      'tool_call_timeout',
      `MCP operation "${operationName}" on "${serverId}" timed out after ${timeoutMs}ms.`,
      serverId,
      true,
    );
    this.name = 'McpTimeoutError';
  }
}

export class McpServerNotConnectedError extends McpError {
  constructor(serverId: McpServerId) {
    super('server_not_connected', `MCP server "${serverId}" is not connected.`, serverId, true);
    this.name = 'McpServerNotConnectedError';
  }
}

export class McpProtocolError extends McpError {
  constructor(serverId: McpServerId, message: string, cause?: unknown) {
    super('protocol_error', `MCP protocol error on "${serverId}": ${message}`, serverId, false, cause);
    this.name = 'McpProtocolError';
  }
}
packages/tools-mcp/src/transport/types.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/transport/types.ts
// Pass 17 Part 2 — Transport contract
// ─────────────────────────────────────────────────────────────────────────────

import type { JsonRpcMessage } from '../types.js';

/**
 * A transport is the low-level message channel to an MCP server.
 * It handles bytes-in / bytes-out and framing.
 * The McpClient sits above it and handles JSON-RPC semantics.
 */
export interface ITransport {
  /**
   * Open the transport. Resolves when the channel is ready to send/receive.
   * Must be idempotent (calling on an already-open transport is a no-op).
   */
  connect(): Promise<void>;

  /**
   * Close the transport cleanly. Resolves when fully closed.
   */
  disconnect(): Promise<void>;

  /**
   * Send a JSON-RPC message. Rejects if the transport is not connected.
   */
  send(message: JsonRpcMessage): Promise<void>;

  /**
   * Register a handler for incoming messages.
   * Returns an unsubscribe function.
   */
  onMessage(handler: (message: JsonRpcMessage) => void): () => void;

  /**
   * Register a handler for transport-level errors.
   * Returns an unsubscribe function.
   */
  onError(handler: (error: Error) => void): () => void;

  /**
   * Register a handler for transport close events.
   * Returns an unsubscribe function.
   */
  onClose(handler: () => void): () => void;

  /** Whether the transport is currently open */
  readonly isConnected: boolean;
}
packages/tools-mcp/src/transport/stdio.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/transport/stdio.ts
// Pass 17 Part 2 — StdioTransport
//
// Spawns a child process and communicates via its stdin/stdout using
// newline-delimited JSON (NDJSON) framing — the standard MCP stdio transport.
// ─────────────────────────────────────────────────────────────────────────────

import { spawn, type ChildProcessWithoutNullStreams } from 'node:child_process';
import { createInterface } from 'node:readline';
import type { ITransport } from './types.js';
import type { JsonRpcMessage } from '../types.js';
import { McpConnectionError, McpProtocolError } from '../errors.js';
import type { McpServerId } from '../types.js';

export interface StdioTransportOptions {
  command: string;
  args?: readonly string[];
  env?: Record<string, string>;
  cwd?: string;
  connectTimeoutMs?: number;
}

export class StdioTransport implements ITransport {
  private child: ChildProcessWithoutNullStreams | null = null;
  private readonly messageHandlers: Array<(msg: JsonRpcMessage) => void> = [];
  private readonly errorHandlers: Array<(err: Error) => void> = [];
  private readonly closeHandlers: Array<() => void> = [];
  private _isConnected = false;

  constructor(
    private readonly serverId: McpServerId,
    private readonly opts: StdioTransportOptions,
  ) {}

  get isConnected(): boolean {
    return this._isConnected;
  }

  async connect(): Promise<void> {
    if (this._isConnected) return;

    return new Promise((resolve, reject) => {
      const timeoutMs = this.opts.connectTimeoutMs ?? 10_000;
      const timer = setTimeout(() => {
        reject(new McpConnectionError(this.serverId, `Stdio connect timed out after ${timeoutMs}ms.`, false));
        this.child?.kill();
      }, timeoutMs);

      try {
        this.child = spawn(this.opts.command, [...(this.opts.args ?? [])], {
          cwd: this.opts.cwd,
          env: { ...process.env, ...(this.opts.env ?? {}) },
          stdio: ['pipe', 'pipe', 'pipe'],
        });
      } catch (err) {
        clearTimeout(timer);
        reject(new McpConnectionError(this.serverId, `Failed to spawn "${this.opts.command}": ${String(err)}`, false, err));
        return;
      }

      this.child.on('error', (err) => {
        clearTimeout(timer);
        this._isConnected = false;
        this.emitError(err);
        reject(new McpConnectionError(this.serverId, err.message, false, err));
      });

      this.child.on('exit', (_code, signal) => {
        this._isConnected = false;
        this.emitClose();
      });

      // Set up readline for NDJSON framing on stdout
      const rl = createInterface({ input: this.child.stdout, crlfDelay: Infinity });

      rl.on('line', (line) => {
        const trimmed = line.trim();
        if (!trimmed) return;
        try {
          const message = JSON.parse(trimmed) as JsonRpcMessage;
          for (const h of this.messageHandlers) {
            try { h(message); } catch { /* handler errors never propagate */ }
          }
        } catch (err) {
          this.emitError(new McpProtocolError(this.serverId, `Failed to parse NDJSON line: ${trimmed}`, err));
        }
      });

      // Stderr → log but don't treat as error
      this.child.stderr?.on('data', (_data: Buffer) => {
        // stderr from MCP servers is informational only
      });

      // Process spawned successfully
      clearTimeout(timer);
      this._isConnected = true;
      resolve();
    });
  }

  async disconnect(): Promise<void> {
    if (!this.child || !this._isConnected) return;
    return new Promise((resolve) => {
      this.child!.once('exit', () => {
        this._isConnected = false;
        resolve();
      });
      this.child!.stdin.end();
      setTimeout(() => {
        this.child?.kill('SIGTERM');
        resolve();
      }, 3_000);
    });
  }

  async send(message: JsonRpcMessage): Promise<void> {
    if (!this._isConnected || !this.child) {
      throw new McpConnectionError(this.serverId, 'Transport not connected. Call connect() first.', true);
    }
    const line = JSON.stringify(message) + '\n';
    return new Promise((resolve, reject) => {
      this.child!.stdin.write(line, 'utf8', (err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }

  onMessage(handler: (message: JsonRpcMessage) => void): () => void {
    this.messageHandlers.push(handler);
    return () => {
      const idx = this.messageHandlers.indexOf(handler);
      if (idx !== -1) this.messageHandlers.splice(idx, 1);
    };
  }

  onError(handler: (error: Error) => void): () => void {
    this.errorHandlers.push(handler);
    return () => {
      const idx = this.errorHandlers.indexOf(handler);
      if (idx !== -1) this.errorHandlers.splice(idx, 1);
    };
  }

  onClose(handler: () => void): () => void {
    this.closeHandlers.push(handler);
    return () => {
      const idx = this.closeHandlers.indexOf(handler);
      if (idx !== -1) this.closeHandlers.splice(idx, 1);
    };
  }

  private emitError(err: Error): void {
    for (const h of this.errorHandlers) {
      try { h(err); } catch { /* never propagate */ }
    }
  }

  private emitClose(): void {
    for (const h of this.closeHandlers) {
      try { h(); } catch { /* never propagate */ }
    }
  }
}
packages/tools-mcp/src/transport/sse.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/transport/sse.ts
// Pass 17 Part 2 — HttpSseTransport
//
// Connects to an MCP server that exposes an HTTP/SSE endpoint.
// The client sends JSON-RPC requests via HTTP POST to the server's
// message endpoint, and receives responses/notifications via SSE.
//
// Protocol:
//   GET <sseUrl>           → SSE stream (server → client messages)
//   POST <postUrl>/<id>    → Client → server messages
//   The server sends a "endpoint" SSE event first with the POST URL.
// ─────────────────────────────────────────────────────────────────────────────

import type { ITransport } from './types.js';
import type { JsonRpcMessage } from '../types.js';
import { McpConnectionError, McpProtocolError } from '../errors.js';
import type { McpServerId } from '../types.js';

export interface HttpSseTransportOptions {
  url: string;
  headers?: Record<string, string>;
  connectTimeoutMs?: number;
  requestTimeoutMs?: number;
}

export class HttpSseTransport implements ITransport {
  private messageHandlers: Array<(msg: JsonRpcMessage) => void> = [];
  private errorHandlers: Array<(err: Error) => void> = [];
  private closeHandlers: Array<() => void> = [];
  private _isConnected = false;
  private abortController: AbortController | null = null;
  private postEndpoint: string | null = null;
  private sseReadingPromise: Promise<void> | null = null;

  constructor(
    private readonly serverId: McpServerId,
    private readonly opts: HttpSseTransportOptions,
  ) {}

  get isConnected(): boolean {
    return this._isConnected;
  }

  async connect(): Promise<void> {
    if (this._isConnected) return;

    this.abortController = new AbortController();
    const timeoutMs = this.opts.connectTimeoutMs ?? 10_000;

    // The SSE endpoint sends an initial "endpoint" event with the POST URL
    const postEndpointPromise = new Promise<string>((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new McpConnectionError(this.serverId, `SSE connect timed out after ${timeoutMs}ms.`, false));
        this.abortController?.abort();
      }, timeoutMs);

      this.sseReadingPromise = this.readSseStream(resolve, timer);
      this.sseReadingPromise.catch((err) => {
        clearTimeout(timer);
        reject(err);
      });
    });

    this.postEndpoint = await postEndpointPromise;
    this._isConnected = true;
  }

  private async readSseStream(
    onEndpoint: (url: string) => void,
    connectTimer: ReturnType<typeof setTimeout>,
  ): Promise<void> {
    let endpointResolved = false;

    try {
      const res = await fetch(this.opts.url, {
        headers: {
          Accept: 'text/event-stream',
          'Cache-Control': 'no-cache',
          ...(this.opts.headers ?? {}),
        },
        signal: this.abortController!.signal,
      });

      if (!res.ok || !res.body) {
        throw new McpConnectionError(
          this.serverId,
          `SSE connection failed: HTTP ${res.status}`,
          false,
        );
      }

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';
      let currentEvent = '';

      try {
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;

          buffer += decoder.decode(value, { stream: true });
          const lines = buffer.split('\n');
          buffer = lines.pop() ?? '';

          for (const line of lines) {
            if (line.startsWith('event:')) {
              currentEvent = line.slice(6).trim();
            } else if (line.startsWith('data:')) {
              const data = line.slice(5).trim();

              if (currentEvent === 'endpoint') {
                // Server provides the POST URL
                clearTimeout(connectTimer);
                endpointResolved = true;
                onEndpoint(data);
                currentEvent = '';
              } else {
                // All other SSE data events are JSON-RPC messages
                try {
                  const message = JSON.parse(data) as JsonRpcMessage;
                  for (const h of this.messageHandlers) {
                    try { h(message); } catch { /* never propagate */ }
                  }
                } catch (err) {
                  this.emitError(new McpProtocolError(this.serverId, `Failed to parse SSE data: ${data}`, err));
                }
                currentEvent = '';
              }
            } else if (line === '') {
              currentEvent = '';
            }
          }
        }
      } finally {
        reader.releaseLock();
      }
    } catch (err) {
      if (!endpointResolved) {
        clearTimeout(connectTimer);
      }
      if (err instanceof Error && err.name === 'AbortError') {
        this._isConnected = false;
        this.emitClose();
        return;
      }
      this._isConnected = false;
      const mcpErr = err instanceof McpConnectionError
        ? err
        : new McpConnectionError(this.serverId, String(err), true, err);
      this.emitError(mcpErr);
      this.emitClose();
    }
  }

  async disconnect(): Promise<void> {
    this.abortController?.abort();
    this._isConnected = false;
    this.postEndpoint = null;
    if (this.sseReadingPromise) {
      await this.sseReadingPromise.catch(() => {});
    }
  }

  async send(message: JsonRpcMessage): Promise<void> {
    if (!this._isConnected || !this.postEndpoint) {
      throw new McpConnectionError(this.serverId, 'SSE transport not connected.', true);
    }

    const res = await fetch(this.postEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.opts.headers ?? {}),
      },
      body: JSON.stringify(message),
      signal: AbortSignal.timeout(this.opts.requestTimeoutMs ?? 30_000),
    });

    if (!res.ok) {
      throw new McpProtocolError(
        this.serverId,
        `HTTP POST to MCP server failed: ${res.status}`,
      );
    }
  }

  onMessage(handler: (message: JsonRpcMessage) => void): () => void {
    this.messageHandlers.push(handler);
    return () => {
      const idx = this.messageHandlers.indexOf(handler);
      if (idx !== -1) this.messageHandlers.splice(idx, 1);
    };
  }

  onError(handler: (error: Error) => void): () => void {
    this.errorHandlers.push(handler);
    return () => {
      const idx = this.errorHandlers.indexOf(handler);
      if (idx !== -1) this.errorHandlers.splice(idx, 1);
    };
  }

  onClose(handler: () => void): () => void {
    this.closeHandlers.push(handler);
    return () => {
      const idx = this.closeHandlers.indexOf(handler);
      if (idx !== -1) this.closeHandlers.splice(idx, 1);
    };
  }

  private emitError(err: Error): void {
    for (const h of this.errorHandlers) {
      try { h(err); } catch { /* never propagate */ }
    }
  }

  private emitClose(): void {
    for (const h of this.closeHandlers) {
      try { h(); } catch { /* never propagate */ }
    }
  }
}
packages/tools-mcp/src/transport/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/transport/index.ts
// Pass 17 Part 2
// ─────────────────────────────────────────────────────────────────────────────

export type { ITransport } from './types.js';
export { StdioTransport } from './stdio.js';
export type { StdioTransportOptions } from './stdio.js';
export { HttpSseTransport } from './sse.js';
export type { HttpSseTransportOptions } from './sse.js';
packages/tools-mcp/src/client.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/client.ts
// Pass 17 Part 2 — McpClient
//
// McpClient manages a single connection to one MCP server.
// It handles:
//   - The JSON-RPC 2.0 request/response correlation (pending request map)
//   - The MCP initialize handshake
//   - tools/list, tools/call, resources/list, prompts/list
//   - Automatic reconnection on transport close
// ─────────────────────────────────────────────────────────────────────────────

import type { ITransport } from './transport/types.js';
import { StdioTransport } from './transport/stdio.js';
import { HttpSseTransport } from './transport/sse.js';
import type {
  McpServerId,
  McpServerConfig,
  McpConnectionState,
  McpServerState,
  McpTool,
  McpResource,
  McpPrompt,
  McpToolCallParams,
  McpToolCallResult,
  McpInitializeResult,
  JsonRpcMessage,
  JsonRpcRequest,
  JsonRpcSuccessResponse,
  JsonRpcErrorResponse,
} from './types.js';
import {
  McpConnectionError,
  McpToolNotFoundError,
  McpToolCallError,
  McpTimeoutError,
  McpServerNotConnectedError,
  McpProtocolError,
} from './errors.js';

const MCP_PROTOCOL_VERSION = '2024-11-05';
const CLIENT_INFO = { name: 'locoworker', version: '0.1.0' };

interface PendingRequest {
  resolve: (result: unknown) => void;
  reject: (err: Error) => void;
  timeoutHandle: ReturnType<typeof setTimeout>;
}

export class McpClient {
  private transport: ITransport | null = null;
  private _state: McpConnectionState = 'disconnected';
  private pendingRequests = new Map<string | number, PendingRequest>();
  private requestIdCounter = 0;
  private serverInfo: McpInitializeResult | null = null;
  private _tools: McpTool[] = [];
  private _resources: McpResource[] = [];
  private _prompts: McpPrompt[] = [];
  private stateListeners: Array<(state: McpConnectionState) => void> = [];
  private reconnectAttempts = 0;

  constructor(
    public readonly serverId: McpServerId,
    private readonly config: McpServerConfig,
  ) {}

  // ── State ───────────────────────────────────────────────────────────────────

  get state(): McpConnectionState {
    return this._state;
  }

  get tools(): readonly McpTool[] {
    return this._tools;
  }

  get resources(): readonly McpResource[] {
    return this._resources;
  }

  get prompts(): readonly McpPrompt[] {
    return this._prompts;
  }

  getServerState(): McpServerState {
    return {
      serverId: this.serverId,
      connectionState: this._state,
      serverInfo: this.serverInfo?.serverInfo,
      capabilities: this.serverInfo?.capabilities,
      tools: this._tools,
      resources: this._resources,
      prompts: this._prompts,
      connectedAt: this._state === 'connected' ? Date.now() : undefined,
    };
  }

  onStateChange(listener: (state: McpConnectionState) => void): () => void {
    this.stateListeners.push(listener);
    return () => {
      const idx = this.stateListeners.indexOf(listener);
      if (idx !== -1) this.stateListeners.splice(idx, 1);
    };
  }

  // ── Lifecycle ───────────────────────────────────────────────────────────────

  async connect(): Promise<void> {
    if (this._state === 'connected' || this._state === 'connecting') return;
    this.setState('connecting');

    this.transport = this.createTransport();

    this.transport.onMessage((msg) => this.handleMessage(msg));
    this.transport.onError((err) => this.handleTransportError(err));
    this.transport.onClose(() => this.handleTransportClose());

    try {
      await this.transport.connect();
      await this.initialize();
      await this.loadCapabilities();
      this.setState('connected');
      this.reconnectAttempts = 0;
    } catch (err) {
      this.setState('failed');
      const mcpErr = err instanceof Error ? err : new McpConnectionError(this.serverId, String(err));
      throw mcpErr;
    }
  }

  async disconnect(): Promise<void> {
    this.cancelAllPendingRequests('Client disconnected.');
    await this.transport?.disconnect();
    this.transport = null;
    this._tools = [];
    this._resources = [];
    this._prompts = [];
    this.serverInfo = null;
    this.setState('disconnected');
  }

  // ── MCP operations ──────────────────────────────────────────────────────────

  /**
   * Call an MCP tool by name. The caller is responsible for passing
   * the original tool name (not the qualified `mcp__<id>__<name>` form).
   */
  async callTool(toolName: string, args: Record<string, unknown> = {}): Promise<McpToolCallResult> {
    this.assertConnected();

    const tool = this._tools.find((t) => t.name === toolName);
    if (!tool) throw new McpToolNotFoundError(this.serverId, toolName);

    const start = performance.now();
    try {
      const result = await this.request<McpToolCallResult>('tools/call', {
        name: toolName,
        arguments: args,
      } satisfies McpToolCallParams);
      return result;
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      throw new McpToolCallError(this.serverId, toolName, msg, err);
    }
  }

  async refreshTools(): Promise<readonly McpTool[]> {
    this.assertConnected();
    const result = await this.request<{ tools: McpTool[] }>('tools/list', {});
    this._tools = result.tools ?? [];
    return this._tools;
  }

  async refreshResources(): Promise<readonly McpResource[]> {
    if (!this.serverInfo?.capabilities?.resources) return [];
    this.assertConnected();
    const result = await this.request<{ resources: McpResource[] }>('resources/list', {});
    this._resources = result.resources ?? [];
    return this._resources;
  }

  async refreshPrompts(): Promise<readonly McpPrompt[]> {
    if (!this.serverInfo?.capabilities?.prompts) return [];
    this.assertConnected();
    const result = await this.request<{ prompts: McpPrompt[] }>('prompts/list', {});
    this._prompts = result.prompts ?? [];
    return this._prompts;
  }

  // ── Internal: JSON-RPC ──────────────────────────────────────────────────────

  private request<T>(method: string, params: Record<string, unknown>): Promise<T> {
    this.assertConnected();

    const id = ++this.requestIdCounter;
    const timeoutMs = this.config.requestTimeoutMs;

    return new Promise<T>((resolve, reject) => {
      const timeoutHandle = setTimeout(() => {
        this.pendingRequests.delete(id);
        reject(new McpTimeoutError(this.serverId, method, timeoutMs));
      }, timeoutMs);

      this.pendingRequests.set(id, {
        resolve: resolve as (r: unknown) => void,
        reject,
        timeoutHandle,
      });

      const message: JsonRpcRequest = {
        jsonrpc: '2.0',
        id,
        method,
        params,
      };

      this.transport!.send(message).catch((err) => {
        this.pendingRequests.delete(id);
        clearTimeout(timeoutHandle);
        reject(err);
      });
    });
  }

  private notify(method: string, params: Record<string, unknown>): void {
    if (!this.transport?.isConnected) return;
    this.transport.send({ jsonrpc: '2.0', method, params }).catch(() => {});
  }

  private handleMessage(msg: JsonRpcMessage): void {
    // Response to a pending request
    if ('id' in msg && msg.id !== null) {
      const pending = this.pendingRequests.get(msg.id);
      if (!pending) return;

      this.pendingRequests.delete(msg.id);
      clearTimeout(pending.timeoutHandle);

      if ('error' in msg) {
        const errMsg = msg as JsonRpcErrorResponse;
        pending.reject(new McpProtocolError(this.serverId, errMsg.error.message));
      } else {
        const successMsg = msg as JsonRpcSuccessResponse;
        pending.resolve(successMsg.result);
      }
      return;
    }

    // Notification from server (no id)
    if ('method' in msg && !('id' in msg)) {
      this.handleNotification(msg.method as string, msg.params ?? {});
    }
  }

  private handleNotification(method: string, _params: Record<string, unknown>): void {
    switch (method) {
      case 'notifications/tools/list_changed':
        void this.refreshTools();
        break;
      case 'notifications/resources/list_changed':
        void this.refreshResources();
        break;
      case 'notifications/prompts/list_changed':
        void this.refreshPrompts();
        break;
      default:
        break;
    }
  }

  private handleTransportError(_err: Error): void {
    // Transport errors are surfaced but do not immediately change state
    // (the close event is what triggers reconnection)
  }

  private handleTransportClose(): void {
    this.cancelAllPendingRequests('Transport closed.');
    if (this._state !== 'disconnected') {
      void this.scheduleReconnect();
    }
  }

  private async scheduleReconnect(): Promise<void> {
    if (this.reconnectAttempts >= this.config.maxRetries) {
      this.setState('failed');
      return;
    }

    this.setState('reconnecting');
    this.reconnectAttempts++;
    const delay = Math.min(1000 * 2 ** this.reconnectAttempts, 30_000);

    await new Promise((r) => setTimeout(r, delay));

    try {
      await this.connect();
    } catch {
      // scheduleReconnect will be called again by the close handler
    }
  }

  private cancelAllPendingRequests(reason: string): void {
    for (const [id, pending] of this.pendingRequests) {
      clearTimeout(pending.timeoutHandle);
      pending.reject(new McpServerNotConnectedError(this.serverId));
    }
    this.pendingRequests.clear();
  }

  // ── Internal: MCP initialize handshake ─────────────────────────────────────

  private async initialize(): Promise<void> {
    const result = await this.request<McpInitializeResult>('initialize', {
      protocolVersion: MCP_PROTOCOL_VERSION,
      clientInfo: CLIENT_INFO,
      capabilities: {
        roots: { listChanged: true },
        sampling: {},
      },
    });

    this.serverInfo = result;

    // Send the initialized notification (required by protocol)
    this.notify('notifications/initialized', {});
  }

  private async loadCapabilities(): Promise<void> {
    await this.refreshTools();

    if (this.serverInfo?.capabilities?.resources) {
      await this.refreshResources().catch(() => {});
    }
    if (this.serverInfo?.capabilities?.prompts) {
      await this.refreshPrompts().catch(() => {});
    }
  }

  // ── Internal helpers ────────────────────────────────────────────────────────

  private createTransport(): ITransport {
    const tc = this.config.transport;
    switch (tc.kind) {
      case 'stdio':
        return new StdioTransport(this.serverId, {
          command: tc.command,
          args: tc.args,
          env: tc.env,
          cwd: tc.cwd,
          connectTimeoutMs: this.config.connectTimeoutMs,
        });
      case 'sse':
        return new HttpSseTransport(this.serverId, {
          url: tc.url,
          headers: tc.headers,
          connectTimeoutMs: this.config.connectTimeoutMs,
          requestTimeoutMs: this.config.requestTimeoutMs,
        });
      case 'websocket':
        throw new McpConnectionError(
          this.serverId,
          'WebSocket transport is not yet implemented. Use stdio or sse.',
          false,
        );
      default: {
        const _never: never = tc;
        throw new McpConnectionError(this.serverId, `Unknown transport kind.`, false);
      }
    }
  }

  private assertConnected(): void {
    if (this._state !== 'connected') {
      throw new McpServerNotConnectedError(this.serverId);
    }
  }

  private setState(next: McpConnectionState): void {
    this._state = next;
    for (const l of this.stateListeners) {
      try { l(next); } catch { /* never propagate */ }
    }
  }
}
packages/tools-mcp/src/catalog.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/catalog.ts
// Pass 17 Part 2 — McpToolCatalog
//
// Merges tool lists from all connected MCP servers into a single view.
// Handles name qualification so each tool has a globally unique name
// in the format: mcp__<serverId>__<toolName>
// ─────────────────────────────────────────────────────────────────────────────

import type { McpServerId, McpTool, McpBridgedTool } from './types.js';

export class McpToolCatalog {
  private readonly tools = new Map<string, McpBridgedTool>();

  /**
   * Register / replace tools from a server.
   * Call this after initial connect and after tools/list_changed notifications.
   */
  updateServer(
    serverId: McpServerId,
    serverDisplayName: string,
    tools: readonly McpTool[],
    allowedTools?: readonly string[],
    deniedTools?: readonly string[],
  ): void {
    // Remove all tools previously from this server
    for (const [key, entry] of this.tools) {
      if (entry.serverId === serverId) this.tools.delete(key);
    }

    for (const tool of tools) {
      if (allowedTools?.length && !allowedTools.includes(tool.name)) continue;
      if (deniedTools?.length && deniedTools.includes(tool.name)) continue;

      const qualified = this.qualify(serverId, tool.name);
      this.tools.set(qualified, {
        qualifiedName: qualified,
        originalName: tool.name,
        serverId,
        serverDisplayName,
        tool,
      });
    }
  }

  removeServer(serverId: McpServerId): void {
    for (const [key, entry] of this.tools) {
      if (entry.serverId === serverId) this.tools.delete(key);
    }
  }

  getByQualifiedName(qualifiedName: string): McpBridgedTool | undefined {
    return this.tools.get(qualifiedName);
  }

  getByServer(serverId: McpServerId): readonly McpBridgedTool[] {
    return [...this.tools.values()].filter((t) => t.serverId === serverId);
  }

  all(): readonly McpBridgedTool[] {
    return [...this.tools.values()];
  }

  has(qualifiedName: string): boolean {
    return this.tools.has(qualifiedName);
  }

  size(): number {
    return this.tools.size;
  }

  /**
   * Produce a stable qualified name for a tool.
   * Format: `mcp__<serverId>__<toolName>`
   * Both serverId and toolName are slug-sanitised (non-alphanumeric → _).
   */
  qualify(serverId: McpServerId, toolName: string): string {
    const safeId = serverId.replace(/[^a-zA-Z0-9]/g, '_');
    const safeName = toolName.replace(/[^a-zA-Z0-9]/g, '_');
    return `mcp__${safeId}__${safeName}`;
  }

  /**
   * Parse a qualified name back into its parts.
   * Returns null if the name is not a valid MCP qualified name.
   */
  parseQualifiedName(
    qualifiedName: string,
  ): { serverId: string; toolName: string } | null {
    if (!qualifiedName.startsWith('mcp__')) return null;
    const rest = qualifiedName.slice('mcp__'.length);
    const secondDunder = rest.indexOf('__');
    if (secondDunder === -1) return null;
    return {
      serverId: rest.slice(0, secondDunder),
      toolName: rest.slice(secondDunder + 2),
    };
  }
}
packages/tools-mcp/src/registry.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/registry.ts
// Pass 17 Part 2 — McpServerRegistry
//
// Top-level entry point for the tools-mcp package.
// Owns all McpClient instances, the tool catalog, and event emission.
// ─────────────────────────────────────────────────────────────────────────────

import { McpClient } from './client.js';
import { McpToolCatalog } from './catalog.js';
import type {
  McpServerId,
  McpServerConfig,
  McpServerState,
  McpTool,
  McpToolCallResult,
  McpBridgedTool,
  McpRegistryEvent,
  McpRegistryListener,
} from './types.js';
import { McpError } from './errors.js';

export interface McpServerRegistryOptions {
  /** Auto-connect all registered servers on startup */
  autoConnect?: boolean;
}

export class McpServerRegistry {
  private readonly clients = new Map<McpServerId, McpClient>();
  private readonly catalog = new McpToolCatalog();
  private readonly listeners: McpRegistryListener[] = [];

  constructor(private readonly opts: McpServerRegistryOptions = {}) {}

  // ── Server management ───────────────────────────────────────────────────────

  async registerServer(config: McpServerConfig): Promise<void> {
    if (this.clients.has(config.id)) {
      throw new McpError('server_not_found', `Server "${config.id}" is already registered.`);
    }

    const client = new McpClient(config.id, config);

    client.onStateChange((state) => {
      if (state === 'connected') {
        this.catalog.updateServer(
          config.id,
          config.displayName,
          client.tools,
          config.allowedTools ? [...config.allowedTools] : undefined,
          config.deniedTools ? [...config.deniedTools] : undefined,
        );
        this.emit({
          type: 'server_connected',
          serverId: config.id,
          info: client.getServerState().serverInfo ?? { name: config.id, version: 'unknown' },
        });
        this.emit({
          type: 'tools_updated',
          serverId: config.id,
          tools: client.tools as McpTool[],
        });
      } else if (state === 'disconnected') {
        this.catalog.removeServer(config.id);
        this.emit({ type: 'server_disconnected', serverId: config.id });
      } else if (state === 'failed') {
        this.catalog.removeServer(config.id);
        this.emit({
          type: 'server_failed',
          serverId: config.id,
          error: 'Connection failed after max retries.',
        });
      }
    });

    this.clients.set(config.id, client);

    if (this.opts.autoConnect !== false) {
      await client.connect().catch((err) => {
        this.emit({
          type: 'server_failed',
          serverId: config.id,
          error: err instanceof Error ? err.message : String(err),
        });
      });
    }
  }

  async unregisterServer(serverId: McpServerId): Promise<void> {
    const client = this.clients.get(serverId);
    if (!client) return;
    await client.disconnect();
    this.clients.delete(serverId);
    this.catalog.removeServer(serverId);
    this.emit({ type: 'server_disconnected', serverId });
  }

  async connectServer(serverId: McpServerId): Promise<void> {
    const client = this.requireClient(serverId);
    await client.connect();
  }

  async disconnectServer(serverId: McpServerId): Promise<void> {
    const client = this.requireClient(serverId);
    await client.disconnect();
  }

  getServerState(serverId: McpServerId): McpServerState | undefined {
    return this.clients.get(serverId)?.getServerState();
  }

  listServers(): readonly McpServerState[] {
    return [...this.clients.values()].map((c) => c.getServerState());
  }

  // ── Tool operations ─────────────────────────────────────────────────────────

  /**
   * Call an MCP tool by its qualified name (mcp__<serverId>__<toolName>).
   */
  async callTool(
    qualifiedName: string,
    args: Record<string, unknown> = {},
  ): Promise<McpToolCallResult> {
    const bridged = this.catalog.getByQualifiedName(qualifiedName);
    if (!bridged) {
      throw new McpError('tool_not_found', `MCP tool "${qualifiedName}" not found in catalog.`);
    }

    const client = this.requireClient(bridged.serverId);
    const start = performance.now();

    try {
      const result = await client.callTool(bridged.originalName, args);
      const durationMs = Math.round(performance.now() - start);
      this.emit({ type: 'tool_called', serverId: bridged.serverId, toolName: bridged.originalName, durationMs });
      return result;
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      this.emit({ type: 'tool_error', serverId: bridged.serverId, toolName: bridged.originalName, error: msg });
      throw err;
    }
  }

  getCatalog(): McpToolCatalog {
    return this.catalog;
  }

  listBridgedTools(): readonly McpBridgedTool[] {
    return this.catalog.all();
  }

  getBridgedTool(qualifiedName: string): McpBridgedTool | undefined {
    return this.catalog.getByQualifiedName(qualifiedName);
  }

  // ── Events ──────────────────────────────────────────────────────────────────

  on(listener: McpRegistryListener): () => void {
    this.listeners.push(listener);
    return () => {
      const idx = this.listeners.indexOf(listener);
      if (idx !== -1) this.listeners.splice(idx, 1);
    };
  }

  private emit(event: McpRegistryEvent): void {
    for (const l of this.listeners) {
      try { l(event); } catch { /* never propagate */ }
    }
  }

  // ── Lifecycle ───────────────────────────────────────────────────────────────

  async connectAll(): Promise<void> {
    await Promise.all([...this.clients.values()].map((c) => c.connect().catch(() => {})));
  }

  async disconnectAll(): Promise<void> {
    await Promise.all([...this.clients.values()].map((c) => c.disconnect()));
  }

  /**
   * Build a registry from an array of server configs.
   * Connects all enabled servers.
   */
  static async fromConfigs(configs: readonly McpServerConfig[]): Promise<McpServerRegistry> {
    const registry = new McpServerRegistry({ autoConnect: false });
    for (const config of configs) {
      if (!config.enabled) continue;
      await registry.registerServer(config);
    }
    await registry.connectAll();
    return registry;
  }

  // ── Private helpers ─────────────────────────────────────────────────────────

  private requireClient(serverId: McpServerId): McpClient {
    const client = this.clients.get(serverId);
    if (!client) throw new McpError('server_not_found', `MCP server "${serverId}" is not registered.`);
    return client;
  }
}
packages/tools-mcp/src/bridge.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/bridge.ts
// Pass 17 Part 2 — McpToolRegistryBridge
//
// Bridges McpServerRegistry → LocoWorker ToolRegistry (packages/core).
//
// Design:
//   - Converts each MCP tool into a LocoWorker ToolDefinition with the
//     qualified name as the tool name.
//   - Registers / deregisters tools dynamically as MCP servers connect /
//     disconnect (via registry event listeners).
//   - Converts MCP tool call results into LocoWorker ToolCallResult format.
//
// Why not depend on @locoworker/core directly:
//   packages/tools-mcp must not create a circular dependency with core.
//   Instead it accepts the ToolRegistry-compatible interface as a parameter
//   (dependency injection). Core's bootstrap wires them together.
//
// Pass 18 action: In apps/cli and apps/gateway bootstrap files, replace
// the "TODO: wire MCP bridge" comment with:
//
//   import { McpToolRegistryBridge } from '@locoworker/tools-mcp';
//   const mcpBridge = new McpToolRegistryBridge(mcpRegistry, toolRegistry);
//   await mcpBridge.attach();
// ─────────────────────────────────────────────────────────────────────────────

import type { McpServerRegistry } from './registry.js';
import type { McpBridgedTool, McpToolCallResult } from './types.js';

// ── Minimal ToolRegistry interface (matches packages/core's IToolRegistry) ───
// We use a structural type here rather than importing from @locoworker/core
// to avoid a circular dependency. If core's ToolRegistry interface changes,
// the TypeScript compiler will catch the mismatch at the bridge call sites.

export interface BridgeToolDefinition {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  handler: (args: Record<string, unknown>) => Promise<BridgeToolResult>;
  /** Permission tier required (maps to core's permission tiers) */
  permissionTier?: string;
  /** Max execution time in ms */
  timeoutMs?: number;
  /** Whether multiple instances can run in parallel */
  parallelSafe?: boolean;
  /** Tag to identify bridge-registered tools */
  tags?: string[];
}

export interface BridgeToolResult {
  success: boolean;
  output: string;
  error?: string;
  metadata?: Record<string, unknown>;
}

export interface IBridgeToolRegistry {
  register(tool: BridgeToolDefinition): void;
  unregister(toolName: string): void;
}

// ── Bridge ────────────────────────────────────────────────────────────────────

export class McpToolRegistryBridge {
  private unsubscribe: (() => void) | null = null;
  private readonly registeredToolNames = new Set<string>();

  constructor(
    private readonly mcpRegistry: McpServerRegistry,
    private readonly toolRegistry: IBridgeToolRegistry,
  ) {}

  /**
   * Start listening to MCP registry events and populate the ToolRegistry.
   * Immediately registers all currently available tools.
   */
  attach(): void {
    if (this.unsubscribe) return; // already attached

    // Register all currently available tools
    for (const bridged of this.mcpRegistry.listBridgedTools()) {
      this.registerBridgedTool(bridged);
    }

    // Subscribe to future changes
    this.unsubscribe = this.mcpRegistry.on((event) => {
      switch (event.type) {
        case 'tools_updated': {
          // Remove old tools from this server, add new ones
          const catalog = this.mcpRegistry.getCatalog();
          const serverTools = catalog.getByServer(event.serverId);

          // Unregister previous tools from this server
          for (const name of [...this.registeredToolNames]) {
            if (name.startsWith(`mcp__${event.serverId}__`)) {
              this.toolRegistry.unregister(name);
              this.registeredToolNames.delete(name);
            }
          }

          // Register new tools from this server
          for (const bridged of serverTools) {
            this.registerBridgedTool(bridged);
          }
          break;
        }

        case 'server_disconnected':
        case 'server_failed': {
          // Remove all tools from this server
          for (const name of [...this.registeredToolNames]) {
            if (name.startsWith(`mcp__${event.serverId}__`)) {
              this.toolRegistry.unregister(name);
              this.registeredToolNames.delete(name);
            }
          }
          break;
        }

        default:
          break;
      }
    });
  }

  /**
   * Stop listening and unregister all MCP tools from the ToolRegistry.
   */
  detach(): void {
    this.unsubscribe?.();
    this.unsubscribe = null;

    for (const name of this.registeredToolNames) {
      this.toolRegistry.unregister(name);
    }
    this.registeredToolNames.clear();
  }

  // ── Private helpers ─────────────────────────────────────────────────────────

  private registerBridgedTool(bridged: McpBridgedTool): void {
    const toolDef: BridgeToolDefinition = {
      name: bridged.qualifiedName,
      description: this.buildDescription(bridged),
      inputSchema: bridged.tool.inputSchema as Record<string, unknown>,
      timeoutMs: 60_000,
      parallelSafe: true,
      permissionTier: 'read_write',
      tags: ['mcp', bridged.serverId],
      handler: async (args) => {
        try {
          const result = await this.mcpRegistry.callTool(bridged.qualifiedName, args);
          return this.convertResult(result);
        } catch (err) {
          return {
            success: false,
            output: '',
            error: err instanceof Error ? err.message : String(err),
          };
        }
      },
    };

    this.toolRegistry.register(toolDef);
    this.registeredToolNames.add(bridged.qualifiedName);
  }

  private buildDescription(bridged: McpBridgedTool): string {
    const base = bridged.tool.description ?? `Tool "${bridged.originalName}" from MCP server.`;
    return `[MCP: ${bridged.serverDisplayName}] ${base}`;
  }

  private convertResult(result: McpToolCallResult): BridgeToolResult {
    if (result.isError) {
      const errorText = result.content
        .filter((c) => c.type === 'text')
        .map((c) => (c as { type: 'text'; text: string }).text)
        .join('\n');
      return { success: false, output: '', error: errorText };
    }

    const output = result.content
      .map((c) => {
        if (c.type === 'text') return (c as { type: 'text'; text: string }).text;
        if (c.type === 'image') return `[image: ${(c as { type: 'image'; mimeType: string }).mimeType}]`;
        return '[resource]';
      })
      .join('\n');

    return { success: true, output };
  }
}

// ── Factory helpers ───────────────────────────────────────────────────────────

/**
 * Build an McpServerRegistry from environment-variable-based config.
 *
 * Reads LOCOWORKER_MCP_SERVERS as a JSON array of McpServerConfig objects.
 * Falls back to empty registry if not set.
 *
 * @example .env
 *   LOCOWORKER_MCP_SERVERS='[{"id":"filesystem","displayName":"Filesystem","transport":{"kind":"stdio","command":"npx","args":["@modelcontextprotocol/server-filesystem","/tmp"]},"enabled":true,"connectTimeoutMs":10000,"requestTimeoutMs":30000,"maxRetries":3}]'
 */
export async function createMcpRegistryFromEnv(
  McpServerRegistryClass: typeof import('./registry.js').McpServerRegistry,
  McpServerConfigSchema: (raw: unknown) => import('./types.js').McpServerConfig,
): Promise<import('./registry.js').McpServerRegistry> {
  const raw = process.env['LOCOWORKER_MCP_SERVERS'];
  if (!raw) {
    return new McpServerRegistryClass({ autoConnect: false });
  }

  let configs: unknown[];
  try {
    configs = JSON.parse(raw) as unknown[];
  } catch {
    console.warn('[tools-mcp] LOCOWORKER_MCP_SERVERS is not valid JSON. Skipping MCP setup.');
    return new McpServerRegistryClass({ autoConnect: false });
  }

  const validConfigs = configs
    .map((c) => {
      try {
        return McpServerConfigSchema(c);
      } catch {
        return null;
      }
    })
    .filter((c): c is import('./types.js').McpServerConfig => c !== null);

  return McpServerRegistryClass.fromConfigs(validConfigs);
}
packages/tools-mcp/src/__tests__/client.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/__tests__/client.test.ts
// Pass 17 Part 2
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { McpClient } from '../client.js';
import type { McpServerConfig } from '../types.js';
import { McpServerNotConnectedError, McpToolNotFoundError } from '../errors.js';

// ── Helpers ───────────────────────────────────────────────────────────────────

function makeConfig(overrides: Partial<McpServerConfig> = {}): McpServerConfig {
  return {
    id: 'test-server',
    displayName: 'Test MCP Server',
    transport: { kind: 'stdio', command: 'echo', args: [] },
    enabled: true,
    connectTimeoutMs: 5_000,
    requestTimeoutMs: 10_000,
    maxRetries: 1,
    ...overrides,
  };
}

describe('McpClient — initial state', () => {
  it('starts in disconnected state', () => {
    const client = new McpClient('srv-1', makeConfig());
    expect(client.state).toBe('disconnected');
  });

  it('has empty tool/resource/prompt lists before connect', () => {
    const client = new McpClient('srv-1', makeConfig());
    expect(client.tools).toHaveLength(0);
    expect(client.resources).toHaveLength(0);
    expect(client.prompts).toHaveLength(0);
  });

  it('throws McpServerNotConnectedError on callTool when disconnected', async () => {
    const client = new McpClient('srv-1', makeConfig());
    await expect(client.callTool('my_tool', {})).rejects.toBeInstanceOf(McpServerNotConnectedError);
  });

  it('throws McpServerNotConnectedError on refreshTools when disconnected', async () => {
    const client = new McpClient('srv-1', makeConfig());
    await expect(client.refreshTools()).rejects.toBeInstanceOf(McpServerNotConnectedError);
  });
});

describe('McpClient — state listeners', () => {
  it('fires state listener when state changes', () => {
    const client = new McpClient('srv-1', makeConfig());
    const states: string[] = [];
    client.onStateChange((s) => states.push(s));

    // Access private setState via any
    (client as unknown as { setState: (s: string) => void }).setState('connecting');
    (client as unknown as { setState: (s: string) => void }).setState('connected');

    expect(states).toEqual(['connecting', 'connected']);
  });

  it('unsubscribes state listener', () => {
    const client = new McpClient('srv-1', makeConfig());
    const states: string[] = [];
    const unsub = client.onStateChange((s) => states.push(s));

    (client as unknown as { setState: (s: string) => void }).setState('connecting');
    unsub();
    (client as unknown as { setState: (s: string) => void }).setState('connected');

    expect(states).toEqual(['connecting']);
  });
});

describe('McpClient — getServerState', () => {
  it('returns correct shape', () => {
    const client = new McpClient('srv-1', makeConfig());
    const state = client.getServerState();
    expect(state.serverId).toBe('srv-1');
    expect(state.connectionState).toBe('disconnected');
    expect(state.tools).toHaveLength(0);
  });
});

describe('McpClient — transport creation', () => {
  it('throws for websocket transport (not yet implemented)', async () => {
    const client = new McpClient('srv-1', makeConfig({
      transport: { kind: 'websocket', url: 'ws://localhost:9000' },
    }));
    await expect(client.connect()).rejects.toThrow(/WebSocket transport is not yet implemented/i);
  });
});
packages/tools-mcp/src/__tests__/catalog.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/__tests__/catalog.test.ts
// Pass 17 Part 2
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, beforeEach } from 'vitest';
import { McpToolCatalog } from '../catalog.js';
import type { McpTool } from '../types.js';

function makeTool(name: string): McpTool {
  return {
    name,
    description: `Tool ${name}`,
    inputSchema: { type: 'object', properties: {}, required: [] },
  };
}

describe('McpToolCatalog', () => {
  let catalog: McpToolCatalog;

  beforeEach(() => {
    catalog = new McpToolCatalog();
  });

  it('starts empty', () => {
    expect(catalog.all()).toHaveLength(0);
    expect(catalog.size()).toBe(0);
  });

  it('registers tools from a server', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file'), makeTool('write_file')]);
    expect(catalog.size()).toBe(2);
  });

  it('produces qualified names', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file')]);
    const tool = catalog.all()[0]!;
    expect(tool.qualifiedName).toBe('mcp__srv_a__read_file');
    expect(tool.originalName).toBe('read_file');
  });

  it('getByQualifiedName returns the tool', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file')]);
    const tool = catalog.getByQualifiedName('mcp__srv_a__read_file');
    expect(tool).toBeDefined();
    expect(tool?.originalName).toBe('read_file');
  });

  it('returns undefined for unknown qualified name', () => {
    expect(catalog.getByQualifiedName('mcp__unknown__tool')).toBeUndefined();
  });

  it('filters tools by allowedTools list', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file'), makeTool('write_file')], ['read_file']);
    expect(catalog.size()).toBe(1);
    expect(catalog.all()[0]?.originalName).toBe('read_file');
  });

  it('excludes tools in deniedTools list', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file'), makeTool('write_file')], undefined, ['write_file']);
    expect(catalog.size()).toBe(1);
    expect(catalog.all()[0]?.originalName).toBe('read_file');
  });

  it('replaces tools on repeated updateServer call', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file'), makeTool('write_file')]);
    catalog.updateServer('srv-a', 'Server A', [makeTool('delete_file')]);
    expect(catalog.size()).toBe(1);
    expect(catalog.all()[0]?.originalName).toBe('delete_file');
  });

  it('removes server tools on removeServer', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file')]);
    catalog.updateServer('srv-b', 'Server B', [makeTool('search')]);
    catalog.removeServer('srv-a');
    expect(catalog.size()).toBe(1);
    expect(catalog.all()[0]?.serverId).toBe('srv-b');
  });

  it('getByServer returns only tools from that server', () => {
    catalog.updateServer('srv-a', 'Server A', [makeTool('read_file')]);
    catalog.updateServer('srv-b', 'Server B', [makeTool('search'), makeTool('index')]);
    expect(catalog.getByServer('srv-b')).toHaveLength(2);
    expect(catalog.getByServer('srv-a')).toHaveLength(1);
  });

  it('parseQualifiedName returns correct parts', () => {
    const result = catalog.parseQualifiedName('mcp__my_server__my_tool');
    expect(result).toEqual({ serverId: 'my_server', toolName: 'my_tool' });
  });

  it('parseQualifiedName returns null for non-MCP names', () => {
    expect(catalog.parseQualifiedName('read_file')).toBeNull();
    expect(catalog.parseQualifiedName('mcp__no_double_underscore')).toBeNull();
  });
});
packages/tools-mcp/src/__tests__/bridge.test.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/__tests__/bridge.test.ts
// Pass 17 Part 2
// ─────────────────────────────────────────────────────────────────────────────

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { McpToolRegistryBridge } from '../bridge.js';
import type { IBridgeToolRegistry, BridgeToolDefinition } from '../bridge.js';
import type { McpServerRegistry } from '../registry.js';
import type { McpBridgedTool, McpRegistryListener, McpTool } from '../types.js';
import { McpToolCatalog } from '../catalog.js';

// ── Mock registry ─────────────────────────────────────────────────────────────

function makeBridgedTool(serverId: string, name: string): McpBridgedTool {
  const catalog = new McpToolCatalog();
  const qualifiedName = catalog.qualify(serverId, name);
  return {
    qualifiedName,
    originalName: name,
    serverId,
    serverDisplayName: `Server ${serverId}`,
    tool: {
      name,
      description: `Does ${name}`,
      inputSchema: { type: 'object', properties: {}, required: [] },
    },
  };
}

function makeMockRegistry(initialTools: McpBridgedTool[] = []): {
  registry: McpServerRegistry;
  emit: (event: Parameters<McpRegistryListener>[0]) => void;
  catalog: McpToolCatalog;
} {
  const listeners: McpRegistryListener[] = [];
  const catalog = new McpToolCatalog();

  for (const tool of initialTools) {
    catalog.updateServer(tool.serverId, tool.serverDisplayName, [tool.tool]);
  }

  const registry = {
    on: vi.fn((listener: McpRegistryListener) => {
      listeners.push(listener);
      return () => {
        const idx = listeners.indexOf(listener);
        if (idx !== -1) listeners.splice(idx, 1);
      };
    }),
    listBridgedTools: vi.fn(() => catalog.all()),
    getCatalog: vi.fn(() => catalog),
    callTool: vi.fn(async (_qn: string, _args: Record<string, unknown>) => ({
      content: [{ type: 'text' as const, text: 'result' }],
      isError: false,
    })),
  } as unknown as McpServerRegistry;

  const emit = (event: Parameters<McpRegistryListener>[0]) => {
    for (const l of listeners) l(event);
  };

  return { registry, emit, catalog };
}

function makeMockToolRegistry(): {
  toolRegistry: IBridgeToolRegistry;
  registered: Map<string, BridgeToolDefinition>;
  unregistered: string[];
} {
  const registered = new Map<string, BridgeToolDefinition>();
  const unregistered: string[] = [];

  const toolRegistry: IBridgeToolRegistry = {
    register: vi.fn((tool: BridgeToolDefinition) => registered.set(tool.name, tool)),
    unregister: vi.fn((name: string) => {
      unregistered.push(name);
      registered.delete(name);
    }),
  };

  return { toolRegistry, registered, unregistered };
}

// ── Tests ─────────────────────────────────────────────────────────────────────

describe('McpToolRegistryBridge', () => {
  it('registers existing tools on attach', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry } = makeMockRegistry([tool]);
    const { toolRegistry, registered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    expect(registered.has(tool.qualifiedName)).toBe(true);
  });

  it('registers tool with correct description', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry } = makeMockRegistry([tool]);
    const { toolRegistry, registered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    const def = registered.get(tool.qualifiedName)!;
    expect(def.description).toContain('[MCP:');
    expect(def.description).toContain('read_file');
  });

  it('unregisters tools on detach', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry } = makeMockRegistry([tool]);
    const { toolRegistry, unregistered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();
    bridge.detach();

    expect(unregistered).toContain(tool.qualifiedName);
  });

  it('updates tools when tools_updated event fires', () => {
    const initial = makeBridgedTool('srv-a', 'old_tool');
    const { registry, emit, catalog } = makeMockRegistry([initial]);
    const { toolRegistry, registered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    // Simulate server updating its tools
    const newTool: McpTool = {
      name: 'new_tool',
      description: 'A new tool',
      inputSchema: { type: 'object', properties: {}, required: [] },
    };
    catalog.updateServer('srv-a', 'Server srv-a', [newTool]);

    emit({ type: 'tools_updated', serverId: 'srv-a', tools: [newTool] });

    expect(registered.has('mcp__srv_a__new_tool')).toBe(true);
    expect(registered.has('mcp__srv_a__old_tool')).toBe(false);
  });

  it('removes tools on server_disconnected event', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry, emit } = makeMockRegistry([tool]);
    const { toolRegistry, unregistered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    emit({ type: 'server_disconnected', serverId: 'srv-a' });

    expect(unregistered).toContain(tool.qualifiedName);
  });

  it('removes tools on server_failed event', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry, emit } = makeMockRegistry([tool]);
    const { toolRegistry, unregistered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    emit({ type: 'server_failed', serverId: 'srv-a', error: 'Connection failed' });

    expect(unregistered).toContain(tool.qualifiedName);
  });

  it('tool handler calls callTool on registry', async () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry } = makeMockRegistry([tool]);
    const { toolRegistry, registered } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    const def = registered.get(tool.qualifiedName)!;
    const result = await def.handler({ path: '/tmp/test.txt' });

    expect(registry.callTool).toHaveBeenCalledWith(tool.qualifiedName, { path: '/tmp/test.txt' });
    expect(result.success).toBe(true);
    expect(result.output).toBe('result');
  });

  it('tool handler returns failure on error', async () => {
    const tool = makeBridgedTool('srv-a', 'bad_tool');
    const { registry } = makeMockRegistry([tool]);
    (registry.callTool as ReturnType<typeof vi.fn>).mockRejectedValueOnce(new Error('tool exploded'));

    const { toolRegistry, registered } = makeMockToolRegistry();
    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();

    const def = registered.get(tool.qualifiedName)!;
    const result = await def.handler({});

    expect(result.success).toBe(false);
    expect(result.error).toContain('tool exploded');
  });

  it('is idempotent: attach() twice does not double-register', () => {
    const tool = makeBridgedTool('srv-a', 'read_file');
    const { registry } = makeMockRegistry([tool]);
    const { toolRegistry } = makeMockToolRegistry();

    const bridge = new McpToolRegistryBridge(registry, toolRegistry);
    bridge.attach();
    bridge.attach(); // second call should be no-op

    expect(toolRegistry.register).toHaveBeenCalledTimes(1);
  });
});
packages/tools-mcp/src/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// packages/tools-mcp/src/index.ts
// Pass 17 Part 2 — Barrel export
// ─────────────────────────────────────────────────────────────────────────────

// ── Protocol types ────────────────────────────────────────────────────────────
export type {
  McpServerId,
  McpTransportKind,
  McpServerTransportConfig,
  McpServerConfig,
  McpImplementationInfo,
  McpCapabilities,
  McpInitializeResult,
  McpToolInputSchema,
  McpTool,
  McpToolCallParams,
  McpToolContentItem,
  McpToolCallResult,
  McpResource,
  McpResourceContent,
  McpPromptArgument,
  McpPrompt,
  McpConnectionState,
  McpServerState,
  McpBridgedTool,
  McpRegistryEvent,
  McpRegistryListener,
  JsonRpcId,
  JsonRpcRequest,
  JsonRpcNotification,
  JsonRpcSuccessResponse,
  JsonRpcErrorObject,
  JsonRpcErrorResponse,
  JsonRpcMessage,
} from './types.js';

// ── Errors ────────────────────────────────────────────────────────────────────
export {
  McpError,
  McpConnectionError,
  McpToolNotFoundError,
  McpToolCallError,
  McpTimeoutError,
  McpServerNotConnectedError,
  McpProtocolError,
} from './errors.js';
export type { McpErrorCode } from './errors.js';

// ── Transport ─────────────────────────────────────────────────────────────────
export type { ITransport } from './transport/types.js';
export { StdioTransport } from './transport/stdio.js';
export type { StdioTransportOptions } from './transport/stdio.js';
export { HttpSseTransport } from './transport/sse.js';
export type { HttpSseTransportOptions } from './transport/sse.js';

// ── Core classes ──────────────────────────────────────────────────────────────
export { McpClient } from './client.js';
export { McpToolCatalog } from './catalog.js';
export { McpServerRegistry } from './registry.js';
export type { McpServerRegistryOptions } from './registry.js';

// ── Bridge ────────────────────────────────────────────────────────────────────
export { McpToolRegistryBridge, createMcpRegistryFromEnv } from './bridge.js';
export type {
  BridgeToolDefinition,
  BridgeToolResult,
  IBridgeToolRegistry,
} from './bridge.js';
Now we move to the openclaw/ root-level package.

openclaw/package.json
JSON

{
  "name": "@locoworker/openclaw",
  "version": "0.1.0",
  "description": "OpenClaw — code analysis, pattern scanning, and introspection engine for LocoWorker",
  "license": "MIT",
  "private": false,
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": ["dist", "README.md"],
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "typecheck": "tsc --noEmit",
    "dev": "tsc --watch --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "biome check src",
    "clean": "rimraf dist"
  },
  "dependencies": {},
  "devDependencies": {
    "@types/node": "^20.14.0",
    "rimraf": "^6.0.1",
    "typescript": "^5.5.4",
    "vitest": "^2.0.5"
  },
  "peerDependencies": {
    "typescript": ">=5.5.0"
  },
  "keywords": [
    "code-analysis",
    "pattern-scanning",
    "ast",
    "security",
    "locoworker"
  ]
}
openclaw/tsconfig.json
JSON

{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "composite": true,
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"],
  "references": []
}
openclaw/README.md
Markdown

# @locoworker/openclaw

> **Status: Pass 17 scaffold — interfaces and stubs only.**
> Full implementation is planned for Pass 18.

OpenClaw is LocoWorker's **code analysis, pattern scanning, and introspection engine**.

---

## What OpenClaw does (planned)

OpenClaw sits between the raw filesystem / AST data produced by `packages/graphify`
and the agent's need to understand *what is interesting or risky* in a codebase.

Where `graphify` answers: *"what symbols exist and how are they connected?"*,
OpenClaw answers: *"what patterns, vulnerabilities, anti-patterns, or structural
issues exist, and how severe are they?"*

### Core capabilities (planned)

| Capability | Description |
|---|---|
| **Pattern scanner** | Configurable rule engine; match AST patterns, regex, or semantic heuristics |
| **Security analyzer** | Detect secrets, injection vectors, dangerous API usage, SSRF risks |
| **Structural analyzer** | Identify dead code, circular deps, overly complex functions, God objects |
| **Diff analyzer** | Scan only changed files between two git refs (fast incremental mode) |
| **Report builder** | Produce structured, machine-readable + human-readable scan reports |
| **Auto-remediation hooks** | Suggest or apply fixes for known violation classes |
| **graphify integration** | Accept a `GraphifyClient` and overlay findings onto the symbol graph |

---

## Architecture (contracts established in Pass 17)

```
IScanner                IAnalyzer               IReporter
  └─ scan(target)         └─ analyze(findings)    └─ report(results)
       │                       │                       │
  ScanTarget             AnalysisResult          ScanReport
  ScanFinding            AnalysisSeverity        ReportFormat
  ScanRule               AnalysisCategory
```

---

## Usage (Pass 18+)

```typescript
import { createScanner, createAnalyzer, createReporter } from '@locoworker/openclaw';

// Pass 17: these throw NotImplementedError
// Pass 18: these return real implementations
const scanner = createScanner({ target: { kind: 'directory', path: './src' } });
const findings = await scanner.scan();

const analyzer = createAnalyzer();
const results = await analyzer.analyze(findings);

const reporter = createReporter({ format: 'json' });
const report = await reporter.report(results);
console.log(report.summary);
```

---

## Relationship to other packages

```
packages/graphify        ← OpenClaw can consume graphify's symbol graph
packages/wiki            ← OpenClaw can write findings into the wiki
packages/autoresearch    ← AutoResearch can trigger OpenClaw scans
packages/security        ← OpenClaw can augment security package's audit log
openclaw/                ← this package (standalone, no internal deps in scaffold)
```

---

## Contributing

See `CONTRIBUTING.md` at the monorepo root.
OpenClaw is designed to be extended with custom rules/analyzers via the
`IScanner`, `IAnalyzer`, and `IReporter` interfaces defined in `src/types.ts`.
openclaw/src/constants.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/constants.ts
// Pass 17 Part 2 — OpenClaw constants
// ─────────────────────────────────────────────────────────────────────────────

export const OPENCLAW_VERSION = '0.1.0' as const;

/** Placeholder prefix used on all stub errors so callers can detect them */
export const NOT_IMPLEMENTED_PREFIX = '[OpenClaw NotImplemented]' as const;

/** Default severity thresholds */
export const DEFAULT_SEVERITY_THRESHOLD = 'low' as const;

/** Max findings to return in a single scan before truncation */
export const MAX_FINDINGS_PER_SCAN = 10_000 as const;

/** Built-in rule category identifiers */
export const RULE_CATEGORIES = [
  'security',
  'correctness',
  'style',
  'performance',
  'maintainability',
  'complexity',
  'dead-code',
  'dependency',
] as const;

export type RuleCategory = typeof RULE_CATEGORIES[number];

/** Built-in severity levels (ascending) */
export const SEVERITY_LEVELS = ['info', 'low', 'medium', 'high', 'critical'] as const;
export type SeverityLevel = typeof SEVERITY_LEVELS[number];

/** Supported scan target kinds */
export const SCAN_TARGET_KINDS = ['file', 'directory', 'git-diff', 'symbol-graph'] as const;
export type ScanTargetKind = typeof SCAN_TARGET_KINDS[number];

/** Supported report output formats */
export const REPORT_FORMATS = ['json', 'markdown', 'sarif', 'html'] as const;
export type ReportFormat = typeof REPORT_FORMATS[number];
openclaw/src/types.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/types.ts
// Pass 17 Part 2 — OpenClaw interface contracts
//
// These are the canonical interfaces for OpenClaw's three main subsystems:
//   IScanner   — traverses source, applies rules, emits findings
//   IAnalyzer  — aggregates + scores + categorises findings
//   IReporter  — renders results in a requested format
//
// All interfaces are intentionally narrow in Pass 17.
// Pass 18 will extend them with richer options once implementations exist.
// ─────────────────────────────────────────────────────────────────────────────

import type {
  RuleCategory,
  SeverityLevel,
  ScanTargetKind,
  ReportFormat,
} from './constants.js';

// ── Scan target ───────────────────────────────────────────────────────────────

export type ScanTarget =
  | {
      readonly kind: 'file';
      /** Absolute path to a single file */
      readonly path: string;
    }
  | {
      readonly kind: 'directory';
      /** Root directory to scan recursively */
      readonly path: string;
      /** Glob patterns to include (default: all source files) */
      readonly include?: readonly string[];
      /** Glob patterns to exclude */
      readonly exclude?: readonly string[];
    }
  | {
      readonly kind: 'git-diff';
      /** Git ref for the "before" side (e.g. HEAD~1) */
      readonly fromRef: string;
      /** Git ref for the "after" side (e.g. HEAD) */
      readonly toRef: string;
      /** Repo root directory */
      readonly repoPath: string;
    }
  | {
      readonly kind: 'symbol-graph';
      /** Symbol IDs from a graphify graph to scan */
      readonly symbolIds: readonly string[];
      /** Path to the graphify SQLite DB */
      readonly graphDbPath: string;
    };

// ── Rules ─────────────────────────────────────────────────────────────────────

export interface ScanRule {
  /** Unique rule ID (e.g. "SEC001", "CPLX005") */
  readonly id: string;
  readonly name: string;
  readonly description: string;
  readonly category: RuleCategory;
  readonly defaultSeverity: SeverityLevel;
  /** Whether this rule is enabled by default */
  readonly enabledByDefault: boolean;
  /** Optional documentation URL */
  readonly docsUrl?: string;
  /** Tags for grouping / filtering */
  readonly tags?: readonly string[];
}

export interface ScanRuleOverride {
  readonly ruleId: string;
  readonly enabled?: boolean;
  readonly severity?: SeverityLevel;
}

// ── Findings ──────────────────────────────────────────────────────────────────

export interface SourceLocation {
  /** Absolute file path */
  readonly file: string;
  /** 1-based line number */
  readonly line: number;
  /** 1-based column number */
  readonly column: number;
  /** Optional end position for highlighting a range */
  readonly endLine?: number;
  readonly endColumn?: number;
}

export interface ScanFinding {
  /** Unique finding ID within a scan run */
  readonly id: string;
  /** Rule that produced this finding */
  readonly ruleId: string;
  readonly ruleName: string;
  readonly category: RuleCategory;
  readonly severity: SeverityLevel;
  /** Human-readable description of what was found */
  readonly message: string;
  /** Source location where the finding was detected */
  readonly location: SourceLocation;
  /** Optional code snippet around the finding */
  readonly snippet?: string;
  /** Optional suggested fix (plain text or diff) */
  readonly suggestedFix?: string;
  /** Arbitrary metadata for rule-specific data */
  readonly metadata?: Record<string, unknown>;
}

// ── Scanner ───────────────────────────────────────────────────────────────────

export interface ScanOptions {
  readonly target: ScanTarget;
  /** Rules to run. Empty = all enabled rules. */
  readonly rules?: readonly string[];
  /** Per-rule overrides */
  readonly ruleOverrides?: readonly ScanRuleOverride[];
  /** Minimum severity to include in results */
  readonly severityThreshold?: SeverityLevel;
  /** Max findings to return (default: MAX_FINDINGS_PER_SCAN) */
  readonly maxFindings?: number;
  /** If true, emit findings as they are found (streaming mode) */
  readonly stream?: boolean;
}

export interface ScanResult {
  readonly scanId: string;
  readonly target: ScanTarget;
  readonly startedAt: number;
  readonly completedAt: number;
  readonly durationMs: number;
  readonly findings: readonly ScanFinding[];
  readonly totalFilesScanned: number;
  readonly totalRulesApplied: number;
  /** True if scan was truncated due to maxFindings limit */
  readonly truncated: boolean;
}

export interface IScanner {
  /** Run a complete scan and return all findings */
  scan(options: ScanOptions): Promise<ScanResult>;

  /** Stream findings as they are produced */
  scanStream(options: ScanOptions): AsyncGenerator<ScanFinding, ScanResult, unknown>;

  /** List all rules known to this scanner */
  listRules(): readonly ScanRule[];

  /** Check whether a specific rule is supported */
  hasRule(ruleId: string): boolean;
}

// ── Analyzer ──────────────────────────────────────────────────────────────────

export interface AnalysisSummary {
  readonly totalFindings: number;
  readonly bySeverity: Record<SeverityLevel, number>;
  readonly byCategory: Record<RuleCategory, number>;
  readonly topRules: Array<{ ruleId: string; count: number }>;
  readonly riskScore: number; // 0–100
}

export interface AnalysisResult {
  readonly analysisId: string;
  readonly scanResult: ScanResult;
  readonly summary: AnalysisSummary;
  readonly groups: readonly FindingGroup[];
  readonly analysedAt: number;
}

export interface FindingGroup {
  readonly groupId: string;
  readonly label: string;
  readonly severity: SeverityLevel;
  readonly findings: readonly ScanFinding[];
  /** Suggested remediation for the group as a whole */
  readonly remediationNote?: string;
}

export interface AnalyzeOptions {
  /** Group findings by file, rule, or severity */
  readonly groupBy?: 'file' | 'rule' | 'severity';
  /** Whether to compute a risk score */
  readonly computeRiskScore?: boolean;
}

export interface IAnalyzer {
  analyze(scanResult: ScanResult, options?: AnalyzeOptions): Promise<AnalysisResult>;
}

// ── Reporter ──────────────────────────────────────────────────────────────────

export interface ReportOptions {
  readonly format: ReportFormat;
  /** Output file path. If not set, report is returned as a string. */
  readonly outputPath?: string;
  /** Whether to include code snippets in the report */
  readonly includeSnippets?: boolean;
  /** Whether to include suggested fixes */
  readonly includeFixes?: boolean;
  /** Max findings to include in the report body */
  readonly maxFindings?: number;
}

export interface ScanReport {
  readonly reportId: string;
  readonly format: ReportFormat;
  /** Rendered report content (JSON string, Markdown, SARIF JSON, or HTML) */
  readonly content: string;
  /** Plain-text summary (always populated regardless of format) */
  readonly summary: string;
  readonly generatedAt: number;
  /** Path where report was written, if outputPath was set */
  readonly writtenTo?: string;
}

export interface IReporter {
  report(analysisResult: AnalysisResult, options: ReportOptions): Promise<ScanReport>;
}

// ── Top-level factory types ───────────────────────────────────────────────────

export interface OpenClawConfig {
  /** Enable/disable built-in rule packs */
  readonly enabledRulePacks?: readonly string[];
  /** Custom rule overrides applied globally */
  readonly globalRuleOverrides?: readonly ScanRuleOverride[];
  /** Default severity threshold for scans */
  readonly defaultSeverityThreshold?: SeverityLevel;
  /** Whether to integrate with graphify (Pass 18+) */
  readonly graphifyIntegration?: boolean;
}

export interface IOpenClaw {
  createScanner(config?: Partial<OpenClawConfig>): IScanner;
  createAnalyzer(): IAnalyzer;
  createReporter(): IReporter;
  /** Run a complete scan → analyze → report pipeline in one call */
  runPipeline(options: {
    scan: ScanOptions;
    analyze?: AnalyzeOptions;
    report: ReportOptions;
  }): Promise<{ scanResult: ScanResult; analysisResult: AnalysisResult; report: ScanReport }>;
}
openclaw/src/stubs/analyzer.stub.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/stubs/analyzer.stub.ts
// Pass 17 Part 2 — AnalyzerStub (throws NotImplemented)
// ─────────────────────────────────────────────────────────────────────────────

import type { IAnalyzer, ScanResult, AnalysisResult, AnalyzeOptions } from '../types.js';
import { NOT_IMPLEMENTED_PREFIX } from '../constants.js';

export class AnalyzerStub implements IAnalyzer {
  async analyze(scanResult: ScanResult, _options?: AnalyzeOptions): Promise<AnalysisResult> {
    throw new Error(
      `${NOT_IMPLEMENTED_PREFIX} IAnalyzer.analyze() is not yet implemented. ` +
      `This stub is scaffolded in Pass 17. Full implementation lands in Pass 18.`,
    );
  }
}
openclaw/src/stubs/scanner.stub.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/stubs/scanner.stub.ts
// Pass 17 Part 2 — ScannerStub (throws NotImplemented)
// ─────────────────────────────────────────────────────────────────────────────

import type { IScanner, ScanOptions, ScanResult, ScanFinding, ScanRule } from '../types.js';
import { NOT_IMPLEMENTED_PREFIX } from '../constants.js';

export class ScannerStub implements IScanner {
  async scan(_options: ScanOptions): Promise<ScanResult> {
    throw new Error(
      `${NOT_IMPLEMENTED_PREFIX} IScanner.scan() is not yet implemented. ` +
      `This stub is scaffolded in Pass 17. Full implementation lands in Pass 18.`,
    );
  }

  async *scanStream(_options: ScanOptions): AsyncGenerator<ScanFinding, ScanResult, unknown> {
    throw new Error(
      `${NOT_IMPLEMENTED_PREFIX} IScanner.scanStream() is not yet implemented.`,
    );
  }

  listRules(): readonly ScanRule[] {
    return [];
  }

  hasRule(_ruleId: string): boolean {
    return false;
  }
}
openclaw/src/stubs/reporter.stub.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/stubs/reporter.stub.ts
// Pass 17 Part 2 — ReporterStub (throws NotImplemented)
// ─────────────────────────────────────────────────────────────────────────────

import type { IReporter, AnalysisResult, ReportOptions, ScanReport } from '../types.js';
import { NOT_IMPLEMENTED_PREFIX } from '../constants.js';

export class ReporterStub implements IReporter {
  async report(_analysisResult: AnalysisResult, _options: ReportOptions): Promise<ScanReport> {
    throw new Error(
      `${NOT_IMPLEMENTED_PREFIX} IReporter.report() is not yet implemented. ` +
      `This stub is scaffolded in Pass 17. Full implementation lands in Pass 18.`,
    );
  }
}
openclaw/src/stubs/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/stubs/index.ts
// Pass 17 Part 2
// ─────────────────────────────────────────────────────────────────────────────

export { ScannerStub } from './scanner.stub.js';
export { AnalyzerStub } from './analyzer.stub.js';
export { ReporterStub } from './reporter.stub.js';
openclaw/src/index.ts
TypeScript

// ─────────────────────────────────────────────────────────────────────────────
// openclaw/src/index.ts
// Pass 17 Part 2 — OpenClaw barrel export + factory functions
//
// In Pass 17, createScanner / createAnalyzer / createReporter return stubs.
// In Pass 18, replace the stub imports with real implementations.
// ─────────────────────────────────────────────────────────────────────────────

// ── Constants ─────────────────────────────────────────────────────────────────
export {
  OPENCLAW_VERSION,
  NOT_IMPLEMENTED_PREFIX,
  DEFAULT_SEVERITY_THRESHOLD,
  MAX_FINDINGS_PER_SCAN,
  RULE_CATEGORIES,
  SEVERITY_LEVELS,
  SCAN_TARGET_KINDS,
  REPORT_FORMATS,
} from './constants.js';
export type {
  RuleCategory,
  SeverityLevel,
  ScanTargetKind,
  ReportFormat,
} from './constants.js';

// ── Types / interfaces ────────────────────────────────────────────────────────
export type {
  ScanTarget,
  ScanRule,
  ScanRuleOverride,
  SourceLocation,
  ScanFinding,
  ScanOptions,
  ScanResult,
  IScanner,
  AnalysisSummary,
  AnalysisResult,
  FindingGroup,
  AnalyzeOptions,
  IAnalyzer,
  ReportOptions,
  ScanReport,
  IReporter,
  OpenClawConfig,
  IOpenClaw,
} from './types.js';

// ── Stubs (Pass 17) ───────────────────────────────────────────────────────────
export { ScannerStub, AnalyzerStub, ReporterStub } from './stubs/index.js';

// ── Factory functions ─────────────────────────────────────────────────────────
// Pass 17: return stubs.
// Pass 18: swap ScannerStub → RealScanner, etc.

import { ScannerStub } from './stubs/scanner.stub.js';
import { AnalyzerStub } from './stubs/analyzer.stub.js';
import { ReporterStub } from './stubs/reporter.stub.js';
import type { IScanner, IAnalyzer, IReporter, IOpenClaw, OpenClawConfig, ScanOptions, AnalyzeOptions, ReportOptions } from './types.js';

export function createScanner(_config?: Partial<OpenClawConfig>): IScanner {
  return new ScannerStub();
}

export function createAnalyzer(): IAnalyzer {
  return new AnalyzerStub();
}

export function createReporter(): IReporter {
  return new ReporterStub();
}

/**
 * Full OpenClaw facade.
 * In Pass 17 all methods delegate to stubs.
 * In Pass 18 replace with real implementations.
 */
export function createOpenClaw(config?: Partial<OpenClawConfig>): IOpenClaw {
  const scanner = createScanner(config);
  const analyzer = createAnalyzer();
  const reporter = createReporter();

  return {
    createScanner: (_cfg) => createScanner(_cfg ?? config),
    createAnalyzer: () => createAnalyzer(),
    createReporter: () => createReporter(),

    async runPipeline(options: {
      scan: ScanOptions;
      analyze?: AnalyzeOptions;
      report: ReportOptions;
    }) {
      const scanResult = await scanner.scan(options.scan);
      const analysisResult = await analyzer.analyze(scanResult, options.analyze);
      const report = await reporter.report(analysisResult, options.report);
      return { scanResult, analysisResult, report };
    },
  };
}
Updated pnpm-workspace.yaml (Pass 17 Part 2 — adds openclaw + tools-mcp)
YAML

# ─────────────────────────────────────────────────────────────────────────────
# LocoWorker monorepo — pnpm workspace config
# Pass 17 Part 2 — adds openclaw/ and packages/tools-mcp
# Rule: later pass wins; this replaces the Pass 16 Part 1 version.
# ─────────────────────────────────────────────────────────────────────────────

packages:
  # ── Shared utilities (no internal deps)
  - "packages/shared"

  # ── Core agent engine
  - "packages/core"

  # ── Agent capability packages (depend on core + shared)
  - "packages/memory"
  - "packages/graphify"
  - "packages/wiki"
  - "packages/kairos"
  - "packages/orchestrator"
  - "packages/security"
  - "packages/gateway"

  # ── Provider registry (depend on shared)
  - "packages/providers"

  # ── Research + simulation (depend on core + capability packages)
  - "packages/autoresearch"
  - "packages/mirofish"

  # ── Tool packages (depend on core + shared + security)
  - "packages/tools-fs"
  - "packages/tools-bash"
  - "packages/tools-git"
  - "packages/tools-search"
  - "packages/tools-web"
  - "packages/tools-mcp"        # ← NEW Pass 17 Part 2

  # ── OpenClaw — code analysis engine (standalone, no internal deps yet)
  - "openclaw"                   # ← NEW Pass 17 Part 2

  # ── Applications (depend on all packages above)
  - "apps/cli"
  - "apps/gateway"
  - "apps/desktop"
  - "apps/dashboard"

  # ── Test suites (not published; depend on all apps + packages)
  - "tests/integration"
  - "tests/e2e"
Updated tsconfig.json (root — adds openclaw + tools-mcp references)
JSON

{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "LocoWorker root tsconfig — project references (Pass 17 Part 2)",
  "files": [],
  "references": [
    { "path": "packages/shared" },
    { "path": "packages/core" },
    { "path": "packages/memory" },
    { "path": "packages/graphify" },
    { "path": "packages/wiki" },
    { "path": "packages/kairos" },
    { "path": "packages/orchestrator" },
    { "path": "packages/security" },
    { "path": "packages/gateway" },
    { "path": "packages/providers" },
    { "path": "packages/autoresearch" },
    { "path": "packages/mirofish" },
    { "path": "packages/tools-fs" },
    { "path": "packages/tools-bash" },
    { "path": "packages/tools-git" },
    { "path": "packages/tools-search" },
    { "path": "packages/tools-web" },
    { "path": "packages/tools-mcp" },
    { "path": "openclaw" },
    { "path": "apps/cli" },
    { "path": "apps/gateway" },
    { "path": "apps/desktop" },
    { "path": "apps/dashboard" },
    { "path": "tests/integration" },
    { "path": "tests/e2e" }
  ]
}
State after Pass 17 (both parts complete)
Package	Status	Key exports
packages/providers	✅ Pass 17 P1	ProviderRegistry, ProviderRouter, all adapters, routing strategies, cost tracking, health monitor
packages/tools-mcp	✅ Pass 17 P2	McpServerRegistry, McpClient, McpToolCatalog, McpToolRegistryBridge, StdioTransport, HttpSseTransport
openclaw/	✅ Pass 17 P2	IScanner, IAnalyzer, IReporter, all stubs, createOpenClaw() factory
packages/core/src/providers/bridge.ts	✅ Pass 17 P1	Wiring point for Pass 18 core cutover
pnpm-workspace.yaml	✅ Updated	Includes all 3 new packages
tsconfig.json (root)	✅ Updated	References all 3 new packages
Pass 18	🔜 Next	Full packages/providers + packages/core cutover wiring; openclaw real implementations; ADRs for Tauri/Ink decisions
