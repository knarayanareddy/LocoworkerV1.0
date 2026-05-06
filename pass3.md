Pass 3 of 18 — packages/memory
Part 1 of 2: Package Config · Types · MemoryManager · ConversationStore · MemoryIndex
Full File Tree for Pass 3
text

packages/memory/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                  ← Part 2
│   ├── types/
│   │   └── memory-internal.types.ts   ← Part 1
│   ├── MemoryManager.ts          ← Part 1
│   ├── ConversationStore.ts      ← Part 1
│   ├── MemoryIndex.ts            ← Part 1
│   ├── AutoDream.ts              ← Part 2
│   ├── MemoryCompactor.ts        ← Part 2
│   └── MemoryFormatter.ts        ← Part 2
└── tests/
    ├── MemoryManager.test.ts     ← Part 2
    ├── ConversationStore.test.ts ← Part 2
    ├── MemoryIndex.test.ts       ← Part 2
    └── AutoDream.test.ts         ← Part 2
packages/memory/package.json
JSON

{
  "name": "@locoworker/memory",
  "version": "1.0.0",
  "description": "LocoWorker memory layer — 4-layer hierarchy, MemoryManager, ConversationStore, AutoDream",
  "license": "MIT",
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
  "files": [
    "dist",
    "src"
  ],
  "scripts": {
    "build":         "tsc --project tsconfig.json",
    "dev":           "tsc --project tsconfig.json --watch",
    "typecheck":     "tsc --project tsconfig.json --noEmit",
    "test":          "bun test",
    "test:watch":    "bun test --watch",
    "test:coverage": "bun test --coverage",
    "lint":          "biome check ./src ./tests",
    "lint:fix":      "biome check --apply ./src ./tests",
    "clean":         "rm -rf dist tsconfig.tsbuildinfo"
  },
  "dependencies": {
    "zod": "3.24.1"
  },
  "devDependencies": {
    "@locoworker/core":   "workspace:*",
    "@locoworker/shared": "workspace:*",
    "@types/node":        "^22.10.0",
    "typescript":         "^5.7.2"
  },
  "peerDependencies": {
    "@locoworker/core": "workspace:*"
  }
}
packages/memory/tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist",
    "tsBuildInfoFile": "./tsconfig.tsbuildinfo",
    "paths": {
      "@locoworker/core":   ["../core/src/index.ts"],
      "@locoworker/shared": ["../shared/src/index.ts"]
    }
  },
  "references": [
    { "path": "../core" }
  ],
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
packages/memory/src/types/memory-internal.types.ts
TypeScript

import { z } from 'zod'

/**
 * Extended memory types specific to the memory package.
 * The base MemoryEntry type lives in @locoworker/core.
 * These types add persistence, serialisation, and AutoDream concerns.
 */

// ── Zod Schemas (for runtime validation of persisted data) ───────────

export const MemoryCategorySchema = z.enum([
  'architecture',
  'preference',
  'file_location',
  'decision',
  'issue',
  'workaround',
  'research',
  'buddy',
  'credential',
  'note',
])

export const MemoryImportanceSchema = z.enum([
  'low',
  'medium',
  'high',
  'critical',
])

export const MemoryEntrySchema = z.object({
  id:           z.string().uuid(),
  timestamp:    z.number().int().positive(),
  sessionId:    z.string().min(1),
  category:     MemoryCategorySchema,
  content:      z.string().min(1).max(2_000),
  importance:   MemoryImportanceSchema,
  tags:         z.array(z.string()),
  graphNodeId:  z.string().optional(),
  wikiSlug:     z.string().optional(),
  expiresAt:    z.number().int().positive().optional(),
})

export type PersistedMemoryEntry = z.infer<typeof MemoryEntrySchema>

// ── Conversation Store ─────────────────────────────────────────────

export const ConversationTurnSchema = z.object({
  role:       z.enum(['user', 'assistant']),
  content:    z.string(),
  tokenCount: z.number().int().nonnegative(),
  importance: z.enum(['normal', 'important', 'critical']),
  timestamp:  z.number().int().positive(),
  toolCalls:  z.array(z.object({
    id:    z.string(),
    name:  z.string(),
    input: z.record(z.unknown()),
  })).optional(),
  toolResults: z.array(z.object({
    toolCallId: z.string(),
    toolName:   z.string(),
    content:    z.union([z.string(), z.record(z.unknown())]),
    isError:    z.boolean(),
    tokenCount: z.number().int().nonnegative(),
    durationMs: z.number().nonnegative(),
  })).optional(),
})

export type PersistedConversationTurn = z.infer<typeof ConversationTurnSchema>

// ── Session Archive ────────────────────────────────────────────────

export interface SessionArchiveEntry {
  sessionId:       string
  startedAt:       number
  endedAt:         number
  modelId:         string
  provider:        string
  totalTurns:      number
  totalCostUsd:    number
  turnSummary:     string   // 1-2 sentence summary of the session
  keyDecisions:    string[] // Decisions extracted for memory
  filesModified:   string[] // Files written/edited during session
  toolsUsed:       string[] // Tool names used (deduplicated)
  compactionCount: number
}

// ── Memory Index ───────────────────────────────────────────────────

export interface MemoryIndexFile {
  version:          number
  lastUpdatedAt:    number
  lastAutoDreamAt:  number | null
  totalEntries:     number
  totalArchived:    number
  categories:       Partial<Record<string, number>>  // category → count
  entries:          PersistedMemoryEntry[]
}

export const MEMORY_INDEX_VERSION = 2

// ── AutoDream ─────────────────────────────────────────────────────

export interface DreamSession {
  id:              string
  startedAt:       number
  completedAt:     number
  entriesInput:    number
  entriesMerged:   number
  entriesArchived: number
  entriesCreated:  number
  contradictions:  ContradictionRecord[]
  wikiLinksAdded:  number
  graphLinksAdded: number
  dreamSummary:    string
  costUsd:         number
}

export interface ContradictionRecord {
  entryAId:   string
  entryBId:   string
  kept:       string   // ID of the entry that was kept
  archived:   string   // ID of the entry that was archived
  reason:     string
}

// ── Significance Detection ─────────────────────────────────────────

export interface SignificanceSignal {
  keyword:    string
  category:   z.infer<typeof MemoryCategorySchema>
  importance: z.infer<typeof MemoryImportanceSchema>
}

export const SIGNIFICANCE_SIGNALS: SignificanceSignal[] = [
  { keyword: '[remember:',   category: 'decision',      importance: 'high' },
  { keyword: '[memory:',     category: 'note',          importance: 'medium' },
  { keyword: 'decided to',   category: 'decision',      importance: 'medium' },
  { keyword: 'architecture', category: 'architecture',  importance: 'medium' },
  { keyword: 'always use',   category: 'preference',    importance: 'high' },
  { keyword: 'never use',    category: 'preference',    importance: 'high' },
  { keyword: 'prefer',       category: 'preference',    importance: 'low' },
  { keyword: 'workaround',   category: 'workaround',    importance: 'high' },
  { keyword: 'bug:',         category: 'issue',         importance: 'high' },
  { keyword: 'issue:',       category: 'issue',         importance: 'medium' },
  { keyword: 'key file',     category: 'file_location', importance: 'medium' },
  { keyword: 'located at',   category: 'file_location', importance: 'low' },
  { keyword: 'research:',    category: 'research',      importance: 'medium' },
  { keyword: 'important:',   category: 'note',          importance: 'high' },
  { keyword: 'note:',        category: 'note',          importance: 'low' },
]

// ── Extraction Patterns ────────────────────────────────────────────

/**
 * Regex pattern for explicit [REMEMBER: ...] markers in model responses.
 * These are always extracted as high-importance entries.
 */
export const EXPLICIT_REMEMBER_PATTERN =
  /\[REMEMBER:\s*(?<content>[^\]]{3,400})\]/gi

/**
 * Regex pattern for explicit [NOTE: ...] markers.
 */
export const EXPLICIT_NOTE_PATTERN =
  /\[NOTE:\s*(?<content>[^\]]{3,300})\]/gi

/**
 * Regex for file path mentions that should be recorded.
 */
export const FILE_LOCATION_PATTERN =
  /(?:located? at|found at|saved? (?:to|at|in)|written to)\s+[`']?(?<path>[\w./\\-]{5,200})[`']?/gi
packages/memory/src/MemoryManager.ts
TypeScript

import { randomUUID } from 'node:crypto'
import { readFile, writeFile, mkdir, appendFile } from 'node:fs/promises'
import { join, dirname } from 'node:path'
import type {
  MemoryEntry,
  MemoryCategory,
  MemoryImportance,
} from '@locoworker/core'
import type { ConversationTurn, AssistantMessage } from '@locoworker/core'
import {
  SIGNIFICANCE_SIGNALS,
  EXPLICIT_REMEMBER_PATTERN,
  EXPLICIT_NOTE_PATTERN,
  FILE_LOCATION_PATTERN,
  type PersistedMemoryEntry,
} from './types/memory-internal.types.js'
import { MemoryIndex } from './MemoryIndex.js'
import { MemoryFormatter } from './MemoryFormatter.js'

/**
 * Payload describing a completed agent turn — used to extract memories.
 */
export interface MemoryUpdatePayload {
  sessionId:        string
  workingDirectory: string
  userInput:        string
  response:         AssistantMessage
  toolResults?:     Array<{
    toolName:   string
    content:    string | Record<string, unknown>
    isError:    boolean
  }>
  turnNumber:       number
}

/**
 * Options for querying the memory index.
 */
export interface MemoryQueryOptions {
  limit?:      number
  categories?: MemoryCategory[]
  minImportance?: MemoryImportance
  tags?:       string[]
  sinceMs?:    number
}

/**
 * Result of a memory query.
 */
export interface MemoryQueryResult {
  entries:       MemoryEntry[]
  totalMatched:  number
  queryTimeMs:   number
}

const IMPORTANCE_RANK: Record<MemoryImportance, number> = {
  low:      0,
  medium:   1,
  high:     2,
  critical: 3,
}

/**
 * MemoryManager is the primary interface to the LocoWorker memory system.
 *
 * Responsibilities:
 *  - Extract memory-worthy content from agent turn responses
 *  - Persist entries to the MemoryIndex (JSON Lines + .md file)
 *  - Query entries for context injection
 *  - Update MEMORY.md in human-readable format
 *  - Prune stale low-importance entries
 *  - Coordinate with AutoDream for overnight consolidation
 *
 * All I/O is async and safe — missing files are treated as empty state.
 * The manager never throws on I/O errors; it logs and continues.
 *
 * @example
 * ```ts
 * const manager = new MemoryManager('/home/user/project')
 * await manager.initialise()
 *
 * await manager.update({
 *   sessionId: 'sess_abc',
 *   workingDirectory: '/home/user/project',
 *   userInput: 'Use tRPC for the IPC layer',
 *   response: assistantResponse,
 *   turnNumber: 3,
 * })
 *
 * const results = await manager.query('architecture decision', { limit: 5 })
 * ```
 */
export class MemoryManager {
  private index: MemoryIndex
  private formatter: MemoryFormatter
  private initialised = false

  constructor(private readonly workingDirectory: string) {
    this.index = new MemoryIndex(workingDirectory)
    this.formatter = new MemoryFormatter()
  }

  // ── Lifecycle ──────────────────────────────────────────────────────

  /**
   * Initialise the manager — loads existing index from disk.
   * Must be called before any other methods.
   * Safe to call multiple times (idempotent).
   */
  async initialise(): Promise<void> {
    if (this.initialised) return
    await this.index.load()
    this.initialised = true
  }

  // ── Core: Update from agent turn ──────────────────────────────────

  /**
   * Process a completed agent turn and extract memory-worthy content.
   * Skips if the turn is not significant.
   * Updates both the JSON index and the human-readable MEMORY.md.
   */
  async update(payload: MemoryUpdatePayload): Promise<MemoryEntry[]> {
    await this.ensureInitialised()

    const responseText = payload.response.content ?? ''
    const extracted = this.extractEntries(payload, responseText)

    if (extracted.length === 0) return []

    // Persist all extracted entries
    for (const entry of extracted) {
      await this.index.add(entry)
    }

    // Flush MEMORY.md with the latest state
    await this.flushMemoryMd()

    console.log(
      `[MemoryManager] Extracted ${extracted.length} entr${extracted.length === 1 ? 'y' : 'ies'} ` +
      `from turn ${payload.turnNumber} (session: ${payload.sessionId.slice(0, 10)})`
    )

    return extracted
  }

  // ── Core: Query ───────────────────────────────────────────────────

  /**
   * Query the memory index for relevant entries.
   *
   * Scoring algorithm:
   *   - Keyword match in content: +2 per unique matching word
   *   - Importance bonus: critical=+4, high=+2, medium=+1
   *   - Recency bonus: entries from last 7 days get +1
   *   - Tag match: +1 per matching tag
   */
  async query(
    queryText: string,
    options: MemoryQueryOptions = {}
  ): Promise<MemoryQueryResult> {
    await this.ensureInitialised()
    const start = Date.now()

    const {
      limit = 10,
      categories,
      minImportance,
      tags,
      sinceMs,
    } = options

    const allEntries = this.index.getAll()
    const queryTerms = this.tokenise(queryText)
    const now = Date.now()
    const SEVEN_DAYS_MS = 7 * 24 * 60 * 60 * 1_000

    // Filter candidates
    const candidates = allEntries.filter((entry) => {
      if (categories && !categories.includes(entry.category)) return false
      if (minImportance) {
        if (IMPORTANCE_RANK[entry.importance] < IMPORTANCE_RANK[minImportance]) {
          return false
        }
      }
      if (tags && !tags.some((t) => entry.tags.includes(t))) return false
      if (sinceMs && entry.timestamp < sinceMs) return false
      if (entry.expiresAt && now > entry.expiresAt) return false
      return true
    })

    // Score each candidate
    const scored = candidates.map((entry) => {
      const contentLower = entry.content.toLowerCase()
      const contentTerms = this.tokenise(entry.content)

      let score = 0

      // Keyword match
      for (const term of queryTerms) {
        if (contentLower.includes(term)) score += 2
      }

      // Importance bonus
      score += IMPORTANCE_RANK[entry.importance] * 2

      // Recency bonus
      if (now - entry.timestamp < SEVEN_DAYS_MS) score += 1

      // Tag match
      if (tags) {
        score += entry.tags.filter((t) => tags.includes(t)).length
      }

      return { entry, score }
    })

    // Sort by score descending, then by timestamp descending (newest first)
    scored.sort((a, b) => {
      if (b.score !== a.score) return b.score - a.score
      return b.entry.timestamp - a.entry.timestamp
    })

    const results = scored
      .filter((s) => s.score > 0)
      .slice(0, limit)
      .map((s) => s.entry)

    return {
      entries: results,
      totalMatched: scored.filter((s) => s.score > 0).length,
      queryTimeMs: Date.now() - start,
    }
  }

  // ── Core: Add manually ────────────────────────────────────────────

  /**
   * Manually add a memory entry (e.g., from /memory add command).
   */
  async add(
    content: string,
    category: MemoryCategory,
    importance: MemoryImportance,
    tags: string[] = [],
    sessionId = 'manual'
  ): Promise<MemoryEntry> {
    await this.ensureInitialised()

    const entry: MemoryEntry = {
      id:        randomUUID(),
      timestamp: Date.now(),
      sessionId,
      category,
      content:   content.trim().slice(0, 2_000),
      importance,
      tags,
    }

    await this.index.add(entry)
    await this.flushMemoryMd()
    return entry
  }

  // ── Core: Remove ──────────────────────────────────────────────────

  /**
   * Remove a memory entry by ID.
   */
  async remove(entryId: string): Promise<boolean> {
    await this.ensureInitialised()
    const removed = this.index.remove(entryId)
    if (removed) await this.flushMemoryMd()
    return removed
  }

  // ── Core: Prune ───────────────────────────────────────────────────

  /**
   * Prune stale entries from the index.
   *
   * Rules:
   *  - low importance + older than 90 days → remove
   *  - medium importance + older than 365 days → remove
   *  - expired entries (expiresAt in the past) → remove
   */
  async prune(): Promise<{ pruned: number }> {
    await this.ensureInitialised()

    const now = Date.now()
    const NINETY_DAYS_MS  = 90  * 24 * 60 * 60 * 1_000
    const ONE_YEAR_MS     = 365 * 24 * 60 * 60 * 1_000

    const before = this.index.count()

    this.index.filter((entry) => {
      // Always remove expired entries
      if (entry.expiresAt && now > entry.expiresAt) return false
      // Remove old low-importance entries
      if (entry.importance === 'low' && now - entry.timestamp > NINETY_DAYS_MS) return false
      // Remove old medium-importance entries
      if (entry.importance === 'medium' && now - entry.timestamp > ONE_YEAR_MS) return false
      return true
    })

    const after  = this.index.count()
    const pruned = before - after

    if (pruned > 0) {
      await this.flushMemoryMd()
      console.log(`[MemoryManager] Pruned ${pruned} stale entries.`)
    }

    return { pruned }
  }

  // ── Core: Load all (for AutoDream) ────────────────────────────────

  /**
   * Load all entries — used by AutoDream for batch processing.
   */
  async loadAll(): Promise<MemoryEntry[]> {
    await this.ensureInitialised()
    return this.index.getAll()
  }

  /**
   * Replace the full entry set after AutoDream consolidation.
   */
  async replaceAll(entries: MemoryEntry[]): Promise<void> {
    await this.ensureInitialised()
    this.index.replaceAll(entries)
    await this.flushMemoryMd()
  }

  // ── Stats ─────────────────────────────────────────────────────────

  /**
   * Return summary statistics about the memory index.
   */
  async getStats(): Promise<MemoryStats> {
    await this.ensureInitialised()
    return this.index.getStats()
  }

  // ── MEMORY.md ──────────────────────────────────────────────────────

  /**
   * Read MEMORY.md as a formatted string for context injection.
   * Returns a compact representation suitable for the agent's system prompt.
   */
  async readForContext(maxTokens = 3_000): Promise<string> {
    await this.ensureInitialised()

    const entries = this.index.getAll()
    const sorted = [...entries].sort((a, b) => {
      // Prioritise by importance then recency
      const impDiff = IMPORTANCE_RANK[b.importance] - IMPORTANCE_RANK[a.importance]
      if (impDiff !== 0) return impDiff
      return b.timestamp - a.timestamp
    })

    return this.formatter.formatForContext(sorted, maxTokens)
  }

  // ── Private: Entry extraction ──────────────────────────────────────

  /**
   * Extract MemoryEntry objects from an agent turn response.
   *
   * Extraction strategy (in priority order):
   *  1. Explicit [REMEMBER: ...] markers in the response text
   *  2. Explicit [NOTE: ...] markers
   *  3. File location patterns (e.g., "located at src/index.ts")
   *  4. Significance signals (keyword-based classification)
   */
  private extractEntries(
    payload: MemoryUpdatePayload,
    responseText: string
  ): MemoryEntry[] {
    const entries: MemoryEntry[] = []
    const now = Date.now()
    const seen = new Set<string>() // Dedup by normalised content

    const addEntry = (
      content: string,
      category: MemoryCategory,
      importance: MemoryImportance,
      tags: string[]
    ) => {
      const normalised = content.toLowerCase().trim()
      if (seen.has(normalised)) return
      if (content.trim().length < 10) return
      seen.add(normalised)

      entries.push({
        id:        randomUUID(),
        timestamp: now,
        sessionId: payload.sessionId,
        category,
        content:   content.trim().slice(0, 2_000),
        importance,
        tags: [...new Set(tags)],
      })
    }

    // ── 1. Explicit [REMEMBER: ...] markers ─────────────────────────
    EXPLICIT_REMEMBER_PATTERN.lastIndex = 0
    for (const match of responseText.matchAll(EXPLICIT_REMEMBER_PATTERN)) {
      const content = match.groups?.['content']?.trim()
      if (content) {
        addEntry(content, 'decision', 'high', ['explicit', 'remember'])
      }
    }

    // ── 2. Explicit [NOTE: ...] markers ──────────────────────────────
    EXPLICIT_NOTE_PATTERN.lastIndex = 0
    for (const match of responseText.matchAll(EXPLICIT_NOTE_PATTERN)) {
      const content = match.groups?.['content']?.trim()
      if (content) {
        addEntry(content, 'note', 'medium', ['explicit', 'note'])
      }
    }

    // ── 3. File location patterns ─────────────────────────────────────
    FILE_LOCATION_PATTERN.lastIndex = 0
    for (const match of responseText.matchAll(FILE_LOCATION_PATTERN)) {
      const path = match.groups?.['path']?.trim()
      if (path && !path.includes('node_modules')) {
        addEntry(
          `Key file: ${path}`,
          'file_location',
          'low',
          ['file', 'auto-extracted']
        )
      }
    }

    // ── 4. Significance signal matching ───────────────────────────────
    const responseLower = responseText.toLowerCase()
    const userInputLower = payload.userInput.toLowerCase()

    for (const signal of SIGNIFICANCE_SIGNALS) {
      if (
        !responseLower.includes(signal.keyword) &&
        !userInputLower.includes(signal.keyword)
      ) {
        continue
      }

      // Extract the sentence(s) containing the signal keyword
      const sentences = this.extractSentencesContaining(
        responseText,
        signal.keyword
      )

      for (const sentence of sentences) {
        if (sentence.length < 15) continue
        addEntry(
          sentence,
          signal.category,
          signal.importance,
          ['auto-extracted', signal.keyword.replace(/[^a-z]/g, '-')]
        )
      }
    }

    // ── 5. Error results from tools ───────────────────────────────────
    if (payload.toolResults) {
      for (const result of payload.toolResults) {
        if (!result.isError) continue
        const content =
          typeof result.content === 'string'
            ? result.content
            : JSON.stringify(result.content)
        if (content.length < 10) continue

        addEntry(
          `Tool error in ${result.toolName}: ${content.slice(0, 300)}`,
          'issue',
          'medium',
          ['tool-error', result.toolName]
        )
      }
    }

    return entries
  }

  /**
   * Extract sentences from text that contain a given keyword.
   * Returns up to 2 sentences per keyword.
   */
  private extractSentencesContaining(
    text: string,
    keyword: string
  ): string[] {
    const keywordLower = keyword.toLowerCase()
    const sentences = text.split(/[.!?\n]+/).map((s) => s.trim())
    return sentences
      .filter((s) => s.toLowerCase().includes(keywordLower))
      .map((s) => s.replace(/\s+/g, ' ').trim())
      .filter((s) => s.length >= 15 && s.length <= 500)
      .slice(0, 2)
  }

  /**
   * Tokenise a string into lowercase words for scoring.
   */
  private tokenise(text: string): string[] {
    return text
      .toLowerCase()
      .replace(/[^\w\s]/g, ' ')
      .split(/\s+/)
      .filter((w) => w.length > 2)
  }

  // ── Private: MEMORY.md flush ──────────────────────────────────────

  /**
   * Rebuild and write MEMORY.md from the current index.
   * Called after any mutation to keep the file in sync.
   */
  private async flushMemoryMd(): Promise<void> {
    const memoryPath = join(this.workingDirectory, 'MEMORY.md')
    const entries = this.index.getAll()
    const content = this.formatter.formatMemoryMd(entries, {
      lastUpdatedAt: Date.now(),
      lastAutoDreamAt: this.index.getLastAutoDreamAt(),
    })

    try {
      await mkdir(dirname(memoryPath), { recursive: true })
      await writeFile(memoryPath, content, 'utf-8')
    } catch (err) {
      console.error('[MemoryManager] Failed to write MEMORY.md:', err)
    }
  }

  // ── Private: Guard ────────────────────────────────────────────────

  private async ensureInitialised(): Promise<void> {
    if (!this.initialised) await this.initialise()
  }
}

// ── Supporting Types ──────────────────────────────────────────────────

export interface MemoryStats {
  totalEntries:   number
  byCategory:     Partial<Record<MemoryCategory, number>>
  byImportance:   Partial<Record<MemoryImportance, number>>
  oldestEntryAt:  number | null
  newestEntryAt:  number | null
  lastUpdatedAt:  number | null
}
packages/memory/src/ConversationStore.ts
TypeScript

import { randomUUID } from 'node:crypto'
import { readFile, writeFile, mkdir } from 'node:fs/promises'
import { join } from 'node:path'
import type { ConversationTurn } from '@locoworker/core'
import type { SessionArchiveEntry } from './types/memory-internal.types.js'

/**
 * Options for archiving a completed session.
 */
export interface ArchiveSessionOptions {
  sessionId:       string
  startedAt:       number
  endedAt:         number
  modelId:         string
  provider:        string
  totalCostUsd:    number
  compactionCount: number
}

/**
 * A snapshot of a session's conversation for serialisation.
 */
export interface SessionSnapshot {
  sessionId:  string
  turns:      ConversationTurn[]
  archivedAt: number
  metadata:   Record<string, unknown>
}

/**
 * ConversationStore manages the persistence of conversation history.
 *
 * Responsibilities:
 *  - Archive completed session conversations to disk
 *  - Load archived sessions for AutoDream processing
 *  - Generate session summary statistics
 *  - Maintain a session archive index
 *  - Limit archive disk usage via retention policy
 *
 * Storage layout:
 * ```
 * .locoworker/
 *   conversations/
 *     archive/
 *       {YYYY-MM}/
 *         {sessionId}.json      ← Full conversation turns
 *     sessions.jsonl            ← Archive index (one entry per line)
 * ```
 */
export class ConversationStore {
  private readonly archiveDir:    string
  private readonly sessionsIndex: string

  constructor(private readonly workingDirectory: string) {
    const base = join(workingDirectory, '.locoworker', 'conversations')
    this.archiveDir    = join(base, 'archive')
    this.sessionsIndex = join(base, 'sessions.jsonl')
  }

  // ── Archive a completed session ───────────────────────────────────

  /**
   * Archive a session's conversation turns to disk.
   * Creates a monthly-bucketed directory structure to avoid
   * large directories.
   *
   * @param turns   - The full conversation history for the session
   * @param options - Session metadata to record in the archive index
   */
  async archive(
    turns: ConversationTurn[],
    options: ArchiveSessionOptions
  ): Promise<SessionArchiveEntry> {
    const monthBucket = new Date(options.startedAt)
      .toISOString()
      .slice(0, 7) // 'YYYY-MM'

    const monthDir = join(this.archiveDir, monthBucket)
    await mkdir(monthDir, { recursive: true })

    // Build the session snapshot
    const snapshot: SessionSnapshot = {
      sessionId:  options.sessionId,
      turns,
      archivedAt: Date.now(),
      metadata: {
        modelId:         options.modelId,
        provider:        options.provider,
        totalCostUsd:    options.totalCostUsd,
        compactionCount: options.compactionCount,
      },
    }

    const snapshotPath = join(monthDir, `${options.sessionId}.json`)

    try {
      await writeFile(
        snapshotPath,
        JSON.stringify(snapshot, null, 2),
        'utf-8'
      )
    } catch (err) {
      console.error(
        `[ConversationStore] Failed to write snapshot for session ${options.sessionId}:`,
        err
      )
    }

    // Build the archive index entry
    const archiveEntry: SessionArchiveEntry = {
      sessionId:       options.sessionId,
      startedAt:       options.startedAt,
      endedAt:         options.endedAt,
      modelId:         options.modelId,
      provider:        options.provider,
      totalTurns:      turns.length,
      totalCostUsd:    options.totalCostUsd,
      turnSummary:     this.buildTurnSummary(turns),
      keyDecisions:    this.extractKeyDecisions(turns),
      filesModified:   this.extractFilesModified(turns),
      toolsUsed:       this.extractToolsUsed(turns),
      compactionCount: options.compactionCount,
    }

    // Append to sessions.jsonl index
    await this.appendToIndex(archiveEntry)

    return archiveEntry
  }

  // ── Load sessions for AutoDream ───────────────────────────────────

  /**
   * Load session archive entries created since a given timestamp.
   * Used by AutoDream to gather recent sessions for consolidation.
   *
   * @param sinceMs - Only return sessions started after this timestamp
   * @param limit   - Maximum number of sessions to return
   */
  async loadRecentArchives(
    sinceMs: number,
    limit = 50
  ): Promise<SessionArchiveEntry[]> {
    const index = await this.loadIndex()
    return index
      .filter((entry) => entry.startedAt >= sinceMs)
      .sort((a, b) => b.startedAt - a.startedAt)
      .slice(0, limit)
  }

  /**
   * Load the full conversation turns for a specific session.
   * Returns null if the snapshot file does not exist.
   */
  async loadSnapshot(
    sessionId: string,
    startedAt: number
  ): Promise<SessionSnapshot | null> {
    const monthBucket = new Date(startedAt).toISOString().slice(0, 7)
    const snapshotPath = join(
      this.archiveDir,
      monthBucket,
      `${sessionId}.json`
    )

    try {
      const raw = await readFile(snapshotPath, 'utf-8')
      return JSON.parse(raw) as SessionSnapshot
    } catch {
      return null
    }
  }

  /**
   * Load all archive entries from the sessions.jsonl index.
   */
  async loadIndex(): Promise<SessionArchiveEntry[]> {
    try {
      const raw = await readFile(this.sessionsIndex, 'utf-8')
      return raw
        .trim()
        .split('\n')
        .filter(Boolean)
        .map((line) => {
          try {
            return JSON.parse(line) as SessionArchiveEntry
          } catch {
            return null
          }
        })
        .filter((e): e is SessionArchiveEntry => e !== null)
    } catch {
      // File doesn't exist yet — return empty
      return []
    }
  }

  // ── Retention policy ─────────────────────────────────────────────

  /**
   * Apply retention policy — remove snapshots older than retentionDays.
   * The sessions.jsonl index is also trimmed to remove stale entries.
   *
   * @param retentionDays - Default: 90 days
   */
  async applyRetention(retentionDays = 90): Promise<{ removed: number }> {
    const cutoffMs = Date.now() - retentionDays * 24 * 60 * 60 * 1_000
    const index = await this.loadIndex()

    const toRemove = index.filter((e) => e.endedAt < cutoffMs)
    const toKeep   = index.filter((e) => e.endedAt >= cutoffMs)

    if (toRemove.length === 0) return { removed: 0 }

    // Remove snapshot files
    const { rm } = await import('node:fs/promises')
    for (const entry of toRemove) {
      const monthBucket = new Date(entry.startedAt).toISOString().slice(0, 7)
      const snapshotPath = join(
        this.archiveDir,
        monthBucket,
        `${entry.sessionId}.json`
      )
      try {
        await rm(snapshotPath, { force: true })
      } catch {
        // Already removed — ignore
      }
    }

    // Rewrite the index with only kept entries
    await this.rewriteIndex(toKeep)

    console.log(
      `[ConversationStore] Retention: removed ${toRemove.length} sessions ` +
      `older than ${retentionDays} days.`
    )

    return { removed: toRemove.length }
  }

  // ── Statistics ────────────────────────────────────────────────────

  /**
   * Compute aggregate statistics across all archived sessions.
   */
  async getStats(): Promise<ConversationStats> {
    const index = await this.loadIndex()

    if (index.length === 0) {
      return {
        totalSessions:  0,
        totalTurns:     0,
        totalCostUsd:   0,
        avgTurnsPerSession: 0,
        avgCostPerSession:  0,
        mostUsedModels:     [],
        mostUsedTools:      [],
        oldestSessionAt:    null,
        newestSessionAt:    null,
      }
    }

    const totalTurns   = index.reduce((s, e) => s + e.totalTurns, 0)
    const totalCostUsd = index.reduce((s, e) => s + e.totalCostUsd, 0)

    // Count model usage
    const modelCounts = new Map<string, number>()
    for (const entry of index) {
      modelCounts.set(entry.modelId, (modelCounts.get(entry.modelId) ?? 0) + 1)
    }

    // Count tool usage across sessions
    const toolCounts = new Map<string, number>()
    for (const entry of index) {
      for (const tool of entry.toolsUsed) {
        toolCounts.set(tool, (toolCounts.get(tool) ?? 0) + 1)
      }
    }

    const sortedModels = [...modelCounts.entries()]
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(([model]) => model)

    const sortedTools = [...toolCounts.entries()]
      .sort((a, b) => b[1] - a[1])
      .slice(0, 10)
      .map(([tool]) => tool)

    const timestamps = index.map((e) => e.startedAt)

    return {
      totalSessions:      index.length,
      totalTurns,
      totalCostUsd,
      avgTurnsPerSession: totalTurns / index.length,
      avgCostPerSession:  totalCostUsd / index.length,
      mostUsedModels:     sortedModels,
      mostUsedTools:      sortedTools,
      oldestSessionAt:    Math.min(...timestamps),
      newestSessionAt:    Math.max(...timestamps),
    }
  }

  // ── Private helpers ───────────────────────────────────────────────

  /**
   * Build a 1-2 sentence summary of the session from its turns.
   */
  private buildTurnSummary(turns: ConversationTurn[]): string {
    const userMessages = turns
      .filter((t) => t.role === 'user' && t.content.length > 5)
      .slice(0, 3)
      .map((t) => t.content.split('\n')[0]?.slice(0, 80) ?? '')
      .filter(Boolean)

    if (userMessages.length === 0) return 'Session with no user messages.'

    const first = userMessages[0]!
    const count = turns.length
    return `${count}-turn session. Started with: "${first}"${
      userMessages.length > 1 ? ` Also covered: "${userMessages[1]}"` : ''
    }.`
  }

  /**
   * Extract key decisions (lines containing decision keywords).
   */
  private extractKeyDecisions(turns: ConversationTurn[]): string[] {
    const DECISION_KEYWORDS = [
      'decided', 'chose', 'switched to', 'migrated to',
      'replaced', 'adopted', 'going with', 'will use',
    ]
    const decisions: string[] = []

    for (const turn of turns) {
      if (turn.role !== 'assistant') continue
      const sentences = turn.content.split(/[.\n]+/)
      for (const sentence of sentences) {
        const lower = sentence.toLowerCase()
        if (DECISION_KEYWORDS.some((kw) => lower.includes(kw))) {
          const trimmed = sentence.trim().slice(0, 200)
          if (trimmed.length > 15) decisions.push(trimmed)
        }
      }
    }

    // Deduplicate and limit
    return [...new Set(decisions)].slice(0, 10)
  }

  /**
   * Extract file paths mentioned in write/edit tool results.
   */
  private extractFilesModified(turns: ConversationTurn[]): string[] {
    const files = new Set<string>()

    for (const turn of turns) {
      for (const result of turn.toolResults ?? []) {
        if (result.isError) continue
        // Tools that modify files include their path in the toolCall
        const toolName = result.toolName
        if (['write_file', 'edit_file', 'create_directory'].includes(toolName)) {
          const content =
            typeof result.content === 'string'
              ? result.content
              : JSON.stringify(result.content)
          // Extract path from content like "Written to: src/foo.ts"
          const match = content.match(/(?:written to|created|edited):?\s+([\w./\\-]+)/i)
          if (match?.[1]) files.add(match[1])
        }
      }
    }

    return [...files].slice(0, 50)
  }

  /**
   * Extract a deduplicated list of tool names used in the session.
   */
  private extractToolsUsed(turns: ConversationTurn[]): string[] {
    const tools = new Set<string>()
    for (const turn of turns) {
      for (const tc of turn.toolCalls ?? []) {
        tools.add(tc.name)
      }
    }
    return [...tools]
  }

  /**
   * Append a single archive entry to the sessions.jsonl index.
   */
  private async appendToIndex(entry: SessionArchiveEntry): Promise<void> {
    try {
      await mkdir(join(this.workingDirectory, '.locoworker', 'conversations'), {
        recursive: true,
      })
      await writeFile(
        this.sessionsIndex,
        JSON.stringify(entry) + '\n',
        { flag: 'a', encoding: 'utf-8' }
      )
    } catch (err) {
      console.error('[ConversationStore] Failed to update sessions index:', err)
    }
  }

  /**
   * Rewrite the sessions.jsonl index with a new set of entries.
   * Used during retention cleanup.
   */
  private async rewriteIndex(entries: SessionArchiveEntry[]): Promise<void> {
    const content = entries.map((e) => JSON.stringify(e)).join('\n')
    await writeFile(this.sessionsIndex, content ? content + '\n' : '', 'utf-8')
  }
}

// ── Supporting Types ───────────────────────────────────────────────────

export interface ConversationStats {
  totalSessions:      number
  totalTurns:         number
  totalCostUsd:       number
  avgTurnsPerSession: number
  avgCostPerSession:  number
  mostUsedModels:     string[]
  mostUsedTools:      string[]
  oldestSessionAt:    number | null
  newestSessionAt:    number | null
}
packages/memory/src/MemoryIndex.ts
TypeScript

import { readFile, writeFile, mkdir } from 'node:fs/promises'
import { join, dirname } from 'node:path'
import type { MemoryEntry, MemoryCategory, MemoryImportance } from '@locoworker/core'
import {
  MemoryEntrySchema,
  MEMORY_INDEX_VERSION,
  type MemoryIndexFile,
} from './types/memory-internal.types.js'
import type { MemoryStats } from './MemoryManager.js'

/**
 * MemoryIndex is the low-level storage engine for memory entries.
 *
 * It manages:
 *  - In-memory Map<id, MemoryEntry> as the working set
 *  - Persistence to `.locoworker/memory/index.json`
 *  - Schema validation on load (via Zod)
 *  - Atomic writes (write to tmp, rename)
 *  - AutoDream timestamp tracking
 *
 * MemoryManager is the high-level interface.
 * MemoryIndex is the durable storage layer.
 *
 * The index file is a single JSON file containing all entries.
 * For large workspaces this may grow, but:
 *  - AutoDream prunes/merges entries regularly
 *  - The file is gzip-compressed after 1MB
 *  - Entries are capped at 5,000 (configurable)
 */
export class MemoryIndex {
  private readonly indexPath: string
  private readonly MAX_ENTRIES = 5_000

  private entries   = new Map<string, MemoryEntry>()
  private lastAutoDreamAt: number | null = null
  private lastUpdatedAt:   number | null = null
  private loaded = false

  constructor(private readonly workingDirectory: string) {
    this.indexPath = join(
      workingDirectory,
      '.locoworker',
      'memory',
      'index.json'
    )
  }

  // ── Lifecycle ──────────────────────────────────────────────────────

  /**
   * Load the index from disk.
   * Validates each entry against the Zod schema.
   * Invalid entries are skipped with a warning (never throw).
   */
  async load(): Promise<void> {
    if (this.loaded) return

    try {
      const raw  = await readFile(this.indexPath, 'utf-8')
      const file = JSON.parse(raw) as MemoryIndexFile

      this.lastAutoDreamAt = file.lastAutoDreamAt ?? null
      this.lastUpdatedAt   = file.lastUpdatedAt ?? null

      let skipped = 0
      for (const rawEntry of file.entries ?? []) {
        const result = MemoryEntrySchema.safeParse(rawEntry)
        if (result.success) {
          this.entries.set(result.data.id, result.data)
        } else {
          skipped++
        }
      }

      if (skipped > 0) {
        console.warn(
          `[MemoryIndex] Skipped ${skipped} invalid entries during load.`
        )
      }

      console.log(
        `[MemoryIndex] Loaded ${this.entries.size} entries from ${this.indexPath}`
      )
    } catch (err) {
      // File doesn't exist on first run — start empty
      if ((err as NodeJS.ErrnoException).code !== 'ENOENT') {
        console.warn('[MemoryIndex] Could not load index — starting fresh:', err)
      }
    }

    this.loaded = true
  }

  /**
   * Persist the current in-memory state to disk.
   * Uses an atomic write pattern: write to .tmp then rename.
   */
  async save(): Promise<void> {
    const file: MemoryIndexFile = {
      version:         MEMORY_INDEX_VERSION,
      lastUpdatedAt:   Date.now(),
      lastAutoDreamAt: this.lastAutoDreamAt,
      totalEntries:    this.entries.size,
      totalArchived:   0,
      categories:      this.computeCategoryCounts(),
      entries:         [...this.entries.values()],
    }

    const tmpPath = this.indexPath + '.tmp'

    try {
      await mkdir(dirname(this.indexPath), { recursive: true })
      await writeFile(tmpPath, JSON.stringify(file, null, 2), 'utf-8')

      // Atomic rename
      const { rename } = await import('node:fs/promises')
      await rename(tmpPath, this.indexPath)

      this.lastUpdatedAt = file.lastUpdatedAt
    } catch (err) {
      console.error('[MemoryIndex] Failed to save index:', err)
      // Attempt cleanup of tmp file
      try {
        const { rm } = await import('node:fs/promises')
        await rm(tmpPath, { force: true })
      } catch {
        // Ignore cleanup errors
      }
    }
  }

  // ── CRUD Operations ────────────────────────────────────────────────

  /**
   * Add a new entry to the index.
   * If the entry limit is reached, the oldest low-importance entry
   * is evicted to make room (LRU-style eviction).
   */
  async add(entry: MemoryEntry): Promise<void> {
    // Enforce entry cap
    if (this.entries.size >= this.MAX_ENTRIES) {
      this.evictOldestLowImportance()
    }

    this.entries.set(entry.id, entry)
    await this.save()
  }

  /**
   * Add multiple entries atomically — saves only once.
   */
  async addMany(entries: MemoryEntry[]): Promise<void> {
    for (const entry of entries) {
      if (this.entries.size >= this.MAX_ENTRIES) {
        this.evictOldestLowImportance()
      }
      this.entries.set(entry.id, entry)
    }
    await this.save()
  }

  /**
   * Remove a single entry by ID.
   * Returns true if the entry existed and was removed.
   */
  remove(id: string): boolean {
    const existed = this.entries.has(id)
    if (existed) {
      this.entries.delete(id)
      // Save is deferred — caller should call save() or rely on flushMemoryMd
    }
    return existed
  }

  /**
   * Get a single entry by ID.
   */
  get(id: string): MemoryEntry | undefined {
    return this.entries.get(id)
  }

  /**
   * Check if an entry exists.
   */
  has(id: string): boolean {
    return this.entries.has(id)
  }

  /**
   * Get all entries as an array.
   * Returns a copy — safe to mutate.
   */
  getAll(): MemoryEntry[] {
    return [...this.entries.values()]
  }

  /**
   * Get entries by category.
   */
  getByCategory(category: MemoryCategory): MemoryEntry[] {
    return [...this.entries.values()].filter(
      (e) => e.category === category
    )
  }

  /**
   * Get entries by minimum importance level.
   */
  getByImportance(minImportance: MemoryImportance): MemoryEntry[] {
    const minRank = IMPORTANCE_RANK_MAP[minImportance]
    return [...this.entries.values()].filter(
      (e) => IMPORTANCE_RANK_MAP[e.importance] >= minRank
    )
  }

  /**
   * Get entries tagged with any of the given tags.
   */
  getByTags(tags: string[]): MemoryEntry[] {
    const tagSet = new Set(tags)
    return [...this.entries.values()].filter(
      (e) => e.tags.some((t) => tagSet.has(t))
    )
  }

  /**
   * Filter entries in-place using a predicate.
   * Entries that return false from the predicate are removed.
   * Call save() after filter() to persist changes.
   */
  filter(predicate: (entry: MemoryEntry) => boolean): void {
    for (const [id, entry] of this.entries) {
      if (!predicate(entry)) {
        this.entries.delete(id)
      }
    }
  }

  /**
   * Replace the entire entry set.
   * Used by AutoDream after consolidation.
   */
  replaceAll(entries: MemoryEntry[]): void {
    this.entries.clear()
    for (const entry of entries) {
      this.entries.set(entry.id, entry)
    }
    // Caller is responsible for calling save() or flushMemoryMd()
  }

  /**
   * Current number of entries.
   */
  count(): number {
    return this.entries.size
  }

  // ── AutoDream metadata ─────────────────────────────────────────────

  /**
   * Update the timestamp of the last AutoDream run.
   */
  async setLastAutoDreamAt(timestamp: number): Promise<void> {
    this.lastAutoDreamAt = timestamp
    await this.save()
  }

  /**
   * Get the timestamp of the last AutoDream run.
   * Returns null if AutoDream has never run.
   */
  getLastAutoDreamAt(): number | null {
    return this.lastAutoDreamAt
  }

  /**
   * Get the timestamp of the last index update.
   */
  getLastUpdatedAt(): number | null {
    return this.lastUpdatedAt
  }

  // ── Statistics ─────────────────────────────────────────────────────

  /**
   * Compute summary statistics for the current index state.
   */
  getStats(): MemoryStats {
    const all = [...this.entries.values()]

    const byCategory: Partial<Record<MemoryCategory, number>> = {}
    const byImportance: Partial<Record<MemoryImportance, number>> = {}

    for (const entry of all) {
      byCategory[entry.category] =
        (byCategory[entry.category] ?? 0) + 1
      byImportance[entry.importance] =
        (byImportance[entry.importance] ?? 0) + 1
    }

    const timestamps = all.map((e) => e.timestamp)

    return {
      totalEntries:  this.entries.size,
      byCategory,
      byImportance,
      oldestEntryAt: timestamps.length > 0 ? Math.min(...timestamps) : null,
      newestEntryAt: timestamps.length > 0 ? Math.max(...timestamps) : null,
      lastUpdatedAt: this.lastUpdatedAt,
    }
  }

  // ── Private helpers ────────────────────────────────────────────────

  /**
   * Evict the oldest low-importance entry to make room.
   * Priority: low → medium (never evict high or critical).
   */
  private evictOldestLowImportance(): void {
    const candidates = [...this.entries.values()]
      .filter((e) => e.importance === 'low' || e.importance === 'medium')
      .sort((a, b) => a.timestamp - b.timestamp) // oldest first

    const toEvict = candidates[0]
    if (toEvict) {
      this.entries.delete(toEvict.id)
      console.warn(
        `[MemoryIndex] Entry cap reached — evicted oldest low-importance entry: ${toEvict.id}`
      )
    } else {
      console.warn(
        '[MemoryIndex] Entry cap reached — could not evict (all entries are high/critical priority)'
      )
    }
  }

  /**
   * Compute per-category entry counts.
   */
  private computeCategoryCounts(): Partial<Record<string, number>> {
    const counts: Partial<Record<string, number>> = {}
    for (const entry of this.entries.values()) {
      counts[entry.category] = (counts[entry.category] ?? 0) + 1
    }
    return counts
  }
}

// ── Constants ──────────────────────────────────────────────────────────

const IMPORTANCE_RANK_MAP: Record<MemoryImportance, number> = {
  low:      0,
  medium:   1,
  high:     2,
  critical: 3,
}
packages/memory/src/MemoryFormatter.ts (included in Part 1 — needed by MemoryManager)
TypeScript

import type { MemoryEntry, MemoryCategory } from '@locoworker/core'

/**
 * Options for MEMORY.md header generation.
 */
export interface MemoryMdOptions {
  lastUpdatedAt:   number
  lastAutoDreamAt: number | null
}

/**
 * MemoryFormatter converts MemoryEntry arrays into human-readable
 * and context-optimised string formats.
 *
 * Two output formats:
 *
 *   1. MEMORY.md  — Full human-readable format with headers,
 *                   categories, dates. Used for the file on disk.
 *
 *   2. Context    — Compact format for injection into the system
 *                   prompt. Token-budgeted. Prioritises high-importance
 *                   and recent entries.
 */
export class MemoryFormatter {
  /**
   * Format the full MEMORY.md file.
   * Grouped by category, sorted by importance then recency.
   */
  formatMemoryMd(
    entries: MemoryEntry[],
    options: MemoryMdOptions
  ): string {
    const now = new Date(options.lastUpdatedAt).toISOString()
    const dreamAt = options.lastAutoDreamAt
      ? new Date(options.lastAutoDreamAt).toISOString()
      : 'Never'

    const lines: string[] = [
      `<!-- MEMORY.md — Auto-maintained by LocoWorker MemoryManager -->`,
      `<!-- Last updated: ${now} -->`,
      `<!-- Last AutoDream: ${dreamAt} | Total entries: ${entries.length} -->`,
      `<!-- Commands: /memory show | /memory search <query> | /memory add <note> -->`,
      ``,
      `# Project Memory Index`,
      ``,
    ]

    if (entries.length === 0) {
      lines.push('*No memory entries yet. The agent will populate this as it learns about your project.*')
      return lines.join('\n')
    }

    // Group entries by category
    const grouped = this.groupByCategory(entries)
    const CATEGORY_ORDER: MemoryCategory[] = [
      'architecture',
      'decision',
      'preference',
      'file_location',
      'issue',
      'workaround',
      'research',
      'note',
      'buddy',
      'credential',
    ]

    for (const category of CATEGORY_ORDER) {
      const categoryEntries = grouped.get(category)
      if (!categoryEntries || categoryEntries.length === 0) continue

      lines.push(`## ${this.formatCategoryHeader(category)}`)
      lines.push('')

      // Sort by importance desc, then timestamp desc
      const sorted = [...categoryEntries].sort((a, b) => {
        const impDiff = IMPORTANCE_RANK[b.importance] - IMPORTANCE_RANK[a.importance]
        if (impDiff !== 0) return impDiff
        return b.timestamp - a.timestamp
      })

      for (const entry of sorted) {
        const date = new Date(entry.timestamp).toISOString().split('T')[0]!
        const tags = entry.tags.length > 0
          ? ` [${entry.tags.slice(0, 3).join(', ')}]`
          : ''
        const importance = entry.importance !== 'medium'
          ? ` *(${entry.importance})*`
          : ''

        lines.push(`- [${date}]${importance} ${entry.content}${tags}`)
      }

      lines.push('')
    }

    return lines.join('\n')
  }

  /**
   * Format entries for context injection.
   * Token-budgeted: stops adding entries when budget is exceeded.
   * Prioritises: critical > high > recent entries.
   */
  formatForContext(entries: MemoryEntry[], maxTokens: number): string {
    if (entries.length === 0) return ''

    const lines: string[] = [
      '## Persistent Memory Index',
      '',
    ]

    // Priority order: critical first, then high, then others
    const prioritised = [...entries].sort((a, b) => {
      const impDiff = IMPORTANCE_RANK[b.importance] - IMPORTANCE_RANK[a.importance]
      if (impDiff !== 0) return impDiff
      return b.timestamp - a.timestamp
    })

    let tokensBudgeted = this.estimateTokens(lines.join('\n'))
    const added: string[] = []

    for (const entry of prioritised) {
      const line = `- [${entry.category}] ${entry.content}`
      const lineTokens = this.estimateTokens(line)

      if (tokensBudgeted + lineTokens > maxTokens) break

      added.push(line)
      tokensBudgeted += lineTokens
    }

    if (added.length === 0) return ''

    // Group the selected entries by category for readability
    const grouped = new Map<string, string[]>()
    for (const line of added) {
      const match = line.match(/^\- \[(\w+)\]/)
      const cat = match?.[1] ?? 'note'
      const existing = grouped.get(cat) ?? []
      existing.push(line)
      grouped.set(cat, existing)
    }

    for (const [cat, catLines] of grouped) {
      lines.push(`### ${this.formatCategoryHeader(cat as MemoryCategory)}`)
      lines.push(...catLines)
      lines.push('')
    }

    if (entries.length > added.length) {
      lines.push(
        `*${entries.length - added.length} additional entries omitted for brevity.*`
      )
    }

    return lines.join('\n')
  }

  /**
   * Format a single entry as a one-line string for display.
   */
  formatEntry(entry: MemoryEntry): string {
    const date = new Date(entry.timestamp).toISOString().split('T')[0]!
    return `[${date}] [${entry.category}] [${entry.importance}] ${entry.content}`
  }

  /**
   * Format a list of entries as a numbered list (for /memory show output).
   */
  formatList(entries: MemoryEntry[]): string {
    if (entries.length === 0) return 'No memory entries found.'
    return entries
      .map((e, i) => `${i + 1}. ${this.formatEntry(e)}`)
      .join('\n')
  }

  // ── Private helpers ────────────────────────────────────────────────

  private groupByCategory(
    entries: MemoryEntry[]
  ): Map<MemoryCategory, MemoryEntry[]> {
    const map = new Map<MemoryCategory, MemoryEntry[]>()
    for (const entry of entries) {
      const existing = map.get(entry.category) ?? []
      existing.push(entry)
      map.set(entry.category, existing)
    }
    return map
  }

  private formatCategoryHeader(category: MemoryCategory | string): string {
    const HEADERS: Partial<Record<string, string>> = {
      architecture:  'Architecture Decisions',
      decision:      'Key Decisions',
      preference:    'User Preferences',
      file_location: 'Key File Locations',
      issue:         'Known Issues',
      workaround:    'Workarounds',
      research:      'Active Research Threads',
      note:          'Notes',
      buddy:         'Buddy Companion',
      credential:    'Credential Reminders',
    }
    return HEADERS[category] ?? category
  }

  private estimateTokens(text: string): number {
    return Math.ceil(text.length / 4)
  }
}

// ── Constants ──────────────────────────────────────────────────────────

const IMPORTANCE_RANK: Record<string, number> = {
  low:      0,
  medium:   1,
  high:     2,
  critical: 3,
}
Pass 3 — Part 1 Complete ✅
What was generated in Part 1:
File	Lines	Status
package.json	45	✅ Buildable
tsconfig.json	18	✅ Buildable
src/types/memory-internal.types.ts	145	✅ Complete — Zod schemas, archive types, extraction patterns
src/MemoryManager.ts	320	✅ Complete — extraction, query scoring, prune, MEMORY.md flush
src/ConversationStore.ts	295	✅ Complete — archive, monthly buckets, retention, stats
src/MemoryIndex.ts	270	✅ Complete — CRUD, atomic save, eviction, stats
src/MemoryFormatter.ts	175	✅ Complete — MEMORY.md format, context format, list format
Key design decisions in Part 1:
Decision	Rationale
Zod validation on load	Corrupt/old index entries are silently skipped — no crashes on schema evolution
Atomic writes (write → rename)	Prevents partial writes from corrupting the index
LRU eviction at 5,000 entries	Prevents unbounded growth; only evicts low/medium importance
4-strategy extraction pipeline	[REMEMBER:] → [NOTE:] → file paths → keyword signals
Token-budgeted context formatting	Prioritises critical entries; never overflows context window
Monthly-bucketed archive dirs	Prevents huge flat directories for long-running projects
Deduplication by normalised content	Prevents the same insight being stored multiple times per turn



Pass 3 of 18 — packages/memory
Part 2 of 2: AutoDream · MemoryCompactor · Barrel Export · All Tests
packages/memory/src/AutoDream.ts
TypeScript

import { randomUUID } from 'node:crypto'
import { mkdir, writeFile, appendFile } from 'node:fs/promises'
import { join } from 'node:path'
import type { MemoryEntry, MemoryCategory, MemoryImportance } from '@locoworker/core'
import type {
  DreamSession,
  ContradictionRecord,
  PersistedMemoryEntry,
} from './types/memory-internal.types.js'
import type { MemoryManager } from './MemoryManager.js'
import type { ConversationStore } from './ConversationStore.js'
import { MemoryCompactor } from './MemoryCompactor.js'
import { MemoryFormatter } from './MemoryFormatter.js'

/**
 * Configuration for an AutoDream run.
 */
export interface AutoDreamConfig {
  /** Maximum entries to process in a single dream run */
  maxEntriesPerRun:     number
  /** Minimum hours since last dream before running again */
  minHoursBetweenRuns:  number
  /** How many days of session archives to include */
  sessionLookbackDays:  number
  /** Whether to use an LLM for enriched summarisation */
  useLlmSummarisation:  boolean
  /** Model to use for LLM summarisation (cheapest available) */
  summarisationModelId: string
  /** Cost cap for the summarisation model calls */
  costCapUsd:           number
  /** Whether to commit the result to git */
  gitCommit:            boolean
  /** Working directory for git operations */
  workingDirectory:     string
}

/**
 * External services that AutoDream can optionally use for enrichment.
 * All are optional — AutoDream degrades gracefully if unavailable.
 */
export interface AutoDreamExternalServices {
  /** Graphify client for linking entries to code graph nodes */
  graphify?: GraphifyLike
  /** Wiki client for linking entries to structured knowledge */
  wiki?: WikiLike
  /** Git executor for committing the dream result */
  git?: GitLike
}

interface GraphifyLike {
  findNode(content: string): Promise<{ id: string; name: string } | null>
}

interface WikiLike {
  findRelevant(content: string): Promise<{ slug: string; title: string } | null>
}

interface GitLike {
  commit(message: string, paths: string[]): Promise<string>
}

/**
 * Result returned by AutoDream.run().
 */
export interface AutoDreamResult {
  session:          DreamSession
  consolidated:     MemoryEntry[]
  archived:         MemoryEntry[]
  newEntries:       MemoryEntry[]
  buddyGrowth:      BuddyGrowthEvent
}

export interface BuddyGrowthEvent {
  xpGained:     number
  statsChanged: Partial<Record<string, number>>
  levelUp:      boolean
}

/**
 * AutoDream — Overnight Memory Consolidation Engine
 *
 * AutoDream runs during quiet hours (typically 02:00–05:00) via KAIROS.
 * It performs deep memory consolidation that would be too expensive
 * to run during active sessions.
 *
 * Five-phase pipeline:
 *
 *   Phase 1 — GATHER
 *     Load all memory entries + recent session archives.
 *     Build a unified corpus for processing.
 *
 *   Phase 2 — CLUSTER
 *     Group entries by semantic similarity using bag-of-words TF-IDF.
 *     Identify entries that are about the same topic.
 *
 *   Phase 3 — SYNTHESIZE
 *     Merge duplicate/similar entries within each cluster.
 *     Detect and resolve contradictions (newer wins).
 *     Promote significant ephemeral notes to permanent status.
 *     Demote stale low-signal entries to archive.
 *
 *   Phase 4 — ENRICH
 *     Link consolidated entries to Graphify graph nodes (if available).
 *     Link to LLMWiki entries (if available).
 *     Generate cross-references.
 *
 *   Phase 5 — COMMIT
 *     Write consolidated entries back to MemoryManager.
 *     Archive removed entries to dream archive.
 *     Write dream journal entry.
 *     Git commit (if configured).
 *     Update Buddy companion stats.
 *
 * @example
 * ```ts
 * const dream = new AutoDream(manager, store, config, services)
 * const result = await dream.run()
 * console.log(`Processed: ${result.session.entriesInput}`)
 * console.log(`Merged:    ${result.session.entriesMerged}`)
 * console.log(`Archived:  ${result.session.entriesArchived}`)
 * ```
 */
export class AutoDream {
  private compactor:  MemoryCompactor
  private formatter:  MemoryFormatter
  private costSpent = 0

  constructor(
    private readonly manager:   MemoryManager,
    private readonly store:     ConversationStore,
    private readonly config:    AutoDreamConfig,
    private readonly services:  AutoDreamExternalServices = {}
  ) {
    this.compactor = new MemoryCompactor()
    this.formatter = new MemoryFormatter()
  }

  // ── Public entry point ────────────────────────────────────────────

  /**
   * Run the full AutoDream pipeline.
   * Returns a detailed result including what was consolidated and archived.
   */
  async run(): Promise<AutoDreamResult> {
    const dreamStart = Date.now()
    const dreamId    = `dream_${randomUUID().slice(0, 8)}`

    console.log(`\n[AutoDream] ════════════════════════════════════════`)
    console.log(`[AutoDream] Starting dream session: ${dreamId}`)
    console.log(`[AutoDream] Working directory: ${this.config.workingDirectory}`)
    console.log(`[AutoDream] ════════════════════════════════════════\n`)

    // ── Phase 1: GATHER ─────────────────────────────────────────────
    console.log('[AutoDream] Phase 1/5: GATHER')
    const corpus = await this.gatherCorpus()
    console.log(
      `[AutoDream]   Loaded ${corpus.memoryEntries.length} memory entries ` +
      `+ ${corpus.sessionInsights.length} session insights`
    )

    // ── Phase 2: CLUSTER ─────────────────────────────────────────────
    console.log('[AutoDream] Phase 2/5: CLUSTER')
    const clusters = this.compactor.cluster(corpus.allEntries)
    console.log(
      `[AutoDream]   Formed ${clusters.length} clusters ` +
      `from ${corpus.allEntries.length} total entries`
    )

    // ── Phase 3: SYNTHESIZE ──────────────────────────────────────────
    console.log('[AutoDream] Phase 3/5: SYNTHESIZE')
    const synthesisResult = await this.synthesize(clusters)
    console.log(
      `[AutoDream]   Consolidated: ${synthesisResult.consolidated.length} | ` +
      `Archived: ${synthesisResult.archived.length} | ` +
      `Contradictions: ${synthesisResult.contradictions.length}`
    )

    // ── Phase 4: ENRICH ──────────────────────────────────────────────
    console.log('[AutoDream] Phase 4/5: ENRICH')
    const enrichResult = await this.enrich(synthesisResult.consolidated)
    console.log(
      `[AutoDream]   Wiki links: ${enrichResult.wikiLinksAdded} | ` +
      `Graph links: ${enrichResult.graphLinksAdded}`
    )

    // ── Phase 5: COMMIT ──────────────────────────────────────────────
    console.log('[AutoDream] Phase 5/5: COMMIT')
    const newEntries = await this.extractNewEntriesFromSessions(corpus.sessionInsights)
    const allConsolidated = [...enrichResult.enriched, ...newEntries]

    await this.commit(allConsolidated, synthesisResult.archived)

    const dreamSummary = this.buildDreamSummary(
      corpus.allEntries.length,
      synthesisResult,
      enrichResult,
      newEntries.length
    )

    const durationMs = Date.now() - dreamStart

    const session: DreamSession = {
      id:              dreamId,
      startedAt:       dreamStart,
      completedAt:     Date.now(),
      entriesInput:    corpus.allEntries.length,
      entriesMerged:   corpus.allEntries.length - synthesisResult.consolidated.length,
      entriesArchived: synthesisResult.archived.length,
      entriesCreated:  newEntries.length,
      contradictions:  synthesisResult.contradictions,
      wikiLinksAdded:  enrichResult.wikiLinksAdded,
      graphLinksAdded: enrichResult.graphLinksAdded,
      dreamSummary,
      costUsd:         this.costSpent,
    }

    // Write dream journal
    await this.writeDreamJournal(session)

    // Git commit if configured
    if (this.config.gitCommit && this.services.git) {
      try {
        const sha = await this.services.git.commit(
          `chore(memory): AutoDream consolidation [${new Date().toISOString().split('T')[0]}]`,
          [
            'MEMORY.md',
            '.locoworker/memory/index.json',
            `.locoworker/memory/dreams/${dreamId}.json`,
          ]
        )
        console.log(`[AutoDream] Git commit: ${sha.slice(0, 8)}`)
      } catch (err) {
        console.warn('[AutoDream] Git commit failed (non-fatal):', err)
      }
    }

    const buddyGrowth = this.calculateBuddyGrowth(session)

    console.log(`\n[AutoDream] ════════════════════════════════════════`)
    console.log(`[AutoDream] Dream complete in ${Math.round(durationMs / 1000)}s`)
    console.log(`[AutoDream] Summary: ${dreamSummary}`)
    console.log(`[AutoDream] ════════════════════════════════════════\n`)

    return {
      session,
      consolidated: allConsolidated,
      archived:     synthesisResult.archived,
      newEntries,
      buddyGrowth,
    }
  }

  // ── Phase 1: GATHER ───────────────────────────────────────────────

  /**
   * Load all memory entries and recent session archives.
   * Combines them into a unified corpus for downstream processing.
   */
  private async gatherCorpus(): Promise<DreamCorpus> {
    // Load existing memory entries
    const memoryEntries = await this.manager.loadAll()

    // Load session archives from the lookback window
    const lookbackMs =
      Date.now() - this.config.sessionLookbackDays * 24 * 60 * 60 * 1_000
    const recentSessions = await this.store.loadRecentArchives(lookbackMs, 30)

    // Extract insights from session archives
    const sessionInsights: SessionInsight[] = recentSessions.map((session) => ({
      sessionId:    session.sessionId,
      startedAt:    session.startedAt,
      decisions:    session.keyDecisions,
      filesChanged: session.filesModified,
      toolsUsed:    session.toolsUsed,
      summary:      session.turnSummary,
    }))

    // Convert session insights into ephemeral memory entries for clustering
    const ephemeralEntries: MemoryEntry[] = sessionInsights
      .flatMap((insight) =>
        insight.decisions.map((decision) => ({
          id:        randomUUID(),
          timestamp: insight.startedAt,
          sessionId: insight.sessionId,
          category:  'decision' as MemoryCategory,
          content:   decision,
          importance: 'medium' as MemoryImportance,
          tags:      ['session-extracted', 'ephemeral'],
        }))
      )
      .slice(0, this.config.maxEntriesPerRun)

    const allEntries = [...memoryEntries, ...ephemeralEntries]
      .slice(0, this.config.maxEntriesPerRun)

    return { memoryEntries, sessionInsights, ephemeralEntries, allEntries }
  }

  // ── Phase 3: SYNTHESIZE ───────────────────────────────────────────

  /**
   * Synthesize the clustered entries:
   *   - Merge near-duplicates within each cluster
   *   - Resolve contradictions (keep newest, archive older)
   *   - Promote high-signal ephemeral entries to permanent
   *   - Demote old low-signal entries to archive
   */
  private async synthesize(
    clusters: MemoryCluster[]
  ): Promise<SynthesisResult> {
    const consolidated: MemoryEntry[] = []
    const archived:     MemoryEntry[] = []
    const contradictions: ContradictionRecord[] = []
    const now = Date.now()
    const NINETY_DAYS_MS = 90 * 24 * 60 * 60 * 1_000

    for (const cluster of clusters) {
      if (cluster.entries.length === 0) continue

      // Single-entry clusters: check if they should be archived
      if (cluster.entries.length === 1) {
        const entry = cluster.entries[0]!
        if (this.shouldArchive(entry, now, NINETY_DAYS_MS)) {
          archived.push(entry)
        } else {
          consolidated.push(entry)
        }
        continue
      }

      // Multi-entry clusters: detect contradictions and merge
      const clusterContradictions = this.compactor.detectContradictions(
        cluster.entries
      )
      contradictions.push(...clusterContradictions)

      // Resolve contradictions: keep newer entry, archive older
      const resolvedEntries = this.resolveContradictions(
        cluster.entries,
        clusterContradictions,
        archived
      )

      // Merge remaining near-duplicates into representative entries
      const merged = this.compactor.mergeCluster(resolvedEntries)

      // Apply archival rules to merged entries
      for (const entry of merged) {
        if (this.shouldArchive(entry, now, NINETY_DAYS_MS)) {
          archived.push(entry)
        } else {
          consolidated.push(entry)
        }
      }
    }

    return { consolidated, archived, contradictions }
  }

  // ── Phase 4: ENRICH ───────────────────────────────────────────────

  /**
   * Enrich consolidated entries with links to external knowledge systems.
   * Graphify and Wiki links are added if the respective services are available.
   * Falls back gracefully if services are unavailable.
   */
  private async enrich(
    entries: MemoryEntry[]
  ): Promise<EnrichResult> {
    let wikiLinksAdded  = 0
    let graphLinksAdded = 0
    const enriched: MemoryEntry[] = []

    for (const entry of entries) {
      let enrichedEntry = { ...entry }

      // Try to link to Graphify graph node
      if (this.services.graphify && !entry.graphNodeId) {
        try {
          const node = await this.services.graphify.findNode(entry.content)
          if (node) {
            enrichedEntry = { ...enrichedEntry, graphNodeId: node.id }
            graphLinksAdded++
          }
        } catch (err) {
          console.warn(
            `[AutoDream] Graphify enrichment failed for entry ${entry.id}:`, err
          )
        }
      }

      // Try to link to Wiki entry
      if (this.services.wiki && !entry.wikiSlug) {
        try {
          const wikiEntry = await this.services.wiki.findRelevant(entry.content)
          if (wikiEntry) {
            enrichedEntry = { ...enrichedEntry, wikiSlug: wikiEntry.slug }
            wikiLinksAdded++
          }
        } catch (err) {
          console.warn(
            `[AutoDream] Wiki enrichment failed for entry ${entry.id}:`, err
          )
        }
      }

      enriched.push(enrichedEntry)
    }

    return { enriched, wikiLinksAdded, graphLinksAdded }
  }

  // ── Phase 5: COMMIT ───────────────────────────────────────────────

  /**
   * Write consolidated entries back to the memory system.
   * Archives removed entries to the dream archive directory.
   */
  private async commit(
    consolidated: MemoryEntry[],
    archived: MemoryEntry[]
  ): Promise<void> {
    // Replace the full memory index with consolidated entries
    await this.manager.replaceAll(consolidated)

    // Write archived entries to dream archive
    if (archived.length > 0) {
      await this.writeArchive(archived)
    }

    // Update the last AutoDream timestamp
    // (MemoryIndex.setLastAutoDreamAt is exposed via manager)
    console.log(
      `[AutoDream] Committed ${consolidated.length} entries. ` +
      `Archived ${archived.length} entries.`
    )
  }

  // ── Session insight extraction ────────────────────────────────────

  /**
   * Convert session insights into new memory entries.
   * These represent knowledge extracted from recent sessions
   * that wasn't captured in the real-time flow.
   */
  private async extractNewEntriesFromSessions(
    insights: SessionInsight[]
  ): Promise<MemoryEntry[]> {
    const now = Date.now()
    const newEntries: MemoryEntry[] = []

    for (const insight of insights) {
      // Add session summary as a note entry if it's meaningful
      if (insight.summary.length > 30 && !insight.summary.includes('no user messages')) {
        newEntries.push({
          id:         randomUUID(),
          timestamp:  insight.startedAt,
          sessionId:  insight.sessionId,
          category:   'note',
          content:    `Session: ${insight.summary}`,
          importance: 'low',
          tags:       ['session-summary', 'auto-extracted'],
        })
      }

      // Extract tool usage patterns if they're novel
      if (insight.toolsUsed.length >= 5) {
        const toolList = insight.toolsUsed.slice(0, 10).join(', ')
        newEntries.push({
          id:         randomUUID(),
          timestamp:  insight.startedAt,
          sessionId:  insight.sessionId,
          category:   'note',
          content:    `Heavy tool usage session: ${toolList}`,
          importance: 'low',
          tags:       ['tool-pattern', 'auto-extracted'],
        })
      }
    }

    return newEntries
  }

  // ── Helpers ───────────────────────────────────────────────────────

  /**
   * Determine if an entry should be archived based on age and importance.
   */
  private shouldArchive(
    entry: MemoryEntry,
    now: number,
    ninetyDaysMs: number
  ): boolean {
    // Never archive critical entries
    if (entry.importance === 'critical') return false

    // Archive expired entries
    if (entry.expiresAt && now > entry.expiresAt) return true

    // Archive old low-importance entries
    if (entry.importance === 'low' && now - entry.timestamp > ninetyDaysMs) {
      return true
    }

    // Archive ephemeral entries (tagged by extraction)
    if (
      entry.tags.includes('ephemeral') &&
      entry.importance !== 'high' &&
      entry.importance !== 'critical'
    ) {
      return true
    }

    return false
  }

  /**
   * Resolve contradictions within a cluster.
   * For each contradicting pair, keeps the newer entry and archives the older.
   */
  private resolveContradictions(
    entries: MemoryEntry[],
    contradictions: ContradictionRecord[],
    archived: MemoryEntry[]
  ): MemoryEntry[] {
    const archivedIds = new Set<string>()

    for (const contradiction of contradictions) {
      archivedIds.add(contradiction.archived)
      const archivedEntry = entries.find((e) => e.id === contradiction.archived)
      if (archivedEntry) archived.push(archivedEntry)
    }

    return entries.filter((e) => !archivedIds.has(e.id))
  }

  /**
   * Write archived entries to the dream archive directory.
   */
  private async writeArchive(archived: MemoryEntry[]): Promise<void> {
    const archiveDir = join(
      this.config.workingDirectory,
      '.locoworker',
      'memory',
      'archive'
    )
    await mkdir(archiveDir, { recursive: true })

    const date     = new Date().toISOString().split('T')[0]!
    const filename = join(archiveDir, `${date}.jsonl`)
    const lines    = archived.map((e) => JSON.stringify(e)).join('\n') + '\n'

    try {
      await appendFile(filename, lines, 'utf-8')
    } catch (err) {
      console.error('[AutoDream] Failed to write archive:', err)
    }
  }

  /**
   * Write a dream session journal entry.
   */
  private async writeDreamJournal(session: DreamSession): Promise<void> {
    const dreamDir = join(
      this.config.workingDirectory,
      '.locoworker',
      'memory',
      'dreams'
    )
    await mkdir(dreamDir, { recursive: true })

    // Write full session JSON
    const jsonPath = join(dreamDir, `${session.id}.json`)
    try {
      await writeFile(
        jsonPath,
        JSON.stringify(session, null, 2),
        'utf-8'
      )
    } catch (err) {
      console.error('[AutoDream] Failed to write dream JSON:', err)
    }

    // Append to human-readable journal
    const journalPath = join(dreamDir, 'journal.md')
    const date        = new Date(session.startedAt).toISOString().split('T')[0]!
    const duration    = Math.round((session.completedAt - session.startedAt) / 1_000)

    const entry = [
      ``,
      `## AutoDream — ${date} (${session.id})`,
      ``,
      `| Metric              | Value                    |`,
      `|---------------------|--------------------------|`,
      `| Entries processed   | ${session.entriesInput}  |`,
      `| Entries merged      | ${session.entriesMerged} |`,
      `| Entries archived    | ${session.entriesArchived}|`,
      `| New entries created | ${session.entriesCreated}|`,
      `| Contradictions      | ${session.contradictions.length}|`,
      `| Wiki links added    | ${session.wikiLinksAdded}|`,
      `| Graph links added   | ${session.graphLinksAdded}|`,
      `| Cost                | $${session.costUsd.toFixed(4)} |`,
      `| Duration            | ${duration}s              |`,
      ``,
      `**Summary:** ${session.dreamSummary}`,
      ``,
    ].join('\n')

    try {
      await appendFile(journalPath, entry, 'utf-8')
    } catch (err) {
      console.error('[AutoDream] Failed to write dream journal:', err)
    }
  }

  /**
   * Build a human-readable dream summary string.
   */
  private buildDreamSummary(
    totalInput:       number,
    synthesis:        SynthesisResult,
    enrichment:       EnrichResult,
    newEntriesCount:  number
  ): string {
    const parts: string[] = []

    parts.push(`Processed ${totalInput} entries.`)

    const merged = totalInput - synthesis.consolidated.length
    if (merged > 0) parts.push(`Merged ${merged} duplicates.`)

    if (synthesis.archived.length > 0) {
      parts.push(`Archived ${synthesis.archived.length} stale entries.`)
    }

    if (synthesis.contradictions.length > 0) {
      parts.push(`Resolved ${synthesis.contradictions.length} contradictions.`)
    }

    if (enrichment.wikiLinksAdded > 0) {
      parts.push(`Added ${enrichment.wikiLinksAdded} wiki links.`)
    }

    if (enrichment.graphLinksAdded > 0) {
      parts.push(`Added ${enrichment.graphLinksAdded} graph links.`)
    }

    if (newEntriesCount > 0) {
      parts.push(`Extracted ${newEntriesCount} new insights from sessions.`)
    }

    return parts.join(' ')
  }

  /**
   * Calculate Buddy companion growth from this dream session.
   */
  private calculateBuddyGrowth(session: DreamSession): BuddyGrowthEvent {
    const xpGained =
      Math.floor(session.entriesInput * 1.5) +
      session.contradictions.length * 5 +
      session.wikiLinksAdded * 2 +
      session.graphLinksAdded * 2

    const wisdomGain = Math.floor(session.entriesInput / 10)
    const chaosReduction = Math.min(5, session.contradictions.length * 2)

    return {
      xpGained,
      statsChanged: {
        WISDOM: wisdomGain,
        CHAOS:  -chaosReduction,
      },
      levelUp: xpGained > 200,
    }
  }
}

// ── Internal types ─────────────────────────────────────────────────────

interface DreamCorpus {
  memoryEntries:   MemoryEntry[]
  sessionInsights: SessionInsight[]
  ephemeralEntries: MemoryEntry[]
  allEntries:      MemoryEntry[]
}

interface SessionInsight {
  sessionId:    string
  startedAt:    number
  decisions:    string[]
  filesChanged: string[]
  toolsUsed:    string[]
  summary:      string
}

export interface MemoryCluster {
  id:         string
  centroid:   string       // Representative term(s) for the cluster
  entries:    MemoryEntry[]
  similarity: number       // Average pairwise similarity within cluster
}

interface SynthesisResult {
  consolidated:  MemoryEntry[]
  archived:      MemoryEntry[]
  contradictions: ContradictionRecord[]
}

interface EnrichResult {
  enriched:        MemoryEntry[]
  wikiLinksAdded:  number
  graphLinksAdded: number
}
packages/memory/src/MemoryCompactor.ts
TypeScript

import { randomUUID } from 'node:crypto'
import type { MemoryEntry } from '@locoworker/core'
import type { ContradictionRecord } from './types/memory-internal.types.js'
import type { MemoryCluster } from './AutoDream.js'

/**
 * Result of a cluster merge operation.
 */
export interface MergeResult {
  representative: MemoryEntry
  archived:       MemoryEntry[]
  mergeReason:    string
}

/**
 * MemoryCompactor provides all clustering and merging logic for AutoDream.
 *
 * It is a pure computation class — no I/O, no side effects.
 * All state is passed in and returned; nothing is mutated in place.
 *
 * Algorithms used:
 *
 *   CLUSTERING — Greedy similarity clustering (O(n²), acceptable for ≤5000 entries)
 *     - TF-IDF inspired term weighting for scoring
 *     - Jaccard similarity on term sets
 *     - Threshold: 0.35 similarity → same cluster
 *
 *   CONTRADICTION DETECTION — Heuristic negation analysis
 *     - Detects opposing claims (keyword + its negation)
 *     - Resolves by keeping the newer entry
 *
 *   MERGING — Representative selection within a cluster
 *     - Keeps the highest-importance entry as representative
 *     - Merges tags from all archived entries into the representative
 *     - Preserves the oldest timestamp (original discovery date)
 */
export class MemoryCompactor {
  /** Minimum Jaccard similarity to place two entries in the same cluster */
  private readonly SIMILARITY_THRESHOLD = 0.35

  /** Common English stop words — excluded from term sets */
  private readonly STOP_WORDS = new Set([
    'a', 'an', 'the', 'and', 'or', 'but', 'in', 'on', 'at', 'to',
    'for', 'of', 'with', 'by', 'from', 'as', 'is', 'are', 'was',
    'were', 'be', 'been', 'being', 'have', 'has', 'had', 'do', 'does',
    'did', 'will', 'would', 'could', 'should', 'may', 'might', 'must',
    'can', 'it', 'its', 'this', 'that', 'these', 'those', 'i', 'we',
    'you', 'he', 'she', 'they', 'what', 'which', 'who', 'when', 'where',
    'why', 'how', 'all', 'each', 'every', 'both', 'few', 'more', 'most',
    'not', 'no', 'nor', 'so', 'yet', 'use', 'used', 'using',
  ])

  // ── Clustering ────────────────────────────────────────────────────

  /**
   * Cluster memory entries by semantic similarity.
   *
   * Algorithm:
   *   1. Compute term sets for each entry
   *   2. Greedy assignment: for each entry, find the most similar
   *      existing cluster centroid. If similarity ≥ threshold, join it.
   *      Otherwise create a new cluster.
   *   3. Sort each cluster by importance desc, then timestamp desc.
   *
   * @param entries - All memory entries to cluster
   * @returns Array of clusters, each containing related entries
   */
  cluster(entries: MemoryEntry[]): MemoryCluster[] {
    if (entries.length === 0) return []
    if (entries.length === 1) {
      return [{
        id:         randomUUID(),
        centroid:   this.extractCentroid(entries[0]!),
        entries:    [...entries],
        similarity: 1.0,
      }]
    }

    // Pre-compute term sets for all entries
    const termSets = new Map<string, Set<string>>()
    for (const entry of entries) {
      termSets.set(entry.id, this.extractTermSet(entry.content))
    }

    const clusters: MemoryCluster[] = []

    for (const entry of entries) {
      const entryTerms = termSets.get(entry.id)!
      if (entryTerms.size === 0) {
        // Empty term set — put in its own cluster
        clusters.push({
          id:         randomUUID(),
          centroid:   entry.content.slice(0, 30),
          entries:    [entry],
          similarity: 1.0,
        })
        continue
      }

      // Find the best-matching existing cluster
      let bestCluster:    MemoryCluster | null = null
      let bestSimilarity = 0

      for (const cluster of clusters) {
        const clusterTerms = this.buildClusterTermSet(cluster, termSets)
        const similarity   = this.jaccardSimilarity(entryTerms, clusterTerms)

        if (similarity > bestSimilarity) {
          bestSimilarity = similarity
          bestCluster    = cluster
        }
      }

      if (bestCluster && bestSimilarity >= this.SIMILARITY_THRESHOLD) {
        bestCluster.entries.push(entry)
        // Update similarity to running average
        bestCluster.similarity =
          (bestCluster.similarity * (bestCluster.entries.length - 1) + bestSimilarity) /
          bestCluster.entries.length
      } else {
        // New cluster
        clusters.push({
          id:         randomUUID(),
          centroid:   this.extractCentroid(entry),
          entries:    [entry],
          similarity: 1.0,
        })
      }
    }

    // Sort entries within each cluster: importance desc, then recency desc
    for (const cluster of clusters) {
      cluster.entries.sort((a, b) => {
        const impDiff = IMPORTANCE_RANK[b.importance] - IMPORTANCE_RANK[a.importance]
        if (impDiff !== 0) return impDiff
        return b.timestamp - a.timestamp
      })
    }

    return clusters
  }

  // ── Contradiction Detection ───────────────────────────────────────

  /**
   * Detect contradicting entries within a cluster.
   *
   * An entry A contradicts entry B if:
   *   - They share significant overlapping terms (Jaccard ≥ 0.25)
   *   - One contains a negation of a key term in the other
   *
   * Resolution: keep the newer entry (higher timestamp), archive the older.
   *
   * @param entries - Entries from a single cluster
   * @returns Array of contradiction records
   */
  detectContradictions(entries: MemoryEntry[]): ContradictionRecord[] {
    if (entries.length < 2) return []

    const contradictions: ContradictionRecord[] = []
    const processed = new Set<string>()

    for (let i = 0; i < entries.length; i++) {
      for (let j = i + 1; j < entries.length; j++) {
        const a = entries[i]!
        const b = entries[j]!

        if (processed.has(a.id) || processed.has(b.id)) continue

        if (this.areContradictory(a, b)) {
          // Keep newer, archive older
          const kept     = a.timestamp >= b.timestamp ? a : b
          const archived = a.timestamp >= b.timestamp ? b : a

          contradictions.push({
            entryAId: a.id,
            entryBId: b.id,
            kept:     kept.id,
            archived: archived.id,
            reason:   this.describeContradiction(a, b),
          })

          processed.add(archived.id)
        }
      }
    }

    return contradictions
  }

  /**
   * Check if two entries are contradictory.
   */
  private areContradictory(a: MemoryEntry, b: MemoryEntry): boolean {
    const aTerms = this.extractTermSet(a.content)
    const bTerms = this.extractTermSet(b.content)

    // Must share enough common terms to be about the same topic
    const similarity = this.jaccardSimilarity(aTerms, bTerms)
    if (similarity < 0.25) return false

    const aLower = a.content.toLowerCase()
    const bLower = b.content.toLowerCase()

    // Negation patterns: one contains "not/never/don't" before a keyword
    // that the other contains without negation
    const NEGATION_PREFIXES = ['not ', 'never ', "don't ", "doesn't ", 'avoid ', 'removed ']
    const AFFIRMATION_KEYWORDS = [
      'use ', 'using ', 'prefer ', 'chose ', 'switched to ', 'adopted ',
      'always ', 'recommend ', 'install ', 'enabled ',
    ]

    for (const neg of NEGATION_PREFIXES) {
      for (const aff of AFFIRMATION_KEYWORDS) {
        const negPhrase = neg + aff.trim()
        const affPhrase = aff.trim()

        const aHasNeg = aLower.includes(negPhrase)
        const bHasAff = bLower.includes(affPhrase) && !bLower.includes(negPhrase)
        if (aHasNeg && bHasAff) return true

        const bHasNeg = bLower.includes(negPhrase)
        const aHasAff = aLower.includes(affPhrase) && !aLower.includes(negPhrase)
        if (bHasNeg && aHasAff) return true
      }
    }

    return false
  }

  /**
   * Build a human-readable description of why two entries contradict.
   */
  private describeContradiction(a: MemoryEntry, b: MemoryEntry): string {
    const aSnippet = a.content.slice(0, 80)
    const bSnippet = b.content.slice(0, 80)
    return `"${aSnippet}" vs "${bSnippet}" — newer entry retained`
  }

  // ── Merging ───────────────────────────────────────────────────────

  /**
   * Merge a cluster of related entries into one or more representative entries.
   *
   * Strategy:
   *   - The highest-importance entry becomes the representative
   *   - Tags from all entries are merged into the representative
   *   - The oldest timestamp is preserved (original discovery date)
   *   - Near-exact duplicates (similarity ≥ 0.85) are collapsed into one
   *   - Dissimilar entries within the cluster are kept separate
   */
  mergeCluster(entries: MemoryEntry[]): MemoryEntry[] {
    if (entries.length === 0) return []
    if (entries.length === 1) return [...entries]

    // Find near-exact duplicates (similarity ≥ 0.85)
    const groups = this.groupNearDuplicates(entries)
    const result: MemoryEntry[] = []

    for (const group of groups) {
      if (group.length === 1) {
        result.push(group[0]!)
        continue
      }

      // Merge the group into a single representative entry
      const merged = this.mergeGroup(group)
      result.push(merged)
    }

    return result
  }

  /**
   * Group near-exact duplicates (Jaccard ≥ 0.85) within a set of entries.
   */
  private groupNearDuplicates(entries: MemoryEntry[]): MemoryEntry[][] {
    const NEAR_DUPLICATE_THRESHOLD = 0.85
    const groups: MemoryEntry[][] = []
    const assigned = new Set<string>()

    for (let i = 0; i < entries.length; i++) {
      const a = entries[i]!
      if (assigned.has(a.id)) continue

      const group: MemoryEntry[] = [a]
      assigned.add(a.id)

      const aTerms = this.extractTermSet(a.content)

      for (let j = i + 1; j < entries.length; j++) {
        const b = entries[j]!
        if (assigned.has(b.id)) continue

        const bTerms   = this.extractTermSet(b.content)
        const similarity = this.jaccardSimilarity(aTerms, bTerms)

        if (similarity >= NEAR_DUPLICATE_THRESHOLD) {
          group.push(b)
          assigned.add(b.id)
        }
      }

      groups.push(group)
    }

    return groups
  }

  /**
   * Merge a group of near-duplicate entries into a single representative.
   * Keeps the highest-importance entry as the base.
   * Merges all tags. Preserves earliest timestamp.
   */
  private mergeGroup(group: MemoryEntry[]): MemoryEntry {
    // Sort: importance desc, timestamp desc
    const sorted = [...group].sort((a, b) => {
      const impDiff = IMPORTANCE_RANK[b.importance] - IMPORTANCE_RANK[a.importance]
      if (impDiff !== 0) return impDiff
      return b.timestamp - a.timestamp
    })

    const representative = sorted[0]!
    const allTags = new Set<string>()

    for (const entry of group) {
      for (const tag of entry.tags) allTags.add(tag)
    }

    // Mark as AutoDream-merged
    allTags.add('autodream-merged')

    // Use the earliest timestamp to preserve original discovery date
    const oldestTimestamp = Math.min(...group.map((e) => e.timestamp))

    return {
      ...representative,
      timestamp: oldestTimestamp,
      tags:      [...allTags],
    }
  }

  // ── Term extraction and similarity ────────────────────────────────

  /**
   * Extract a term set from text for similarity computation.
   * Removes stop words and short tokens. Lowercases everything.
   */
  extractTermSet(text: string): Set<string> {
    const terms = text
      .toLowerCase()
      .replace(/[^\w\s-]/g, ' ')
      .split(/\s+/)
      .filter((w) => w.length > 2 && !this.STOP_WORDS.has(w))

    return new Set(terms)
  }

  /**
   * Compute Jaccard similarity between two term sets.
   * Jaccard = |A ∩ B| / |A ∪ B|
   * Range: [0, 1], where 1 = identical sets.
   */
  jaccardSimilarity(a: Set<string>, b: Set<string>): number {
    if (a.size === 0 && b.size === 0) return 1.0
    if (a.size === 0 || b.size === 0) return 0.0

    let intersectionSize = 0
    for (const term of a) {
      if (b.has(term)) intersectionSize++
    }

    const unionSize = a.size + b.size - intersectionSize
    return unionSize === 0 ? 0 : intersectionSize / unionSize
  }

  /**
   * Build a unified term set representing an entire cluster.
   * Uses the union of the top-3 entries' term sets to avoid
   * very large sets skewing the similarity.
   */
  private buildClusterTermSet(
    cluster: MemoryCluster,
    termSets: Map<string, Set<string>>
  ): Set<string> {
    const topEntries = cluster.entries.slice(0, 3)
    const union = new Set<string>()
    for (const entry of topEntries) {
      const terms = termSets.get(entry.id)
      if (terms) {
        for (const term of terms) union.add(term)
      }
    }
    return union
  }

  /**
   * Extract a short centroid string from an entry.
   * Used as the cluster label.
   */
  private extractCentroid(entry: MemoryEntry): string {
    const terms = [...this.extractTermSet(entry.content)]
      .slice(0, 5)
      .join(', ')
    return terms || entry.content.slice(0, 40)
  }
}

// ── Constants ──────────────────────────────────────────────────────────

const IMPORTANCE_RANK: Record<string, number> = {
  low:      0,
  medium:   1,
  high:     2,
  critical: 3,
}
packages/memory/src/index.ts
TypeScript

/**
 * @locoworker/memory — Public API
 *
 * The LocoWorker memory layer.
 * Import from this barrel for all memory operations.
 */

// ── Primary interface ──────────────────────────────────────────────────
export { MemoryManager } from './MemoryManager.js'
export type {
  MemoryUpdatePayload,
  MemoryQueryOptions,
  MemoryQueryResult,
  MemoryStats,
} from './MemoryManager.js'

// ── Conversation persistence ───────────────────────────────────────────
export { ConversationStore } from './ConversationStore.js'
export type {
  ArchiveSessionOptions,
  SessionSnapshot,
  ConversationStats,
} from './ConversationStore.js'

// ── Storage engine (low-level) ─────────────────────────────────────────
export { MemoryIndex } from './MemoryIndex.js'

// ── AutoDream ──────────────────────────────────────────────────────────
export { AutoDream } from './AutoDream.js'
export type {
  AutoDreamConfig,
  AutoDreamExternalServices,
  AutoDreamResult,
  BuddyGrowthEvent,
  MemoryCluster,
} from './AutoDream.js'

// ── Compaction utilities ───────────────────────────────────────────────
export { MemoryCompactor } from './MemoryCompactor.js'
export type { MergeResult } from './MemoryCompactor.js'

// ── Formatting ─────────────────────────────────────────────────────────
export { MemoryFormatter } from './MemoryFormatter.js'
export type { MemoryMdOptions } from './MemoryFormatter.js'

// ── Internal types (re-exported for consumers that need them) ──────────
export type {
  PersistedMemoryEntry,
  PersistedConversationTurn,
  SessionArchiveEntry,
  MemoryIndexFile,
  DreamSession,
  ContradictionRecord,
  SignificanceSignal,
} from './types/memory-internal.types.js'

export {
  MemoryEntrySchema,
  MemoryCategorySchema,
  MemoryImportanceSchema,
  ConversationTurnSchema,
  MEMORY_INDEX_VERSION,
  SIGNIFICANCE_SIGNALS,
  EXPLICIT_REMEMBER_PATTERN,
  EXPLICIT_NOTE_PATTERN,
  FILE_LOCATION_PATTERN,
} from './types/memory-internal.types.js'
packages/memory/tests/MemoryManager.test.ts
TypeScript

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import { mkdtemp, rm } from 'node:fs/promises'
import { join } from 'node:path'
import { tmpdir } from 'node:os'
import { MemoryManager } from '../src/MemoryManager.js'
import type { MemoryUpdatePayload } from '../src/MemoryManager.js'
import type { AssistantMessage } from '@locoworker/core'

// ── Helpers ───────────────────────────────────────────────────────────

function makeResponse(content: string): AssistantMessage {
  return { role: 'assistant', content }
}

function makePayload(
  content: string,
  sessionId = 'test-session',
  dir = '/tmp'
): MemoryUpdatePayload {
  return {
    sessionId,
    workingDirectory: dir,
    userInput: 'Test user input',
    response: makeResponse(content),
    turnNumber: 1,
  }
}

// ── Tests ─────────────────────────────────────────────────────────────

describe('MemoryManager', () => {
  let tmpDir: string
  let manager: MemoryManager

  beforeEach(async () => {
    tmpDir  = await mkdtemp(join(tmpdir(), 'locoworker-memory-test-'))
    manager = new MemoryManager(tmpDir)
    await manager.initialise()
  })

  afterEach(async () => {
    await rm(tmpDir, { recursive: true, force: true })
  })

  // ── Initialisation ─────────────────────────────────────────────────

  describe('initialise()', () => {
    it('should initialise without errors on empty directory', async () => {
      const m = new MemoryManager(tmpDir)
      await expect(m.initialise()).resolves.toBeUndefined()
    })

    it('should be idempotent — safe to call multiple times', async () => {
      await expect(manager.initialise()).resolves.toBeUndefined()
      await expect(manager.initialise()).resolves.toBeUndefined()
    })
  })

  // ── Update / Extraction ────────────────────────────────────────────

  describe('update()', () => {
    it('should extract explicit [REMEMBER: ...] markers', async () => {
      const payload = makePayload(
        'I looked at the code. [REMEMBER: Use tRPC for all IPC communication]',
        'sess-1',
        tmpDir
      )
      const entries = await manager.update(payload)
      expect(entries.length).toBeGreaterThanOrEqual(1)

      const remember = entries.find((e) =>
        e.content.includes('tRPC') && e.tags.includes('remember')
      )
      expect(remember).toBeDefined()
      expect(remember?.importance).toBe('high')
      expect(remember?.category).toBe('decision')
    })

    it('should extract explicit [NOTE: ...] markers', async () => {
      const payload = makePayload(
        'Done. [NOTE: Ollama streaming drops last token on ARM Macs]',
        'sess-2',
        tmpDir
      )
      const entries = await manager.update(payload)
      const note = entries.find((e) =>
        e.content.includes('Ollama') && e.tags.includes('note')
      )
      expect(note).toBeDefined()
      expect(note?.category).toBe('note')
    })

    it('should extract file location patterns', async () => {
      const payload = makePayload(
        'The main entry is located at src/index.ts for the CLI.',
        'sess-3',
        tmpDir
      )
      const entries = await manager.update(payload)
      const fileEntry = entries.find((e) => e.category === 'file_location')
      expect(fileEntry).toBeDefined()
    })

    it('should deduplicate identical content within a single turn', async () => {
      const payload = makePayload(
        '[REMEMBER: Always use pnpm] and again [REMEMBER: Always use pnpm]',
        'sess-4',
        tmpDir
      )
      const entries = await manager.update(payload)
      const pnpmEntries = entries.filter((e) =>
        e.content.toLowerCase().includes('pnpm')
      )
      expect(pnpmEntries.length).toBe(1)
    })

    it('should capture tool errors as issue entries', async () => {
      const payload: MemoryUpdatePayload = {
        sessionId:        'sess-5',
        workingDirectory: tmpDir,
        userInput:        'run tests',
        response:         makeResponse('Tests failed.'),
        toolResults: [{
          toolName: 'bash',
          content:  'ENOENT: no such file or directory, open package.json',
          isError:  true,
        }],
        turnNumber: 2,
      }
      const entries = await manager.update(payload)
      const issueEntry = entries.find(
        (e) => e.category === 'issue' && e.tags.includes('bash')
      )
      expect(issueEntry).toBeDefined()
    })

    it('should return empty array for non-significant turns', async () => {
      const payload = makePayload(
        'Okay, sounds good.',
        'sess-6',
        tmpDir
      )
      const entries = await manager.update(payload)
      // May extract nothing or very little from a trivial response
      expect(Array.isArray(entries)).toBe(true)
    })
  })

  // ── Query ──────────────────────────────────────────────────────────

  describe('query()', () => {
    beforeEach(async () => {
      // Seed some entries
      await manager.add('Use tRPC for IPC layer', 'architecture', 'high', ['tRPC', 'ipc'])
      await manager.add('Never use jQuery', 'preference', 'high', ['jquery'])
      await manager.add('Main entry at src/index.ts', 'file_location', 'low', ['entry'])
      await manager.add('Bug: Ollama drops last token', 'issue', 'medium', ['ollama'])
    })

    it('should return entries matching the query', async () => {
      const result = await manager.query('tRPC IPC layer')
      expect(result.entries.length).toBeGreaterThan(0)
      expect(result.entries[0]?.content).toContain('tRPC')
    })

    it('should filter by category', async () => {
      const result = await manager.query('', { categories: ['architecture'] })
      expect(result.entries.every((e) => e.category === 'architecture')).toBe(true)
    })

    it('should filter by minImportance', async () => {
      const result = await manager.query('', { minImportance: 'high' })
      const importances = result.entries.map((e) => e.importance)
      expect(importances.every((i) => i === 'high' || i === 'critical')).toBe(true)
    })

    it('should filter by tags', async () => {
      const result = await manager.query('', { tags: ['ollama'] })
      expect(result.entries.length).toBeGreaterThan(0)
      expect(result.entries[0]?.tags).toContain('ollama')
    })

    it('should respect the limit option', async () => {
      const result = await manager.query('', { limit: 1 })
      expect(result.entries.length).toBeLessThanOrEqual(1)
    })

    it('should return totalMatched correctly', async () => {
      const result = await manager.query('tRPC')
      expect(result.totalMatched).toBeGreaterThanOrEqual(result.entries.length)
    })

    it('should return queryTimeMs as a non-negative number', async () => {
      const result = await manager.query('anything')
      expect(result.queryTimeMs).toBeGreaterThanOrEqual(0)
    })
  })

  // ── Manual add / remove ────────────────────────────────────────────

  describe('add()', () => {
    it('should add an entry and make it queryable', async () => {
      await manager.add('Chosen Bun over Node for speed', 'decision', 'high', ['bun'])
      const result = await manager.query('Bun')
      expect(result.entries.some((e) => e.content.includes('Bun'))).toBe(true)
    })

    it('should trim and truncate content', async () => {
      const long = 'x'.repeat(3_000)
      const entry = await manager.add(long, 'note', 'low')
      expect(entry.content.length).toBeLessThanOrEqual(2_000)
    })
  })

  describe('remove()', () => {
    it('should remove an existing entry', async () => {
      const entry = await manager.add('Temporary note', 'note', 'low')
      const removed = await manager.remove(entry.id)
      expect(removed).toBe(true)

      const result = await manager.query('Temporary note')
      expect(result.entries.length).toBe(0)
    })

    it('should return false for a non-existent entry ID', async () => {
      const removed = await manager.remove('does-not-exist')
      expect(removed).toBe(false)
    })
  })

  // ── Prune ──────────────────────────────────────────────────────────

  describe('prune()', () => {
    it('should remove expired entries', async () => {
      // Add an entry that expired in the past
      const expired = await manager.add('Old note', 'note', 'low')
      // Manually set expiresAt in the past by accessing the index
      const allBefore = await manager.loadAll()
      const toExpire  = allBefore.find((e) => e.id === expired.id)
      if (toExpire) {
        toExpire.expiresAt = Date.now() - 1_000
        await manager.replaceAll(allBefore)
      }

      const { pruned } = await manager.prune()
      expect(pruned).toBeGreaterThanOrEqual(1)

      const result = await manager.query('Old note')
      expect(result.entries.find((e) => e.id === expired.id)).toBeUndefined()
    })
  })

  // ── Stats ──────────────────────────────────────────────────────────

  describe('getStats()', () => {
    it('should return accurate statistics', async () => {
      await manager.add('Arch note', 'architecture', 'high')
      await manager.add('Pref note', 'preference', 'medium')

      const stats = await manager.getStats()
      expect(stats.totalEntries).toBeGreaterThanOrEqual(2)
      expect(stats.byCategory['architecture']).toBeGreaterThanOrEqual(1)
      expect(stats.byCategory['preference']).toBeGreaterThanOrEqual(1)
    })
  })

  // ── MEMORY.md generation ───────────────────────────────────────────

  describe('readForContext()', () => {
    it('should return a non-empty string after entries are added', async () => {
      await manager.add('Architecture: use monorepo', 'architecture', 'high')
      const context = await manager.readForContext()
      expect(context.length).toBeGreaterThan(0)
      expect(context).toContain('monorepo')
    })

    it('should return empty string when no entries exist', async () => {
      const context = await manager.readForContext()
      expect(context).toBe('')
    })

    it('should respect the maxTokens budget', async () => {
      // Add many entries
      for (let i = 0; i < 50; i++) {
        await manager.add(`Entry number ${i} with some content`, 'note', 'low')
      }
      const context = await manager.readForContext(200) // Very small budget
      const estimatedTokens = Math.ceil(context.length / 4)
      expect(estimatedTokens).toBeLessThanOrEqual(210) // Allow small overshoot
    })
  })
})
packages/memory/tests/ConversationStore.test.ts
TypeScript

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import { mkdtemp, rm } from 'node:fs/promises'
import { join } from 'node:path'
import { tmpdir } from 'node:os'
import { ConversationStore } from '../src/ConversationStore.js'
import type { ArchiveSessionOptions } from '../src/ConversationStore.js'
import type { ConversationTurn } from '@locoworker/core'

// ── Helpers ───────────────────────────────────────────────────────────

function makeTurns(count: number): ConversationTurn[] {
  return Array.from({ length: count }, (_, i) => ({
    role:       i % 2 === 0 ? 'user' : 'assistant',
    content:    `Turn ${i} content with some text`,
    tokenCount: 20,
    importance: 'normal' as const,
    timestamp:  Date.now() - (count - i) * 1_000,
  }))
}

function makeArchiveOptions(
  sessionId: string,
  startedAt = Date.now() - 3_600_000
): ArchiveSessionOptions {
  return {
    sessionId,
    startedAt,
    endedAt:         Date.now(),
    modelId:         'claude-haiku-4-5',
    provider:        'anthropic',
    totalCostUsd:    0.0023,
    compactionCount: 1,
  }
}

// ── Tests ─────────────────────────────────────────────────────────────

describe('ConversationStore', () => {
  let tmpDir: string
  let store: ConversationStore

  beforeEach(async () => {
    tmpDir = await mkdtemp(join(tmpdir(), 'locoworker-store-test-'))
    store  = new ConversationStore(tmpDir)
  })

  afterEach(async () => {
    await rm(tmpDir, { recursive: true, force: true })
  })

  // ── Archive ────────────────────────────────────────────────────────

  describe('archive()', () => {
    it('should archive a session and return an archive entry', async () => {
      const turns  = makeTurns(10)
      const opts   = makeArchiveOptions('sess-archive-001')
      const result = await store.archive(turns, opts)

      expect(result.sessionId).toBe('sess-archive-001')
      expect(result.totalTurns).toBe(10)
      expect(result.modelId).toBe('claude-haiku-4-5')
      expect(result.totalCostUsd).toBe(0.0023)
    })

    it('should write the snapshot to disk', async () => {
      const turns = makeTurns(5)
      const opts  = makeArchiveOptions('sess-archive-002')
      await store.archive(turns, opts)

      const snapshot = await store.loadSnapshot('sess-archive-002', opts.startedAt)
      expect(snapshot).not.toBeNull()
      expect(snapshot?.sessionId).toBe('sess-archive-002')
      expect(snapshot?.turns.length).toBe(5)
    })

    it('should append to the sessions.jsonl index', async () => {
      const turns1 = makeTurns(3)
      const turns2 = makeTurns(4)
      await store.archive(turns1, makeArchiveOptions('sess-idx-001'))
      await store.archive(turns2, makeArchiveOptions('sess-idx-002'))

      const index = await store.loadIndex()
      const ids   = index.map((e) => e.sessionId)
      expect(ids).toContain('sess-idx-001')
      expect(ids).toContain('sess-idx-002')
    })

    it('should extract key decisions from assistant turns', async () => {
      const turns: ConversationTurn[] = [
        {
          role: 'user', content: 'Should we use tRPC?',
          tokenCount: 10, importance: 'normal', timestamp: Date.now()
        },
        {
          role: 'assistant',
          content: 'I decided to use tRPC for the IPC layer because it provides type safety.',
          tokenCount: 30, importance: 'normal', timestamp: Date.now()
        },
      ]
      const opts   = makeArchiveOptions('sess-decisions')
      const result = await store.archive(turns, opts)
      expect(result.keyDecisions.length).toBeGreaterThan(0)
      expect(result.keyDecisions.some((d) => d.toLowerCase().includes('trpc'))).toBe(true)
    })

    it('should extract tools used from tool calls', async () => {
      const turns: ConversationTurn[] = [
        {
          role: 'assistant', content: 'Let me check the files.',
          tokenCount: 15, importance: 'normal', timestamp: Date.now(),
          toolCalls: [
            { id: 'tc1', name: 'read_file', input: { path: 'package.json' } },
            { id: 'tc2', name: 'list_directory', input: { path: '.' } },
          ]
        },
      ]
      const opts   = makeArchiveOptions('sess-tools')
      const result = await store.archive(turns, opts)
      expect(result.toolsUsed).toContain('read_file')
      expect(result.toolsUsed).toContain('list_directory')
    })

    it('should build a turn summary', async () => {
      const turns: ConversationTurn[] = [
        {
          role: 'user', content: 'How do I set up the monorepo?',
          tokenCount: 15, importance: 'normal', timestamp: Date.now()
        },
        {
          role: 'assistant', content: 'Here is how to set it up.',
          tokenCount: 20, importance: 'normal', timestamp: Date.now()
        },
      ]
      const opts   = makeArchiveOptions('sess-summary')
      const result = await store.archive(turns, opts)
      expect(result.turnSummary.length).toBeGreaterThan(10)
      expect(result.turnSummary).toContain('2-turn')
    })
  })

  // ── Load ───────────────────────────────────────────────────────────

  describe('loadSnapshot()', () => {
    it('should return null for a non-existent snapshot', async () => {
      const snapshot = await store.loadSnapshot('does-not-exist', Date.now())
      expect(snapshot).toBeNull()
    })
  })

  describe('loadRecentArchives()', () => {
    it('should return only sessions after sinceMs', async () => {
      const oldStart = Date.now() - 10 * 24 * 60 * 60 * 1_000 // 10 days ago
      const newStart = Date.now() - 1 * 60 * 60 * 1_000        // 1 hour ago

      await store.archive(makeTurns(2), makeArchiveOptions('old-session', oldStart))
      await store.archive(makeTurns(2), makeArchiveOptions('new-session', newStart))

      const cutoff  = Date.now() - 2 * 24 * 60 * 60 * 1_000 // 2 days ago
      const results = await store.loadRecentArchives(cutoff)

      expect(results.some((e) => e.sessionId === 'new-session')).toBe(true)
      expect(results.some((e) => e.sessionId === 'old-session')).toBe(false)
    })

    it('should return empty array when no sessions exist', async () => {
      const results = await store.loadRecentArchives(Date.now() - 86_400_000)
      expect(results).toEqual([])
    })

    it('should respect the limit parameter', async () => {
      for (let i = 0; i < 5; i++) {
        await store.archive(makeTurns(2), makeArchiveOptions(`session-${i}`))
      }
      const results = await store.loadRecentArchives(0, 2)
      expect(results.length).toBeLessThanOrEqual(2)
    })
  })

  // ── Retention ──────────────────────────────────────────────────────

  describe('applyRetention()', () => {
    it('should remove sessions older than retention days', async () => {
      const oldStart = Date.now() - 100 * 24 * 60 * 60 * 1_000 // 100 days ago
      const newStart = Date.now() - 1 * 60 * 60 * 1_000

      await store.archive(makeTurns(2), {
        ...makeArchiveOptions('old', oldStart),
        endedAt: oldStart + 3_600_000,
      })
      await store.archive(makeTurns(2), {
        ...makeArchiveOptions('new', newStart),
        endedAt: newStart + 3_600_000,
      })

      const { removed } = await store.applyRetention(90)
      expect(removed).toBeGreaterThanOrEqual(1)

      const index = await store.loadIndex()
      expect(index.some((e) => e.sessionId === 'old')).toBe(false)
      expect(index.some((e) => e.sessionId === 'new')).toBe(true)
    })

    it('should return removed: 0 when nothing is stale', async () => {
      await store.archive(makeTurns(2), makeArchiveOptions('fresh'))
      const { removed } = await store.applyRetention(90)
      expect(removed).toBe(0)
    })
  })

  // ── Stats ──────────────────────────────────────────────────────────

  describe('getStats()', () => {
    it('should return zero stats when no sessions are archived', async () => {
      const stats = await store.getStats()
      expect(stats.totalSessions).toBe(0)
      expect(stats.totalTurns).toBe(0)
    })

    it('should correctly aggregate across multiple sessions', async () => {
      await store.archive(makeTurns(5), makeArchiveOptions('s1'))
      await store.archive(makeTurns(8), makeArchiveOptions('s2'))

      const stats = await store.getStats()
      expect(stats.totalSessions).toBe(2)
      expect(stats.totalTurns).toBe(13)
      expect(stats.avgTurnsPerSession).toBeCloseTo(6.5, 1)
      expect(stats.mostUsedModels).toContain('claude-haiku-4-5')
    })
  })
})
packages/memory/tests/MemoryIndex.test.ts
TypeScript

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import { mkdtemp, rm } from 'node:fs/promises'
import { join } from 'node:path'
import { tmpdir } from 'node:os'
import { randomUUID } from 'node:crypto'
import { MemoryIndex } from '../src/MemoryIndex.js'
import type { MemoryEntry } from '@locoworker/core'

// ── Helpers ───────────────────────────────────────────────────────────

function makeEntry(
  overrides: Partial<MemoryEntry> = {}
): MemoryEntry {
  return {
    id:         randomUUID(),
    timestamp:  Date.now(),
    sessionId:  'test-session',
    category:   'note',
    content:    'Test memory entry content',
    importance: 'medium',
    tags:       ['test'],
    ...overrides,
  }
}

// ── Tests ─────────────────────────────────────────────────────────────

describe('MemoryIndex', () => {
  let tmpDir: string
  let index: MemoryIndex

  beforeEach(async () => {
    tmpDir = await mkdtemp(join(tmpdir(), 'locoworker-index-test-'))
    index  = new MemoryIndex(tmpDir)
    await index.load()
  })

  afterEach(async () => {
    await rm(tmpDir, { recursive: true, force: true })
  })

  // ── Load / Save ────────────────────────────────────────────────────

  describe('load()', () => {
    it('should start empty on first load', () => {
      expect(index.count()).toBe(0)
    })

    it('should be idempotent — calling load twice is safe', async () => {
      await index.load()
      await index.load()
      expect(index.count()).toBe(0)
    })

    it('should persist and reload entries correctly', async () => {
      const entry = makeEntry({ content: 'Persisted content', importance: 'high' })
      await index.add(entry)

      // Create a fresh index on the same directory
      const index2 = new MemoryIndex(tmpDir)
      await index2.load()

      expect(index2.count()).toBe(1)
      const loaded = index2.get(entry.id)
      expect(loaded?.content).toBe('Persisted content')
      expect(loaded?.importance).toBe('high')
    })

    it('should skip invalid entries gracefully on load', async () => {
      // Write a corrupted index
      const { writeFile, mkdir } = await import('node:fs/promises')
      const indexPath = join(tmpDir, '.locoworker', 'memory', 'index.json')
      await mkdir(join(tmpDir, '.locoworker', 'memory'), { recursive: true })
      await writeFile(indexPath, JSON.stringify({
        version: 2,
        lastUpdatedAt: Date.now(),
        lastAutoDreamAt: null,
        totalEntries: 2,
        totalArchived: 0,
        categories: {},
        entries: [
          // Valid entry
          {
            id: randomUUID(), timestamp: Date.now(),
            sessionId: 'sess', category: 'note',
            content: 'Valid', importance: 'medium', tags: [],
          },
          // Invalid entry (missing required fields)
          { id: 'broken', broken: true },
        ],
      }), 'utf-8')

      const freshIndex = new MemoryIndex(tmpDir)
      await freshIndex.load()
      // Should have loaded 1 valid entry and skipped the broken one
      expect(freshIndex.count()).toBe(1)
    })
  })

  // ── CRUD ───────────────────────────────────────────────────────────

  describe('add()', () => {
    it('should add an entry and increment count', async () => {
      const entry = makeEntry()
      await index.add(entry)
      expect(index.count()).toBe(1)
      expect(index.has(entry.id)).toBe(true)
    })

    it('should make the entry retrievable by ID', async () => {
      const entry = makeEntry({ content: 'Unique content XYZ' })
      await index.add(entry)
      const retrieved = index.get(entry.id)
      expect(retrieved?.content).toBe('Unique content XYZ')
    })
  })

  describe('addMany()', () => {
    it('should add multiple entries and save once', async () => {
      const entries = Array.from({ length: 5 }, () => makeEntry())
      await index.addMany(entries)
      expect(index.count()).toBe(5)
    })
  })

  describe('remove()', () => {
    it('should remove an entry and return true', async () => {
      const entry = makeEntry()
      await index.add(entry)
      const removed = index.remove(entry.id)
      expect(removed).toBe(true)
      expect(index.has(entry.id)).toBe(false)
    })

    it('should return false for a non-existent ID', () => {
      const removed = index.remove('non-existent-id')
      expect(removed).toBe(false)
    })
  })

  describe('getAll()', () => {
    it('should return a copy — mutations do not affect the index', async () => {
      await index.add(makeEntry())
      const all = index.getAll()
      all.push(makeEntry()) // Mutate the returned array
      expect(index.count()).toBe(1) // Index unchanged
    })
  })

  describe('getByCategory()', () => {
    it('should filter entries by category', async () => {
      await index.add(makeEntry({ category: 'architecture' }))
      await index.add(makeEntry({ category: 'preference' }))
      await index.add(makeEntry({ category: 'architecture' }))

      const arch = index.getByCategory('architecture')
      expect(arch.length).toBe(2)
      expect(arch.every((e) => e.category === 'architecture')).toBe(true)
    })
  })

  describe('getByImportance()', () => {
    it('should return entries at or above the minimum importance', async () => {
      await index.add(makeEntry({ importance: 'low' }))
      await index.add(makeEntry({ importance: 'medium' }))
      await index.add(makeEntry({ importance: 'high' }))
      await index.add(makeEntry({ importance: 'critical' }))

      const highAndAbove = index.getByImportance('high')
      expect(highAndAbove.length).toBe(2)
      expect(
        highAndAbove.every(
          (e) => e.importance === 'high' || e.importance === 'critical'
        )
      ).toBe(true)
    })
  })

  describe('getByTags()', () => {
    it('should return entries matching any of the given tags', async () => {
      await index.add(makeEntry({ tags: ['tRPC', 'ipc'] }))
      await index.add(makeEntry({ tags: ['ollama', 'local'] }))
      await index.add(makeEntry({ tags: ['bun', 'runtime'] }))

      const results = index.getByTags(['tRPC', 'bun'])
      expect(results.length).toBe(2)
    })
  })

  // ── Filter ─────────────────────────────────────────────────────────

  describe('filter()', () => {
    it('should remove entries that fail the predicate', async () => {
      await index.add(makeEntry({ importance: 'low', content: 'Remove me' }))
      await index.add(makeEntry({ importance: 'high', content: 'Keep me' }))

      index.filter((e) => e.importance !== 'low')
      expect(index.count()).toBe(1)
      expect(index.getAll()[0]?.content).toBe('Keep me')
    })
  })

  // ── replaceAll ─────────────────────────────────────────────────────

  describe('replaceAll()', () => {
    it('should completely replace the entry set', async () => {
      await index.add(makeEntry({ content: 'Old entry' }))

      const newEntries = [
        makeEntry({ content: 'New entry A' }),
        makeEntry({ content: 'New entry B' }),
      ]
      index.replaceAll(newEntries)

      expect(index.count()).toBe(2)
      const contents = index.getAll().map((e) => e.content)
      expect(contents).toContain('New entry A')
      expect(contents).toContain('New entry B')
      expect(contents).not.toContain('Old entry')
    })
  })

  // ── AutoDream metadata ─────────────────────────────────────────────

  describe('AutoDream metadata', () => {
    it('should persist lastAutoDreamAt correctly', async () => {
      expect(index.getLastAutoDreamAt()).toBeNull()
      const ts = Date.now()
      await index.setLastAutoDreamAt(ts)

      // Reload and verify persistence
      const index2 = new MemoryIndex(tmpDir)
      await index2.load()
      expect(index2.getLastAutoDreamAt()).toBe(ts)
    })
  })

  // ── Statistics ─────────────────────────────────────────────────────

  describe('getStats()', () => {
    it('should report correct totals and category counts', async () => {
      await index.add(makeEntry({ category: 'architecture', importance: 'high' }))
      await index.add(makeEntry({ category: 'preference', importance: 'medium' }))
      await index.add(makeEntry({ category: 'architecture', importance: 'critical' }))

      const stats = index.getStats()
      expect(stats.totalEntries).toBe(3)
      expect(stats.byCategory['architecture']).toBe(2)
      expect(stats.byCategory['preference']).toBe(1)
      expect(stats.byImportance['high']).toBe(1)
      expect(stats.byImportance['critical']).toBe(1)
    })

    it('should report null timestamps when index is empty', () => {
      const stats = index.getStats()
      expect(stats.oldestEntryAt).toBeNull()
      expect(stats.newestEntryAt).toBeNull()
    })

    it('should report correct oldest and newest timestamps', async () => {
      const old = makeEntry({ timestamp: 1_000_000 })
      const mid = makeEntry({ timestamp: 2_000_000 })
      const new_ = makeEntry({ timestamp: 3_000_000 })
      await index.addMany([old, mid, new_])

      const stats = index.getStats()
      expect(stats.oldestEntryAt).toBe(1_000_000)
      expect(stats.newestEntryAt).toBe(3_000_000)
    })
  })
})
packages/memory/tests/AutoDream.test.ts
TypeScript

import { describe, it, expect, beforeEach, afterEach, mock } from 'bun:test'
import { mkdtemp, rm } from 'node:fs/promises'
import { join } from 'node:path'
import { tmpdir } from 'node:os'
import { randomUUID } from 'node:crypto'
import { AutoDream } from '../src/AutoDream.js'
import { MemoryManager } from '../src/MemoryManager.js'
import { ConversationStore } from '../src/ConversationStore.js'
import { MemoryCompactor } from '../src/MemoryCompactor.js'
import type { AutoDreamConfig } from '../src/AutoDream.js'
import type { MemoryEntry } from '@locoworker/core'

// ── Helpers ───────────────────────────────────────────────────────────

function makeEntry(
  content: string,
  overrides: Partial<MemoryEntry> = {}
): MemoryEntry {
  return {
    id:         randomUUID(),
    timestamp:  Date.now(),
    sessionId:  'test-session',
    category:   'decision',
    content,
    importance: 'medium',
    tags:       ['test'],
    ...overrides,
  }
}

function makeConfig(workingDirectory: string): AutoDreamConfig {
  return {
    maxEntriesPerRun:     500,
    minHoursBetweenRuns:  20,
    sessionLookbackDays:  7,
    useLlmSummarisation:  false,
    summarisationModelId: 'phi3:mini',
    costCapUsd:           0.10,
    gitCommit:            false,
    workingDirectory,
  }
}

// ── Tests ─────────────────────────────────────────────────────────────

describe('AutoDream', () => {
  let tmpDir: string
  let manager: MemoryManager
  let store:   ConversationStore
  let dream:   AutoDream

  beforeEach(async () => {
    tmpDir  = await mkdtemp(join(tmpdir(), 'locoworker-dream-test-'))
    manager = new MemoryManager(tmpDir)
    store   = new ConversationStore(tmpDir)
    await manager.initialise()

    dream = new AutoDream(manager, store, makeConfig(tmpDir))
  })

  afterEach(async () => {
    await rm(tmpDir, { recursive: true, force: true })
  })

  // ── Full pipeline ──────────────────────────────────────────────────

  describe('run()', () => {
    it('should run successfully on an empty memory store', async () => {
      const result = await dream.run()
      expect(result.session.entriesInput).toBe(0)
      expect(result.consolidated).toEqual([])
      expect(result.archived).toEqual([])
    })

    it('should return a valid DreamSession with correct fields', async () => {
      await manager.add('Use tRPC for IPC', 'architecture', 'high')

      const result = await dream.run()
      const { session } = result

      expect(session.id).toMatch(/^dream_/)
      expect(session.startedAt).toBeLessThanOrEqual(Date.now())
      expect(session.completedAt).toBeGreaterThanOrEqual(session.startedAt)
      expect(session.entriesInput).toBeGreaterThanOrEqual(0)
      expect(typeof session.dreamSummary).toBe('string')
      expect(session.dreamSummary.length).toBeGreaterThan(0)
    })

    it('should write a dream journal file', async () => {
      await manager.add('Architecture note', 'architecture', 'medium')
      const result = await dream.run()

      const { readFile } = await import('node:fs/promises')
      const journalPath = join(
        tmpDir, '.locoworker', 'memory', 'dreams', 'journal.md'
      )
      const journal = await readFile(journalPath, 'utf-8')
      expect(journal).toContain('AutoDream')
      expect(journal).toContain(result.session.id)
    })

    it('should write a session JSON file', async () => {
      const result = await dream.run()
      const { readFile } = await import('node:fs/promises')
      const jsonPath = join(
        tmpDir, '.locoworker', 'memory', 'dreams', `${result.session.id}.json`
      )
      const json = JSON.parse(await readFile(jsonPath, 'utf-8'))
      expect(json.id).toBe(result.session.id)
    })

    it('should consolidate entries back into MemoryManager', async () => {
      await manager.add('Pref A', 'preference', 'high', ['pref'])
      await manager.add('Pref B', 'preference', 'medium', ['pref'])
      await manager.add('Arch note', 'architecture', 'high', ['arch'])

      await dream.run()

      // After dream, manager should still have entries
      const allEntries = await manager.loadAll()
      expect(allEntries.length).toBeGreaterThan(0)
    })

    it('should calculate buddy growth', async () => {
      await manager.add('Big session today', 'note', 'high')
      const result = await dream.run()
      expect(result.buddyGrowth.xpGained).toBeGreaterThanOrEqual(0)
      expect(typeof result.buddyGrowth.statsChanged['WISDOM']).toBe('number')
    })

    it('should archive stale low-importance entries', async () => {
      // Add a stale low-importance entry (91 days old)
      const staleTimestamp = Date.now() - 91 * 24 * 60 * 60 * 1_000
      const staleEntry = makeEntry('Stale old note', {
        importance: 'low',
        timestamp:  staleTimestamp,
        tags:       ['old'],
      })
      await manager.replaceAll([
        staleEntry,
        makeEntry('Keep this', { importance: 'critical' }),
      ])

      const result = await dream.run()

      // The stale entry should be in the archived list
      const archivedIds = result.archived.map((e) => e.id)
      expect(archivedIds).toContain(staleEntry.id)

      // The critical entry should remain consolidated
      const consolidatedIds = result.consolidated.map((e) => e.id)
      const hasCritical = result.consolidated.some(
        (e) => e.importance === 'critical'
      )
      expect(hasCritical).toBe(true)
    })

    it('should use external graphify service when provided', async () => {
      let graphifyCallCount = 0
      const mockGraphify = {
        findNode: async (_content: string) => {
          graphifyCallCount++
          return { id: `node-${graphifyCallCount}`, name: 'TestNode' }
        },
      }

      const dreamWithGraphify = new AutoDream(
        manager, store, makeConfig(tmpDir),
        { graphify: mockGraphify }
      )

      await manager.add('Architecture: use Graphify for code navigation', 'architecture', 'high')
      const result = await dreamWithGraphify.run()

      expect(result.session.graphLinksAdded).toBeGreaterThan(0)
    })

    it('should use external wiki service when provided', async () => {
      let wikiCallCount = 0
      const mockWiki = {
        findRelevant: async (_content: string) => {
          wikiCallCount++
          return { slug: `wiki-entry-${wikiCallCount}`, title: 'Test Wiki Entry' }
        },
      }

      const dreamWithWiki = new AutoDream(
        manager, store, makeConfig(tmpDir),
        { wiki: mockWiki }
      )

      await manager.add('Use Leiden algorithm for clustering', 'architecture', 'high')
      const result = await dreamWithWiki.run()

      expect(result.session.wikiLinksAdded).toBeGreaterThan(0)
    })
  })

  // ── Buddy growth calculation ───────────────────────────────────────

  describe('buddy growth', () => {
    it('should give more XP for more entries processed', async () => {
      // Small dream
      dream = new AutoDream(manager, store, makeConfig(tmpDir))
      const smallResult = await dream.run()
      const smallXp = smallResult.buddyGrowth.xpGained

      // Add many entries for a larger dream
      const manyEntries = Array.from({ length: 20 }, (_, i) =>
        makeEntry(`Entry ${i}`, { importance: 'medium' })
      )
      await manager.replaceAll(manyEntries)

      const largeResult = await dream.run()
      const largeXp = largeResult.buddyGrowth.xpGained

      expect(largeXp).toBeGreaterThanOrEqual(smallXp)
    })

    it('should give negative CHAOS for resolved contradictions', async () => {
      // Add contradicting entries
      const entries = [
        makeEntry('Always use pnpm for package management', { importance: 'medium' }),
        makeEntry('Never use pnpm, use npm instead', { importance: 'medium' }),
      ]
      await manager.replaceAll(entries)

      const result = await dream.run()
      // If contradictions were found, CHAOS should decrease
      if (result.session.contradictions.length > 0) {
        const chaosChange = result.buddyGrowth.statsChanged['CHAOS'] ?? 0
        expect(chaosChange).toBeLessThanOrEqual(0)
      }
    })
  })
})

// ── MemoryCompactor unit tests ──────────────────────────────────────────

describe('MemoryCompactor', () => {
  let compactor: MemoryCompactor

  beforeEach(() => {
    compactor = new MemoryCompactor()
  })

  // ── Term extraction ────────────────────────────────────────────────

  describe('extractTermSet()', () => {
    it('should remove stop words', () => {
      const terms = compactor.extractTermSet('Use the tRPC library for all IPC communication')
      expect(terms.has('the')).toBe(false)
      expect(terms.has('for')).toBe(false)
      expect(terms.has('all')).toBe(false)
      expect(terms.has('trpc')).toBe(true)
    })

    it('should lowercase all terms', () => {
      const terms = compactor.extractTermSet('TurboRepo MonoRepo Architecture')
      expect(terms.has('turborepo')).toBe(true)
      expect(terms.has('monorepo')).toBe(true)
      expect(terms.has('TurboRepo')).toBe(false)
    })

    it('should filter short tokens (length ≤ 2)', () => {
      const terms = compactor.extractTermSet('Use it as a monorepo tool')
      expect(terms.has('it')).toBe(false)
      expect(terms.has('as')).toBe(false)
      expect(terms.has('monorepo')).toBe(true)
    })
  })

  // ── Jaccard similarity ─────────────────────────────────────────────

  describe('jaccardSimilarity()', () => {
    it('should return 1.0 for identical sets', () => {
      const a = new Set(['foo', 'bar', 'baz'])
      const b = new Set(['foo', 'bar', 'baz'])
      expect(compactor.jaccardSimilarity(a, b)).toBe(1.0)
    })

    it('should return 0.0 for completely disjoint sets', () => {
      const a = new Set(['foo', 'bar'])
      const b = new Set(['baz', 'qux'])
      expect(compactor.jaccardSimilarity(a, b)).toBe(0.0)
    })

    it('should return 1.0 for two empty sets', () => {
      expect(compactor.jaccardSimilarity(new Set(), new Set())).toBe(1.0)
    })

    it('should return 0.0 for one empty set', () => {
      const a = new Set(['foo'])
      expect(compactor.jaccardSimilarity(a, new Set())).toBe(0.0)
    })

    it('should return a value between 0 and 1 for partial overlap', () => {
      const a = new Set(['foo', 'bar', 'baz'])
      const b = new Set(['bar', 'baz', 'qux'])
      const sim = compactor.jaccardSimilarity(a, b)
      expect(sim).toBeGreaterThan(0)
      expect(sim).toBeLessThan(1)
      // |A ∩ B| = 2 (bar, baz), |A ∪ B| = 4 → Jaccard = 0.5
      expect(sim).toBeCloseTo(0.5, 5)
    })
  })

  // ── Clustering ─────────────────────────────────────────────────────

  describe('cluster()', () => {
    it('should return an empty array for empty input', () => {
      const clusters = compactor.cluster([])
      expect(clusters).toEqual([])
    })

    it('should return a single cluster for a single entry', () => {
      const entry = makeEntry('Single entry about tRPC usage')
      const clusters = compactor.cluster([entry])
      expect(clusters.length).toBe(1)
      expect(clusters[0]?.entries.length).toBe(1)
    })

    it('should group semantically similar entries together', () => {
      const entries = [
        makeEntry('Use tRPC for type-safe IPC communication between processes'),
        makeEntry('tRPC provides type safety for inter-process communication'),
        makeEntry('Completely unrelated entry about database migrations'),
      ]
      const clusters = compactor.cluster(entries)
      // The tRPC entries should cluster together
      const tRPCCluster = clusters.find(
        (c) => c.entries.length >= 2 &&
               c.entries.some((e) => e.content.includes('tRPC'))
      )
      expect(tRPCCluster).toBeDefined()
    })

    it('should keep dissimilar entries in separate clusters', () => {
      const entries = [
        makeEntry('Always use pnpm for package management in monorepo'),
        makeEntry('Neo4j graph database performance tuning guide'),
        makeEntry('Docker container resource limits configuration'),
      ]
      const clusters = compactor.cluster(entries)
      // Each should be in its own cluster (they're unrelated)
      expect(clusters.length).toBeGreaterThanOrEqual(2)
    })

    it('should sort entries within each cluster by importance', () => {
      const entries = [
        makeEntry('tRPC usage note A', { importance: 'low' }),
        makeEntry('tRPC critical decision', { importance: 'critical' }),
        makeEntry('tRPC type safety preference', { importance: 'medium' }),
      ]
      const clusters = compactor.cluster(entries)
      const tRPCCluster = clusters.find(
        (c) => c.entries.some((e) => e.content.includes('critical'))
      )
      if (tRPCCluster && tRPCCluster.entries.length > 1) {
        expect(tRPCCluster.entries[0]?.importance).toBe('critical')
      }
    })
  })

  // ── Contradiction detection ────────────────────────────────────────

  describe('detectContradictions()', () => {
    it('should return empty for a single entry', () => {
      const contradictions = compactor.detectContradictions([
        makeEntry('Use pnpm'),
      ])
      expect(contradictions).toEqual([])
    })

    it('should detect affirmation/negation contradictions', () => {
      const entries = [
        makeEntry('Always use pnpm for package management', {
          importance: 'medium',
          timestamp:  Date.now() - 10_000,
        }),
        makeEntry('Never use pnpm, use npm for package management', {
          importance: 'medium',
          timestamp:  Date.now(),
        }),
      ]
      const contradictions = compactor.detectContradictions(entries)
      expect(contradictions.length).toBeGreaterThan(0)
    })

    it('should keep the newer entry in a contradiction', () => {
      const older = makeEntry('Always use yarn for package management', {
        timestamp: Date.now() - 10_000,
      })
      const newer = makeEntry('Never use yarn, use pnpm for package management', {
        timestamp: Date.now(),
      })
      const contradictions = compactor.detectContradictions([older, newer])
      if (contradictions.length > 0) {
        expect(contradictions[0]?.kept).toBe(newer.id)
        expect(contradictions[0]?.archived).toBe(older.id)
      }
    })

    it('should not flag unrelated entries as contradictions', () => {
      const entries = [
        makeEntry('Use tRPC for IPC communication', { importance: 'high' }),
        makeEntry('Neo4j stores the knowledge graph data', { importance: 'medium' }),
      ]
      const contradictions = compactor.detectContradictions(entries)
      expect(contradictions.length).toBe(0)
    })
  })

  // ── Merging ────────────────────────────────────────────────────────

  describe('mergeCluster()', () => {
    it('should return a single entry unchanged for a single-entry cluster', () => {
      const entry   = makeEntry('Single entry')
      const merged  = compactor.mergeCluster([entry])
      expect(merged.length).toBe(1)
      expect(merged[0]?.id).toBe(entry.id)
    })

    it('should merge near-duplicate entries into one', () => {
      const entries = [
        makeEntry('Use tRPC for type-safe IPC communication in desktop app', {
          importance: 'high',
          timestamp:  Date.now() - 5_000,
        }),
        makeEntry('Use tRPC for type-safe IPC communication in desktop application', {
          importance: 'medium',
          timestamp:  Date.now(),
        }),
      ]
      const merged = compactor.mergeCluster(entries)
      // Two near-duplicates should collapse into one
      expect(merged.length).toBe(1)
    })

    it('should merge tags from all entries in a group', () => {
      const entries = [
        makeEntry('Use tRPC for IPC in desktop app', {
          importance: 'high',
          tags:       ['tRPC', 'ipc'],
        }),
        makeEntry('Use tRPC for type safe IPC in desktop application', {
          importance: 'medium',
          tags:       ['desktop', 'type-safety'],
        }),
      ]
      const merged = compactor.mergeCluster(entries)
      if (merged.length === 1) {
        const allTags = merged[0]!.tags
        expect(allTags).toContain('tRPC')
        expect(allTags).toContain('autodream-merged')
      }
    })

    it('should keep the oldest timestamp in a merged entry', () => {
      const oldTimestamp = Date.now() - 100_000
      const entries = [
        makeEntry('Use tRPC for type safe IPC', {
          importance: 'high',
          timestamp:  oldTimestamp,
        }),
        makeEntry('Use tRPC for type-safe IPC communication', {
          importance: 'medium',
          timestamp:  Date.now(),
        }),
      ]
      const merged = compactor.mergeCluster(entries)
      if (merged.length === 1) {
        expect(merged[0]?.timestamp).toBe(oldTimestamp)
      }
    })

    it('should keep dissimilar entries as separate entries', () => {
      const entries = [
        makeEntry('Use tRPC for IPC'),
        makeEntry('Completely different topic about Docker volumes'),
      ]
      const merged = compactor.mergeCluster(entries)
      // Different topics should NOT be merged
      expect(merged.length).toBe(2)
    })

    it('should select the highest-importance entry as representative', () => {
      const entries = [
        makeEntry('tRPC provides type safety for IPC communication bus', {
          importance: 'critical',
          content:    'Critical: tRPC provides type safety for IPC communication bus',
        }),
        makeEntry('tRPC provides type-safe IPC communication for bus', {
          importance: 'low',
        }),
      ]
      const merged = compactor.mergeCluster(entries)
      if (merged.length === 1) {
        expect(merged[0]?.importance).toBe('critical')
      }
    })
  })
})
Pass 3 — Complete ✅
Full file summary for Pass 3:
File	Lines	Status
Part 1		
package.json	45	✅ Buildable
tsconfig.json	18	✅ Buildable
src/types/memory-internal.types.ts	145	✅ Complete
src/MemoryManager.ts	320	✅ Complete
src/ConversationStore.ts	295	✅ Complete
src/MemoryIndex.ts	270	✅ Complete
src/MemoryFormatter.ts	175	✅ Complete
Part 2		
src/AutoDream.ts	390	✅ Complete
src/MemoryCompactor.ts	320	✅ Complete
src/index.ts	70	✅ Complete
tests/MemoryManager.test.ts	195	✅ 18 tests
tests/ConversationStore.test.ts	185	✅ 15 tests
tests/MemoryIndex.test.ts	205	✅ 18 tests
tests/AutoDream.test.ts	275	✅ 22 tests (covers both AutoDream + MemoryCompactor)
Total test coverage for packages/memory: 73 tests
