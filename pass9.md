Pass 9 Part 1: packages/autoresearch — Autonomous Research Loop (Foundation)
Division Plan
Part 1 scope (this pass): Core types & errors, ResearchStore (SQLite persistence), ResearchPlanner (goal decomposition → query plan), QueryEngine (multi-source search dispatcher: web, code, graph, wiki, memory), SourceFetcher (URL fetching + content extraction + fingerprinting), and the EvidenceCollector (de-duplication, relevance scoring, evidence graph assembly).

Part 2 scope (next pass): SynthesisEngine (multi-document synthesis via agent loop), ReportWriter (structured Markdown reports + wiki push), AutoResearchLoop (the top-level async-generator orchestrator), Kairos integration (scheduled triggers, observations), ResearchToolSet (tools exposed to the agent via ToolRegistry), CLI commands, and full test suite.

File Tree
text

packages/autoresearch/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   ├── types/
│   │   ├── index.ts
│   │   ├── research.ts          ← core domain types
│   │   ├── evidence.ts          ← evidence + source types
│   │   ├── planning.ts          ← query plan types
│   │   └── errors.ts            ← typed error hierarchy
│   ├── store/
│   │   ├── ResearchStore.ts     ← single-writer SQLite store
│   │   └── schema.sql           ← DB schema
│   ├── planning/
│   │   ├── ResearchPlanner.ts   ← goal → decomposed query plan
│   │   ├── GoalParser.ts        ← extract intent + constraints
│   │   └── QueryTemplates.ts    ← strategy templates per goal type
│   ├── query/
│   │   ├── QueryEngine.ts       ← dispatcher across all sources
│   │   ├── WebSearchAdapter.ts  ← web search (provider-agnostic)
│   │   ├── CodeSearchAdapter.ts ← graphify node/edge queries
│   │   ├── WikiSearchAdapter.ts ← wiki FTS + backlink queries
│   │   └── MemorySearchAdapter.ts ← memory index queries
│   └── evidence/
│       ├── SourceFetcher.ts     ← fetch + extract + fingerprint
│       ├── ContentExtractor.ts  ← HTML → clean Markdown
│       └── EvidenceCollector.ts ← de-dup + score + graph assembly
└── tests/
    ├── store.test.ts
    ├── planner.test.ts
    ├── query-engine.test.ts
    ├── source-fetcher.test.ts
    └── evidence-collector.test.ts
package.json
JSON

{
  "name": "@locoworker/autoresearch",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    ".": "./src/index.ts"
  },
  "scripts": {
    "dev":       "tsx watch src/index.ts",
    "build":     "tsc --build",
    "typecheck": "tsc --noEmit",
    "test":      "bun test",
    "clean":     "rm -rf dist .turbo node_modules"
  },
  "dependencies": {
    "@locoworker/core":         "workspace:*",
    "@locoworker/memory":       "workspace:*",
    "@locoworker/graphify":     "workspace:*",
    "@locoworker/wiki":         "workspace:*",
    "@locoworker/kairos":       "workspace:*",
    "better-sqlite3":           "^9.4.3",
    "zod":                      "^3.22.4",
    "nanoid":                   "^5.0.6",
    "node-fetch":               "^3.3.2",
    "cheerio":                  "^1.0.0-rc.12",
    "turndown":                 "^7.1.3",
    "p-limit":                  "^5.0.0",
    "p-retry":                  "^6.2.0"
  },
  "devDependencies": {
    "@types/node":              "^20.11.30",
    "@types/better-sqlite3":    "^7.6.10",
    "@types/turndown":          "^5.0.4",
    "tsx":                      "^4.7.2",
    "typescript":               "^5.4.3",
    "bun-types":                "^1.0.30"
  }
}
tsconfig.json
JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "references": [
    { "path": "../core"      },
    { "path": "../memory"    },
    { "path": "../graphify"  },
    { "path": "../wiki"      },
    { "path": "../kairos"    }
  ]
}
src/types/research.ts
TypeScript

import { z } from 'zod';

/**
 * Core research domain types.
 *
 * A Research represents a single autonomous investigation lifecycle:
 *   goal → plan → queries → evidence → synthesis → report
 *
 * Design decisions:
 * - Status lifecycle is linear but allows re-runs (re-enters PLANNING).
 * - Depth controls how many "hops" of follow-up queries are executed.
 * - costUsd is tracked per-research to feed the global cost cap.
 * - All timestamps are ISO-8601 strings (consistent with other packages).
 */

export const ResearchStatusSchema = z.enum([
  'pending',      // created, not yet started
  'planning',     // GoalParser + ResearchPlanner running
  'querying',     // QueryEngine dispatching queries
  'fetching',     // SourceFetcher retrieving pages
  'collecting',   // EvidenceCollector de-duplicating + scoring
  'synthesizing', // SynthesisEngine calling agent loop  (Part 2)
  'writing',      // ReportWriter formatting output      (Part 2)
  'complete',     // report available
  'failed',       // unrecoverable error
  'cancelled',    // user/system cancelled
]);
export type ResearchStatus = z.infer<typeof ResearchStatusSchema>;

export const ResearchDepthSchema = z.enum([
  'surface',   // 1 query round, no follow-ups     (~5–10 sources)
  'standard',  // 2 rounds of follow-up queries    (~15–25 sources)
  'deep',      // 3 rounds + cross-validation      (~30–50 sources)
]);
export type ResearchDepth = z.infer<typeof ResearchDepthSchema>;

export const ResearchGoalTypeSchema = z.enum([
  'technical',      // architecture, APIs, patterns, code
  'factual',        // "what is X", definitions, comparisons
  'exploratory',    // "tell me about X", broad survey
  'investigative',  // "why does X happen", root-cause
  'comparative',    // "X vs Y", trade-off analysis
  'procedural',     // "how to do X", step-by-step
]);
export type ResearchGoalType = z.infer<typeof ResearchGoalTypeSchema>;

export const ResearchSchema = z.object({
  id:             z.string(),
  goal:           z.string(),
  goalType:       ResearchGoalTypeSchema,
  depth:          ResearchDepthSchema,
  status:         ResearchStatusSchema,
  tags:           z.array(z.string()),
  sessionId:      z.string().optional(),   // originating agent session
  kairosTaskId:   z.string().optional(),   // kairos task that triggered this
  wikiSlug:       z.string().optional(),   // wiki page created for report
  reportPath:     z.string().optional(),   // filesystem path to .md report
  totalSources:   z.number().int(),
  usedSources:    z.number().int(),
  totalTokens:    z.number().int(),
  costUsd:        z.number(),
  errorMessage:   z.string().optional(),
  createdAt:      z.string().datetime(),
  startedAt:      z.string().datetime().optional(),
  completedAt:    z.string().datetime().optional(),
  metadata:       z.record(z.unknown()).optional(),
});
export type Research = z.infer<typeof ResearchSchema>;

export const CreateResearchSchema = z.object({
  goal:       z.string().min(5),
  depth:      ResearchDepthSchema.default('standard'),
  tags:       z.array(z.string()).default([]),
  sessionId:  z.string().optional(),
  kairosTaskId: z.string().optional(),
  metadata:   z.record(z.unknown()).optional(),
});
export type CreateResearch = z.infer<typeof CreateResearchSchema>;

export const ResearchSummarySchema = z.object({
  id:           z.string(),
  goal:         z.string(),
  goalType:     ResearchGoalTypeSchema,
  depth:        ResearchDepthSchema,
  status:       ResearchStatusSchema,
  totalSources: z.number().int(),
  usedSources:  z.number().int(),
  costUsd:      z.number(),
  createdAt:    z.string().datetime(),
  completedAt:  z.string().datetime().optional(),
  wikiSlug:     z.string().optional(),
});
export type ResearchSummary = z.infer<typeof ResearchSummarySchema>;
src/types/evidence.ts
TypeScript

import { z } from 'zod';

/**
 * Evidence and source types for the autoresearch pipeline.
 *
 * Design decisions:
 * - Sources are fetched content with provenance metadata.
 * - Evidence is a scored, de-duplicated, extracted claim from one or more sources.
 * - EvidenceGraph is the adjacency structure used during synthesis.
 * - ContentFingerprint is SHA-256 of normalized content for de-duplication.
 */

export const SourceTypeSchema = z.enum([
  'web',        // fetched from a URL
  'code',       // extracted from graphify node
  'wiki',       // from local wiki page
  'memory',     // from memory index
  'file',       // from local filesystem
]);
export type SourceType = z.infer<typeof SourceTypeSchema>;

export const SourceSchema = z.object({
  id:            z.string(),
  researchId:    z.string(),
  type:          SourceTypeSchema,
  url:           z.string().optional(),       // for web sources
  nodeId:        z.string().optional(),       // for code sources
  wikiSlug:      z.string().optional(),       // for wiki sources
  memoryEntryId: z.string().optional(),       // for memory sources
  filePath:      z.string().optional(),       // for file sources
  title:         z.string(),
  rawContent:    z.string(),
  cleanContent:  z.string(),                  // HTML → Markdown
  fingerprint:   z.string(),                  // SHA-256 of cleanContent
  tokenCount:    z.number().int(),
  relevanceScore: z.number().min(0).max(1),   // 0.0–1.0
  fetchedAt:     z.string().datetime(),
  httpStatus:    z.number().int().optional(), // for web sources
  metadata:      z.record(z.unknown()).optional(),
});
export type Source = z.infer<typeof SourceSchema>;

export const EvidenceTypeSchema = z.enum([
  'fact',          // a verifiable claim
  'definition',    // what X is
  'example',       // concrete instance
  'comparison',    // X vs Y
  'procedure',     // steps to do X
  'opinion',       // subjective stance
  'code_pattern',  // code-level pattern
  'warning',       // caution / known issue
  'contradiction', // conflicts with another piece of evidence
]);
export type EvidenceType = z.infer<typeof EvidenceTypeSchema>;

export const EvidenceSchema = z.object({
  id:           z.string(),
  researchId:   z.string(),
  sourceIds:    z.array(z.string()),   // one or more sources supporting this
  type:         EvidenceTypeSchema,
  claim:        z.string(),            // the extracted statement
  context:      z.string(),            // surrounding passage for synthesis
  tags:         z.array(z.string()),
  confidence:   z.number().min(0).max(1),
  relevance:    z.number().min(0).max(1),
  round:        z.number().int().min(1), // which query round produced this
  contradicts:  z.array(z.string()),   // IDs of contradicting evidence
  supports:     z.array(z.string()),   // IDs of corroborating evidence
  createdAt:    z.string().datetime(),
});
export type Evidence = z.infer<typeof EvidenceSchema>;

/**
 * Lightweight graph structure passed to SynthesisEngine.
 * Nodes = evidence IDs, edges = supports/contradicts relationships.
 */
export interface EvidenceGraph {
  researchId:    string;
  nodes:         string[];                        // evidence IDs
  supports:      Array<{ from: string; to: string; weight: number }>;
  contradicts:   Array<{ from: string; to: string; weight: number }>;
  clusters:      Array<{ label: string; nodeIds: string[] }>;
}

/**
 * De-duplication result from EvidenceCollector.
 */
export interface DeduplicationResult {
  total:       number;
  unique:      number;
  duplicates:  number;
  merged:      number;
}
src/types/planning.ts
TypeScript

import { z } from 'zod';

/**
 * Query planning types.
 *
 * A QueryPlan is a tree of rounds.
 * Each round contains a set of queries dispatched in parallel.
 * Follow-up rounds are generated based on gaps in the evidence from prior rounds.
 */

export const QuerySourceSchema = z.enum([
  'web',
  'code',
  'wiki',
  'memory',
]);
export type QuerySource = z.infer<typeof QuerySourceSchema>;

export const QueryStrategySchema = z.enum([
  'broad_survey',          // wide net, many short queries
  'targeted_lookup',       // few precise queries
  'comparative_sweep',     // pairwise comparison queries
  'code_archaeology',      // trace symbol usages, call graphs
  'contradiction_hunt',    // look for conflicting evidence
  'gap_fill',              // follow-up to fill identified gaps
]);
export type QueryStrategy = z.infer<typeof QueryStrategySchema>;

export const QuerySchema = z.object({
  id:          z.string(),
  planId:      z.string(),
  round:       z.number().int().min(1),
  source:      QuerySourceSchema,
  strategy:    QueryStrategySchema,
  text:        z.string(),              // the actual search query string
  priority:    z.number().int().min(1).max(10),
  maxResults:  z.number().int().min(1).max(20),
  executed:    z.boolean().default(false),
  resultCount: z.number().int().default(0),
  executedAt:  z.string().datetime().optional(),
});
export type Query = z.infer<typeof QuerySchema>;

export const QueryPlanSchema = z.object({
  id:          z.string(),
  researchId:  z.string(),
  goalType:    z.string(),
  depth:       z.string(),
  rounds:      z.number().int().min(1).max(3),
  queries:     z.array(QuerySchema),
  gaps:        z.array(z.string()),    // identified knowledge gaps after each round
  createdAt:   z.string().datetime(),
  updatedAt:   z.string().datetime(),
});
export type QueryPlan = z.infer<typeof QueryPlanSchema>;

/**
 * Parsed goal metadata extracted by GoalParser.
 */
export interface ParsedGoal {
  rawGoal:       string;
  goalType:      string;
  keyTerms:      string[];        // extracted nouns, identifiers, versions
  constraints:   string[];        // date ranges, languages, frameworks
  exclusions:    string[];        // things explicitly out of scope
  intent:        string;          // one-sentence rephrasing
  comparands:    string[];        // for comparative goals: [X, Y]
  subQuestions:  string[];        // decomposed sub-questions
}

/**
 * Metrics returned by ResearchPlanner after plan generation.
 */
export interface PlanStats {
  totalQueries:    number;
  queriesBySource: Record<string, number>;
  queriesByRound:  Record<number, number>;
  estimatedTokens: number;        // rough pre-flight estimate
  estimatedCostUsd: number;
}
src/types/errors.ts
TypeScript

/**
 * AutoResearch typed error hierarchy.
 * Consistent with core's error patterns (Pass 2) and gateway's GatewayError.
 */

export class AutoResearchError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly researchId?: string,
    public readonly details?: unknown,
  ) {
    super(message);
    this.name = 'AutoResearchError';
  }
}

export class ResearchNotFoundError extends AutoResearchError {
  constructor(researchId: string) {
    super(
      `Research not found: ${researchId}`,
      'RESEARCH_NOT_FOUND',
      researchId,
    );
    this.name = 'ResearchNotFoundError';
  }
}

export class ResearchAlreadyRunningError extends AutoResearchError {
  constructor(researchId: string) {
    super(
      `Research already running: ${researchId}`,
      'RESEARCH_ALREADY_RUNNING',
      researchId,
    );
    this.name = 'ResearchAlreadyRunningError';
  }
}

export class GoalParseError extends AutoResearchError {
  constructor(goal: string, details?: unknown) {
    super(
      `Failed to parse research goal: "${goal.slice(0, 80)}"`,
      'GOAL_PARSE_ERROR',
      undefined,
      details,
    );
    this.name = 'GoalParseError';
  }
}

export class PlanGenerationError extends AutoResearchError {
  constructor(researchId: string, details?: unknown) {
    super(
      `Failed to generate query plan for research: ${researchId}`,
      'PLAN_GENERATION_ERROR',
      researchId,
      details,
    );
    this.name = 'PlanGenerationError';
  }
}

export class QueryExecutionError extends AutoResearchError {
  constructor(queryId: string, source: string, details?: unknown) {
    super(
      `Query execution failed — id: ${queryId}, source: ${source}`,
      'QUERY_EXECUTION_ERROR',
      undefined,
      details,
    );
    this.name = 'QueryExecutionError';
  }
}

export class SourceFetchError extends AutoResearchError {
  constructor(url: string, status?: number, details?: unknown) {
    super(
      `Failed to fetch source: ${url}${status ? ` (HTTP ${status})` : ''}`,
      'SOURCE_FETCH_ERROR',
      undefined,
      details,
    );
    this.name = 'SourceFetchError';
  }
}

export class ContentExtractionError extends AutoResearchError {
  constructor(sourceId: string, details?: unknown) {
    super(
      `Content extraction failed for source: ${sourceId}`,
      'CONTENT_EXTRACTION_ERROR',
      undefined,
      details,
    );
    this.name = 'ContentExtractionError';
  }
}

export class EvidenceCollectionError extends AutoResearchError {
  constructor(researchId: string, details?: unknown) {
    super(
      `Evidence collection failed for research: ${researchId}`,
      'EVIDENCE_COLLECTION_ERROR',
      researchId,
      details,
    );
    this.name = 'EvidenceCollectionError';
  }
}

export class ResearchCostCapError extends AutoResearchError {
  constructor(researchId: string, costUsd: number, capUsd: number) {
    super(
      `Research cost cap exceeded: $${costUsd.toFixed(4)} > cap $${capUsd.toFixed(4)}`,
      'RESEARCH_COST_CAP_EXCEEDED',
      researchId,
      { costUsd, capUsd },
    );
    this.name = 'ResearchCostCapError';
  }
}

export class WebSearchProviderError extends AutoResearchError {
  constructor(provider: string, details?: unknown) {
    super(
      `Web search provider error: ${provider}`,
      'WEB_SEARCH_PROVIDER_ERROR',
      undefined,
      details,
    );
    this.name = 'WebSearchProviderError';
  }
}
src/types/index.ts
TypeScript

export * from './research.js';
export * from './evidence.js';
export * from './planning.js';
export * from './errors.js';
src/store/schema.sql
SQL

-- ─────────────────────────────────────────────────────────────────────────────
-- Research table: one row per autonomous research lifecycle
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS researches (
  id              TEXT PRIMARY KEY,
  goal            TEXT NOT NULL,
  goal_type       TEXT NOT NULL,
  depth           TEXT NOT NULL,
  status          TEXT NOT NULL DEFAULT 'pending',
  tags            TEXT NOT NULL DEFAULT '[]',   -- JSON array
  session_id      TEXT,
  kairos_task_id  TEXT,
  wiki_slug       TEXT,
  report_path     TEXT,
  total_sources   INTEGER NOT NULL DEFAULT 0,
  used_sources    INTEGER NOT NULL DEFAULT 0,
  total_tokens    INTEGER NOT NULL DEFAULT 0,
  cost_usd        REAL    NOT NULL DEFAULT 0,
  error_message   TEXT,
  created_at      TEXT NOT NULL,
  started_at      TEXT,
  completed_at    TEXT,
  metadata        TEXT                          -- JSON blob
);

CREATE INDEX IF NOT EXISTS idx_researches_status     ON researches(status);
CREATE INDEX IF NOT EXISTS idx_researches_created_at ON researches(created_at);
CREATE INDEX IF NOT EXISTS idx_researches_goal_type  ON researches(goal_type);

-- ─────────────────────────────────────────────────────────────────────────────
-- Query plans: one per research
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS query_plans (
  id           TEXT PRIMARY KEY,
  research_id  TEXT NOT NULL UNIQUE,
  goal_type    TEXT NOT NULL,
  depth        TEXT NOT NULL,
  rounds       INTEGER NOT NULL DEFAULT 1,
  gaps         TEXT NOT NULL DEFAULT '[]',  -- JSON array of gap strings
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  FOREIGN KEY (research_id) REFERENCES researches(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_plans_research ON query_plans(research_id);

-- ─────────────────────────────────────────────────────────────────────────────
-- Queries: many per plan
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS queries (
  id           TEXT PRIMARY KEY,
  plan_id      TEXT NOT NULL,
  research_id  TEXT NOT NULL,
  round        INTEGER NOT NULL DEFAULT 1,
  source       TEXT NOT NULL,
  strategy     TEXT NOT NULL,
  text         TEXT NOT NULL,
  priority     INTEGER NOT NULL DEFAULT 5,
  max_results  INTEGER NOT NULL DEFAULT 10,
  executed     INTEGER NOT NULL DEFAULT 0,    -- boolean
  result_count INTEGER NOT NULL DEFAULT 0,
  executed_at  TEXT,
  FOREIGN KEY (plan_id)     REFERENCES query_plans(id)  ON DELETE CASCADE,
  FOREIGN KEY (research_id) REFERENCES researches(id)   ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_queries_plan       ON queries(plan_id);
CREATE INDEX IF NOT EXISTS idx_queries_research   ON queries(research_id);
CREATE INDEX IF NOT EXISTS idx_queries_round      ON queries(round);
CREATE INDEX IF NOT EXISTS idx_queries_executed   ON queries(executed);

-- ─────────────────────────────────────────────────────────────────────────────
-- Sources: fetched content with provenance
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS sources (
  id              TEXT PRIMARY KEY,
  research_id     TEXT NOT NULL,
  type            TEXT NOT NULL,
  url             TEXT,
  node_id         TEXT,
  wiki_slug       TEXT,
  memory_entry_id TEXT,
  file_path       TEXT,
  title           TEXT NOT NULL,
  raw_content     TEXT NOT NULL,
  clean_content   TEXT NOT NULL,
  fingerprint     TEXT NOT NULL,
  token_count     INTEGER NOT NULL DEFAULT 0,
  relevance_score REAL NOT NULL DEFAULT 0,
  fetched_at      TEXT NOT NULL,
  http_status     INTEGER,
  metadata        TEXT,                       -- JSON blob
  FOREIGN KEY (research_id) REFERENCES researches(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_sources_research     ON sources(research_id);
CREATE INDEX IF NOT EXISTS idx_sources_fingerprint  ON sources(fingerprint);
CREATE INDEX IF NOT EXISTS idx_sources_type         ON sources(type);

-- ─────────────────────────────────────────────────────────────────────────────
-- Evidence: scored, de-duplicated claims
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS evidence (
  id           TEXT PRIMARY KEY,
  research_id  TEXT NOT NULL,
  source_ids   TEXT NOT NULL DEFAULT '[]',   -- JSON array
  type         TEXT NOT NULL,
  claim        TEXT NOT NULL,
  context      TEXT NOT NULL,
  tags         TEXT NOT NULL DEFAULT '[]',   -- JSON array
  confidence   REAL NOT NULL DEFAULT 0.5,
  relevance    REAL NOT NULL DEFAULT 0.5,
  round        INTEGER NOT NULL DEFAULT 1,
  contradicts  TEXT NOT NULL DEFAULT '[]',   -- JSON array of IDs
  supports     TEXT NOT NULL DEFAULT '[]',   -- JSON array of IDs
  created_at   TEXT NOT NULL,
  FOREIGN KEY (research_id) REFERENCES researches(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_evidence_research    ON evidence(research_id);
CREATE INDEX IF NOT EXISTS idx_evidence_type        ON evidence(type);
CREATE INDEX IF NOT EXISTS idx_evidence_round       ON evidence(round);
CREATE INDEX IF NOT EXISTS idx_evidence_relevance   ON evidence(relevance);

-- ─────────────────────────────────────────────────────────────────────────────
-- Schema meta
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS autoresearch_meta (
  key        TEXT PRIMARY KEY,
  value      TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
src/store/ResearchStore.ts
TypeScript

import Database from 'better-sqlite3';
import { nanoid } from 'nanoid';
import { readFileSync } from 'node:fs';
import { join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import type {
  Research,
  ResearchStatus,
  ResearchSummary,
  CreateResearch,
} from '../types/research.js';
import type { Source } from '../types/evidence.js';
import type { Evidence } from '../types/evidence.js';
import type { QueryPlan, Query } from '../types/planning.js';
import { ResearchSchema, ResearchSummarySchema } from '../types/research.js';
import { ResearchNotFoundError } from '../types/errors.js';

const __dirname = dirname(fileURLToPath(import.meta.url));

/**
 * ResearchStore: single-writer SQLite store for the autoresearch pipeline.
 *
 * Consistent design pattern used across:
 *   WikiStore, GatewayStore, OrchestratorStore, KairosStore.
 *
 * Key choices:
 * - WAL mode for read concurrency during long-running research loops.
 * - Atomic upserts via INSERT … ON CONFLICT.
 * - JSON columns for arrays (tags, source_ids, contradicts, supports, gaps).
 * - Zod validation on all read paths; corrupted rows are logged and skipped.
 * - Cascade deletes: removing a Research removes all its Plans/Queries/Sources/Evidence.
 */
export class ResearchStore {
  private db: Database.Database;

  constructor(dbPath: string) {
    this.db = new Database(dbPath);
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('foreign_keys = ON');
    this.initSchema();
  }

  // ============================================================================
  // Schema
  // ============================================================================

  private initSchema(): void {
    const schema = readFileSync(
      join(__dirname, 'schema.sql'),
      'utf8',
    );
    this.db.exec(schema);

    this.db
      .prepare(
        `INSERT INTO autoresearch_meta (key, value, updated_at)
         VALUES ('schema_version', '1', ?)
         ON CONFLICT(key) DO NOTHING`,
      )
      .run(new Date().toISOString());
  }

  // ============================================================================
  // Research CRUD
  // ============================================================================

  /**
   * Create a new Research record.
   */
  createResearch(input: CreateResearch & { goalType: string }): Research {
    const id = nanoid();
    const now = new Date().toISOString();

    this.db
      .prepare(
        `INSERT INTO researches
         (id, goal, goal_type, depth, status, tags, session_id,
          kairos_task_id, total_sources, used_sources, total_tokens,
          cost_usd, created_at, metadata)
         VALUES (?, ?, ?, ?, 'pending', ?, ?, ?, 0, 0, 0, 0, ?, ?)`,
      )
      .run(
        id,
        input.goal,
        input.goalType,
        input.depth,
        JSON.stringify(input.tags),
        input.sessionId ?? null,
        input.kairosTaskId ?? null,
        now,
        input.metadata ? JSON.stringify(input.metadata) : null,
      );

    return this.getResearchOrThrow(id);
  }

  /**
   * Get a Research by ID. Returns null if not found.
   */
  getResearch(id: string): Research | null {
    const row = this.db
      .prepare(`SELECT * FROM researches WHERE id = ?`)
      .get(id);

    if (!row) return null;
    return this.rowToResearch(row as Record<string, unknown>);
  }

  /**
   * Get a Research by ID, throwing if not found.
   */
  getResearchOrThrow(id: string): Research {
    const research = this.getResearch(id);
    if (!research) throw new ResearchNotFoundError(id);
    return research;
  }

  /**
   * List researches with optional filters.
   */
  listResearches(opts: {
    status?: ResearchStatus;
    goalType?: string;
    tags?: string[];
    limit?: number;
    offset?: number;
  } = {}): ResearchSummary[] {
    const conditions: string[] = [];
    const params: unknown[] = [];

    if (opts.status) {
      conditions.push('status = ?');
      params.push(opts.status);
    }
    if (opts.goalType) {
      conditions.push('goal_type = ?');
      params.push(opts.goalType);
    }

    const where = conditions.length ? `WHERE ${conditions.join(' AND ')}` : '';
    const limit  = opts.limit  ?? 20;
    const offset = opts.offset ?? 0;

    const rows = this.db
      .prepare(
        `SELECT id, goal, goal_type, depth, status, total_sources,
                used_sources, cost_usd, created_at, completed_at, wiki_slug
         FROM researches
         ${where}
         ORDER BY created_at DESC
         LIMIT ? OFFSET ?`,
      )
      .all(...params, limit, offset) as Record<string, unknown>[];

    return rows.map(row => ({
      id:           String(row.id),
      goal:         String(row.goal),
      goalType:     String(row.goal_type) as any,
      depth:        String(row.depth) as any,
      status:       String(row.status) as any,
      totalSources: Number(row.total_sources),
      usedSources:  Number(row.used_sources),
      costUsd:      Number(row.cost_usd),
      createdAt:    String(row.created_at),
      completedAt:  row.completed_at ? String(row.completed_at) : undefined,
      wikiSlug:     row.wiki_slug ? String(row.wiki_slug) : undefined,
    }));
  }

  /**
   * Update Research status (with optional fields).
   */
  updateStatus(
    id: string,
    status: ResearchStatus,
    extra: {
      errorMessage?:  string;
      startedAt?:     string;
      completedAt?:   string;
      wikiSlug?:      string;
      reportPath?:    string;
      totalSources?:  number;
      usedSources?:   number;
      totalTokens?:   number;
      costUsd?:       number;
    } = {},
  ): void {
    const setClauses = ['status = ?'];
    const params: unknown[] = [status];

    if (extra.errorMessage  !== undefined) { setClauses.push('error_message = ?');  params.push(extra.errorMessage); }
    if (extra.startedAt     !== undefined) { setClauses.push('started_at = ?');      params.push(extra.startedAt); }
    if (extra.completedAt   !== undefined) { setClauses.push('completed_at = ?');    params.push(extra.completedAt); }
    if (extra.wikiSlug      !== undefined) { setClauses.push('wiki_slug = ?');       params.push(extra.wikiSlug); }
    if (extra.reportPath    !== undefined) { setClauses.push('report_path = ?');     params.push(extra.reportPath); }
    if (extra.totalSources  !== undefined) { setClauses.push('total_sources = ?');   params.push(extra.totalSources); }
    if (extra.usedSources   !== undefined) { setClauses.push('used_sources = ?');    params.push(extra.usedSources); }
    if (extra.totalTokens   !== undefined) { setClauses.push('total_tokens = ?');    params.push(extra.totalTokens); }
    if (extra.costUsd       !== undefined) { setClauses.push('cost_usd = ?');        params.push(extra.costUsd); }

    params.push(id);

    this.db
      .prepare(`UPDATE researches SET ${setClauses.join(', ')} WHERE id = ?`)
      .run(...params);
  }

  /**
   * Accumulate cost + tokens for a running research.
   */
  accumulateCost(id: string, tokens: number, costUsd: number): void {
    this.db
      .prepare(
        `UPDATE researches
         SET total_tokens = total_tokens + ?,
             cost_usd = cost_usd + ?
         WHERE id = ?`,
      )
      .run(tokens, costUsd, id);
  }

  // ============================================================================
  // Query Plan CRUD
  // ============================================================================

  savePlan(plan: Omit<QueryPlan, 'queries'>): void {
    this.db
      .prepare(
        `INSERT INTO query_plans
         (id, research_id, goal_type, depth, rounds, gaps, created_at, updated_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(id) DO UPDATE SET
           rounds = excluded.rounds,
           gaps = excluded.gaps,
           updated_at = excluded.updated_at`,
      )
      .run(
        plan.id,
        plan.researchId,
        plan.goalType,
        plan.depth,
        plan.rounds,
        JSON.stringify(plan.gaps),
        plan.createdAt,
        plan.updatedAt,
      );
  }

  saveQuery(query: Query): void {
    const planRow = this.db
      .prepare(`SELECT research_id FROM query_plans WHERE id = ?`)
      .get(query.planId) as { research_id: string } | undefined;

    const researchId = planRow?.research_id ?? '';

    this.db
      .prepare(
        `INSERT INTO queries
         (id, plan_id, research_id, round, source, strategy, text,
          priority, max_results, executed, result_count, executed_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(id) DO UPDATE SET
           executed = excluded.executed,
           result_count = excluded.result_count,
           executed_at = excluded.executed_at`,
      )
      .run(
        query.id,
        query.planId,
        researchId,
        query.round,
        query.source,
        query.strategy,
        query.text,
        query.priority,
        query.maxResults,
        query.executed ? 1 : 0,
        query.resultCount,
        query.executedAt ?? null,
      );
  }

  markQueryExecuted(queryId: string, resultCount: number): void {
    this.db
      .prepare(
        `UPDATE queries
         SET executed = 1, result_count = ?, executed_at = ?
         WHERE id = ?`,
      )
      .run(resultCount, new Date().toISOString(), queryId);
  }

  getPlan(researchId: string): QueryPlan | null {
    const planRow = this.db
      .prepare(`SELECT * FROM query_plans WHERE research_id = ?`)
      .get(researchId) as Record<string, unknown> | undefined;

    if (!planRow) return null;

    const queryRows = this.db
      .prepare(`SELECT * FROM queries WHERE plan_id = ? ORDER BY round, priority DESC`)
      .all(String(planRow.id)) as Record<string, unknown>[];

    return {
      id:         String(planRow.id),
      researchId: String(planRow.research_id),
      goalType:   String(planRow.goal_type),
      depth:      String(planRow.depth) as any,
      rounds:     Number(planRow.rounds),
      queries:    queryRows.map(r => this.rowToQuery(r)),
      gaps:       JSON.parse(String(planRow.gaps)) as string[],
      createdAt:  String(planRow.created_at),
      updatedAt:  String(planRow.updated_at),
    };
  }

  getPendingQueriesForRound(researchId: string, round: number): Query[] {
    const rows = this.db
      .prepare(
        `SELECT * FROM queries
         WHERE research_id = ? AND round = ? AND executed = 0
         ORDER BY priority DESC`,
      )
      .all(researchId, round) as Record<string, unknown>[];

    return rows.map(r => this.rowToQuery(r));
  }

  // ============================================================================
  // Source CRUD
  // ============================================================================

  saveSource(source: Source): void {
    this.db
      .prepare(
        `INSERT INTO sources
         (id, research_id, type, url, node_id, wiki_slug, memory_entry_id,
          file_path, title, raw_content, clean_content, fingerprint,
          token_count, relevance_score, fetched_at, http_status, metadata)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(id) DO NOTHING`,
      )
      .run(
        source.id,
        source.researchId,
        source.type,
        source.url ?? null,
        source.nodeId ?? null,
        source.wikiSlug ?? null,
        source.memoryEntryId ?? null,
        source.filePath ?? null,
        source.title,
        source.rawContent,
        source.cleanContent,
        source.fingerprint,
        source.tokenCount,
        source.relevanceScore,
        source.fetchedAt,
        source.httpStatus ?? null,
        source.metadata ? JSON.stringify(source.metadata) : null,
      );
  }

  /**
   * Check if a fingerprint already exists for this research (de-duplication).
   */
  fingerprintExists(researchId: string, fingerprint: string): boolean {
    const row = this.db
      .prepare(
        `SELECT id FROM sources WHERE research_id = ? AND fingerprint = ?`,
      )
      .get(researchId, fingerprint);

    return row !== undefined;
  }

  getSources(researchId: string, opts: {
    type?: string;
    minRelevance?: number;
    limit?: number;
  } = {}): Source[] {
    const conditions = ['research_id = ?'];
    const params: unknown[] = [researchId];

    if (opts.type) {
      conditions.push('type = ?');
      params.push(opts.type);
    }
    if (opts.minRelevance !== undefined) {
      conditions.push('relevance_score >= ?');
      params.push(opts.minRelevance);
    }

    const limit = opts.limit ?? 100;

    const rows = this.db
      .prepare(
        `SELECT * FROM sources
         WHERE ${conditions.join(' AND ')}
         ORDER BY relevance_score DESC
         LIMIT ?`,
      )
      .all(...params, limit) as Record<string, unknown>[];

    return rows.map(r => this.rowToSource(r));
  }

  incrementSourceCount(researchId: string): void {
    this.db
      .prepare(
        `UPDATE researches SET total_sources = total_sources + 1 WHERE id = ?`,
      )
      .run(researchId);
  }

  // ============================================================================
  // Evidence CRUD
  // ============================================================================

  saveEvidence(ev: Evidence): void {
    this.db
      .prepare(
        `INSERT INTO evidence
         (id, research_id, source_ids, type, claim, context, tags,
          confidence, relevance, round, contradicts, supports, created_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
         ON CONFLICT(id) DO NOTHING`,
      )
      .run(
        ev.id,
        ev.researchId,
        JSON.stringify(ev.sourceIds),
        ev.type,
        ev.claim,
        ev.context,
        JSON.stringify(ev.tags),
        ev.confidence,
        ev.relevance,
        ev.round,
        JSON.stringify(ev.contradicts),
        JSON.stringify(ev.supports),
        ev.createdAt,
      );
  }

  getEvidence(researchId: string, opts: {
    type?: string;
    minRelevance?: number;
    round?: number;
    limit?: number;
  } = {}): Evidence[] {
    const conditions = ['research_id = ?'];
    const params: unknown[] = [researchId];

    if (opts.type)          { conditions.push('type = ?');           params.push(opts.type); }
    if (opts.round)         { conditions.push('round = ?');          params.push(opts.round); }
    if (opts.minRelevance !== undefined) {
      conditions.push('relevance >= ?');
      params.push(opts.minRelevance);
    }

    const limit = opts.limit ?? 200;

    const rows = this.db
      .prepare(
        `SELECT * FROM evidence
         WHERE ${conditions.join(' AND ')}
         ORDER BY relevance DESC, confidence DESC
         LIMIT ?`,
      )
      .all(...params, limit) as Record<string, unknown>[];

    return rows.map(r => this.rowToEvidence(r));
  }

  linkEvidenceRelationships(
    id: string,
    contradicts: string[],
    supports: string[],
  ): void {
    this.db
      .prepare(
        `UPDATE evidence
         SET contradicts = ?, supports = ?
         WHERE id = ?`,
      )
      .run(JSON.stringify(contradicts), JSON.stringify(supports), id);
  }

  getEvidenceCount(researchId: string): number {
    const row = this.db
      .prepare(`SELECT COUNT(*) as count FROM evidence WHERE research_id = ?`)
      .get(researchId) as { count: number };

    return row.count;
  }

  // ============================================================================
  // Row mappers
  // ============================================================================

  private rowToResearch(row: Record<string, unknown>): Research {
    return {
      id:           String(row.id),
      goal:         String(row.goal),
      goalType:     String(row.goal_type) as any,
      depth:        String(row.depth) as any,
      status:       String(row.status) as any,
      tags:         JSON.parse(String(row.tags ?? '[]')),
      sessionId:    row.session_id     ? String(row.session_id)     : undefined,
      kairosTaskId: row.kairos_task_id ? String(row.kairos_task_id) : undefined,
      wikiSlug:     row.wiki_slug      ? String(row.wiki_slug)      : undefined,
      reportPath:   row.report_path    ? String(row.report_path)    : undefined,
      totalSources: Number(row.total_sources),
      usedSources:  Number(row.used_sources),
      totalTokens:  Number(row.total_tokens),
      costUsd:      Number(row.cost_usd),
      errorMessage: row.error_message ? String(row.error_message) : undefined,
      createdAt:    String(row.created_at),
      startedAt:    row.started_at   ? String(row.started_at)   : undefined,
      completedAt:  row.completed_at ? String(row.completed_at) : undefined,
      metadata:     row.metadata     ? JSON.parse(String(row.metadata)) : undefined,
    };
  }

  private rowToQuery(row: Record<string, unknown>): Query {
    return {
      id:          String(row.id),
      planId:      String(row.plan_id),
      round:       Number(row.round),
      source:      String(row.source) as any,
      strategy:    String(row.strategy) as any,
      text:        String(row.text),
      priority:    Number(row.priority),
      maxResults:  Number(row.max_results),
      executed:    Number(row.executed) === 1,
      resultCount: Number(row.result_count),
      executedAt:  row.executed_at ? String(row.executed_at) : undefined,
    };
  }

  private rowToSource(row: Record<string, unknown>): Source {
    return {
      id:             String(row.id),
      researchId:     String(row.research_id),
      type:           String(row.type) as any,
      url:            row.url            ? String(row.url)            : undefined,
      nodeId:         row.node_id        ? String(row.node_id)        : undefined,
      wikiSlug:       row.wiki_slug      ? String(row.wiki_slug)      : undefined,
      memoryEntryId:  row.memory_entry_id ? String(row.memory_entry_id) : undefined,
      filePath:       row.file_path      ? String(row.file_path)      : undefined,
      title:          String(row.title),
      rawContent:     String(row.raw_content),
      cleanContent:   String(row.clean_content),
      fingerprint:    String(row.fingerprint),
      tokenCount:     Number(row.token_count),
      relevanceScore: Number(row.relevance_score),
      fetchedAt:      String(row.fetched_at),
      httpStatus:     row.http_status ? Number(row.http_status) : undefined,
      metadata:       row.metadata ? JSON.parse(String(row.metadata)) : undefined,
    };
  }

  private rowToEvidence(row: Record<string, unknown>): Evidence {
    return {
      id:          String(row.id),
      researchId:  String(row.research_id),
      sourceIds:   JSON.parse(String(row.source_ids ?? '[]')),
      type:        String(row.type) as any,
      claim:       String(row.claim),
      context:     String(row.context),
      tags:        JSON.parse(String(row.tags ?? '[]')),
      confidence:  Number(row.confidence),
      relevance:   Number(row.relevance),
      round:       Number(row.round),
      contradicts: JSON.parse(String(row.contradicts ?? '[]')),
      supports:    JSON.parse(String(row.supports ?? '[]')),
      createdAt:   String(row.created_at),
    };
  }

  // ============================================================================
  // Meta + close
  // ============================================================================

  getMeta(key: string): string | null {
    const row = this.db
      .prepare(`SELECT value FROM autoresearch_meta WHERE key = ?`)
      .get(key) as { value: string } | undefined;
    return row?.value ?? null;
  }

  setMeta(key: string, value: string): void {
    this.db
      .prepare(
        `INSERT INTO autoresearch_meta (key, value, updated_at)
         VALUES (?, ?, ?)
         ON CONFLICT(key) DO UPDATE SET value = excluded.value, updated_at = excluded.updated_at`,
      )
      .run(key, value, new Date().toISOString());
  }

  close(): void {
    this.db.close();
  }
}
src/planning/QueryTemplates.ts
TypeScript

import type { ResearchGoalType, ResearchDepth } from '../types/research.js';
import type { QuerySource, QueryStrategy } from '../types/planning.js';

/**
 * Query template library: maps goal type + depth to a structured set of
 * query templates with source, strategy, priority, and max result hints.
 *
 * Templates use {TERM}, {COMPARAND_A}, {COMPARAND_B}, {SUBQ} as
 * interpolation tokens filled by ResearchPlanner.
 */

export interface QueryTemplate {
  source:     QuerySource;
  strategy:   QueryStrategy;
  template:   string;
  priority:   number;   // 1–10 (10 = highest)
  maxResults: number;
  round:      number;   // 1 = initial, 2+ = follow-up
}

/**
 * Surface depth: 1 round, broad.
 */
const SURFACE_TEMPLATES: QueryTemplate[] = [
  { source: 'web',    strategy: 'broad_survey',    template: '{TERM} overview',                    priority: 9,  maxResults: 10, round: 1 },
  { source: 'web',    strategy: 'broad_survey',    template: '{TERM} introduction guide',          priority: 8,  maxResults: 8,  round: 1 },
  { source: 'wiki',   strategy: 'targeted_lookup', template: '{TERM}',                             priority: 8,  maxResults: 5,  round: 1 },
  { source: 'memory', strategy: 'targeted_lookup', template: '{TERM}',                             priority: 7,  maxResults: 5,  round: 1 },
  { source: 'code',   strategy: 'code_archaeology',template: '{TERM}',                             priority: 6,  maxResults: 5,  round: 1 },
];

/**
 * Standard depth: 2 rounds.
 */
const STANDARD_TEMPLATES: QueryTemplate[] = [
  // Round 1 — initial sweep
  { source: 'web',    strategy: 'broad_survey',    template: '{TERM} overview',                    priority: 10, maxResults: 10, round: 1 },
  { source: 'web',    strategy: 'broad_survey',    template: '{TERM} best practices',              priority: 9,  maxResults: 8,  round: 1 },
  { source: 'web',    strategy: 'targeted_lookup', template: 'how does {TERM} work',               priority: 8,  maxResults: 8,  round: 1 },
  { source: 'wiki',   strategy: 'targeted_lookup', template: '{TERM}',                             priority: 9,  maxResults: 5,  round: 1 },
  { source: 'memory', strategy: 'targeted_lookup', template: '{TERM}',                             priority: 8,  maxResults: 10, round: 1 },
  { source: 'code',   strategy: 'code_archaeology',template: '{TERM}',                             priority: 7,  maxResults: 10, round: 1 },
  // Round 2 — fill gaps
  { source: 'web',    strategy: 'gap_fill',        template: '{SUBQ}',                             priority: 9,  maxResults: 8,  round: 2 },
  { source: 'web',    strategy: 'targeted_lookup', template: '{TERM} limitations drawbacks',       priority: 7,  maxResults: 6,  round: 2 },
  { source: 'web',    strategy: 'targeted_lookup', template: '{TERM} examples real-world',         priority: 6,  maxResults: 6,  round: 2 },
  { source: 'code',   strategy: 'code_archaeology',template: '{TERM} usage pattern',               priority: 6,  maxResults: 8,  round: 2 },
];

/**
 * Deep depth: 3 rounds + contradiction hunting.
 */
const DEEP_TEMPLATES: QueryTemplate[] = [
  ...STANDARD_TEMPLATES,
  // Round 2 extras
  { source: 'web',    strategy: 'targeted_lookup', template: '{TERM} performance benchmark',       priority: 7,  maxResults: 6,  round: 2 },
  { source: 'web',    strategy: 'targeted_lookup', template: '{TERM} security concerns',           priority: 7,  maxResults: 6,  round: 2 },
  // Round 3 — cross-validation + contradiction
  { source: 'web',    strategy: 'contradiction_hunt', template: '{TERM} criticism alternatives',   priority: 8,  maxResults: 8,  round: 3 },
  { source: 'web',    strategy: 'contradiction_hunt', template: 'against {TERM} why not use',      priority: 7,  maxResults: 6,  round: 3 },
  { source: 'wiki',   strategy: 'gap_fill',           template: '{SUBQ}',                          priority: 7,  maxResults: 5,  round: 3 },
  { source: 'memory', strategy: 'contradiction_hunt', template: '{TERM} conflict issue',           priority: 6,  maxResults: 5,  round: 3 },
  { source: 'code',   strategy: 'code_archaeology',   template: '{TERM} anti-pattern',             priority: 6,  maxResults: 8,  round: 3 },
];

/**
 * Comparative goal overlay: injects COMPARAND_A vs COMPARAND_B queries.
 */
const COMPARATIVE_OVERLAY: QueryTemplate[] = [
  { source: 'web',  strategy: 'comparative_sweep', template: '{COMPARAND_A} vs {COMPARAND_B}',                priority: 10, maxResults: 10, round: 1 },
  { source: 'web',  strategy: 'comparative_sweep', template: '{COMPARAND_A} vs {COMPARAND_B} comparison',    priority: 9,  maxResults: 8,  round: 1 },
  { source: 'web',  strategy: 'comparative_sweep', template: '{COMPARAND_A} vs {COMPARAND_B} pros cons',     priority: 8,  maxResults: 8,  round: 2 },
  { source: 'web',  strategy: 'comparative_sweep', template: 'choose between {COMPARAND_A} and {COMPARAND_B}',priority: 7, maxResults: 6,  round: 2 },
];

/**
 * Technical + code goal overlay: skews toward code + web docs queries.
 */
const TECHNICAL_OVERLAY: QueryTemplate[] = [
  { source: 'code', strategy: 'code_archaeology',  template: '{TERM} implementation',              priority: 10, maxResults: 15, round: 1 },
  { source: 'web',  strategy: 'targeted_lookup',   template: '{TERM} API reference documentation', priority: 9,  maxResults: 8,  round: 1 },
  { source: 'web',  strategy: 'targeted_lookup',   template: '{TERM} source code explanation',     priority: 8,  maxResults: 8,  round: 2 },
];

export const QUERY_TEMPLATES: Record<
  ResearchDepth,
  Record<ResearchGoalType | 'default', QueryTemplate[]>
> = {
  surface: {
    default:       SURFACE_TEMPLATES,
    technical:     [...SURFACE_TEMPLATES, ...TECHNICAL_OVERLAY.filter(t => t.round === 1)],
    factual:       SURFACE_TEMPLATES,
    exploratory:   SURFACE_TEMPLATES,
    investigative: SURFACE_TEMPLATES,
    comparative:   [...SURFACE_TEMPLATES, ...COMPARATIVE_OVERLAY.filter(t => t.round === 1)],
    procedural:    SURFACE_TEMPLATES,
  },
  standard: {
    default:       STANDARD_TEMPLATES,
    technical:     [...STANDARD_TEMPLATES, ...TECHNICAL_OVERLAY],
    factual:       STANDARD_TEMPLATES,
    exploratory:   STANDARD_TEMPLATES,
    investigative: STANDARD_TEMPLATES,
    comparative:   [...STANDARD_TEMPLATES, ...COMPARATIVE_OVERLAY],
    procedural:    STANDARD_TEMPLATES,
  },
  deep: {
    default:       DEEP_TEMPLATES,
    technical:     [...DEEP_TEMPLATES, ...TECHNICAL_OVERLAY],
    factual:       DEEP_TEMPLATES,
    exploratory:   DEEP_TEMPLATES,
    investigative: DEEP_TEMPLATES,
    comparative:   [...DEEP_TEMPLATES, ...COMPARATIVE_OVERLAY],
    procedural:    DEEP_TEMPLATES,
  },
};
src/planning/GoalParser.ts
TypeScript

import type { ResearchGoalType } from '../types/research.js';
import type { ParsedGoal } from '../types/planning.js';

/**
 * GoalParser: pure-logic (no I/O) parser that extracts structure from a
 * free-text research goal.
 *
 * Design decisions:
 * - Pattern-based, zero LLM calls — fast and deterministic.
 * - LLM-assisted parsing lives in ResearchPlanner (Part 2), not here.
 * - Key term extraction uses a combination of:
 *     (a) code-style identifier detection (/[A-Z][a-zA-Z]+/ or camelCase)
 *     (b) quoted phrase detection
 *     (c) capitalised word runs (proper nouns)
 * - Goal type classification uses weighted keyword scoring.
 */

// ─── Goal type keyword weights ────────────────────────────────────────────────

const GOAL_TYPE_SIGNALS: Record<ResearchGoalType, string[]> = {
  technical:     ['implement', 'architecture', 'api', 'code', 'refactor',
                  'performance', 'debug', 'algorithm', 'design pattern', 'sdk'],
  factual:       ['what is', 'define', 'explain', 'meaning of', 'definition',
                  'describe', 'facts about', 'history of'],
  exploratory:   ['explore', 'survey', 'overview of', 'tell me about',
                  'investigate', 'learn about', 'introduction to'],
  investigative: ['why', 'root cause', 'reason', 'cause of', 'why does',
                  'how come', 'diagnose', 'trace', 'investigate why'],
  comparative:   ['vs', 'versus', 'compare', 'comparison', 'difference between',
                  'which is better', 'trade-off', 'pros and cons'],
  procedural:    ['how to', 'steps to', 'guide', 'tutorial', 'walk me through',
                  'process for', 'procedure', 'how do i'],
};

// ─── Constraint patterns ──────────────────────────────────────────────────────

const LANGUAGE_PATTERNS  = /\b(typescript|javascript|python|rust|go|java|c\+\+|ruby)\b/gi;
const VERSION_PATTERNS   = /\bv?\d+\.\d+(\.\d+)?\b/g;
const DATE_PATTERNS      = /\b(2\d{3}|since \d{4}|before \d{4}|in \d{4})\b/gi;
const FRAMEWORK_PATTERNS = /\b(react|nextjs|fastify|express|django|flask|actix|gin|spring)\b/gi;

// ─── Negation / exclusion patterns ───────────────────────────────────────────

const EXCLUSION_PATTERNS = [
  /\bnot\s+(using|including|with)\s+(.+?)(?:\.|,|$)/gi,
  /\bexclude\s+(.+?)(?:\.|,|$)/gi,
  /\bignore\s+(.+?)(?:\.|,|$)/gi,
  /\bwithout\s+(.+?)(?:\.|,|$)/gi,
];

// ─── Sub-question patterns ────────────────────────────────────────────────────

const SUBQUESTION_SEPARATORS = /[?;]|(?:\band\b|\balso\b)/gi;

export class GoalParser {
  /**
   * Parse a raw research goal string into structured metadata.
   */
  parse(rawGoal: string): ParsedGoal {
    const normalised = rawGoal.trim();
    const lower      = normalised.toLowerCase();

    return {
      rawGoal,
      goalType:     this.classifyGoalType(lower),
      keyTerms:     this.extractKeyTerms(normalised),
      constraints:  this.extractConstraints(normalised),
      exclusions:   this.extractExclusions(normalised),
      intent:       this.buildIntent(normalised, lower),
      comparands:   this.extractComparands(lower),
      subQuestions: this.extractSubQuestions(normalised),
    };
  }

  // ── Goal type classification ──────────────────────────────────────────────

  private classifyGoalType(lower: string): ResearchGoalType {
    const scores = new Map<ResearchGoalType, number>();

    for (const [type, signals] of Object.entries(GOAL_TYPE_SIGNALS) as
         [ResearchGoalType, string[]][]) {
      let score = 0;
      for (const signal of signals) {
        if (lower.includes(signal)) score++;
      }
      scores.set(type, score);
    }

    let best: ResearchGoalType = 'exploratory';
    let bestScore = 0;

    for (const [type, score] of scores) {
      if (score > bestScore) {
        bestScore = score;
        best = type;
      }
    }

    return best;
  }

  // ── Key term extraction ────────────────────────────────────────────────────

  private extractKeyTerms(text: string): string[] {
    const terms = new Set<string>();

    // 1. Quoted phrases
    const quoted = text.matchAll(/"([^"]+)"|'([^']+)'/g);
    for (const m of quoted) {
      terms.add((m[1] ?? m[2]!).trim());
    }

    // 2. PascalCase / camelCase identifiers
    const identifiers = text.matchAll(/\b([A-Z][a-zA-Z]{2,}(?:[A-Z][a-zA-Z]*)*)\b/g);
    for (const m of identifiers) {
      terms.add(m[1]!);
    }

    // 3. camelCase
    const camel = text.matchAll(/\b([a-z][a-z]+[A-Z][a-zA-Z]+)\b/g);
    for (const m of camel) {
      terms.add(m[1]!);
    }

    // 4. Capitalised runs (>= 2 capitalised words in sequence)
    const capRuns = text.matchAll(/(?:[A-Z][a-z]+\s){1,3}[A-Z][a-z]+/g);
    for (const m of capRuns) {
      terms.add(m[0]!.trim());
    }

    // 5. Snake/kebab identifiers
    const snakeKebab = text.matchAll(/\b([a-z][a-z0-9]*(?:[_-][a-z0-9]+){1,})\b/g);
    for (const m of snakeKebab) {
      terms.add(m[1]!);
    }

    // Remove stop words and very short terms
    const stopWords = new Set([
      'and', 'the', 'for', 'with', 'how', 'why', 'what',
      'use', 'can', 'are', 'its', 'it',
    ]);

    return Array.from(terms)
      .filter(t => t.length > 2 && !stopWords.has(t.toLowerCase()))
      .slice(0, 10);
  }

  // ── Constraint extraction ─────────────────────────────────────────────────

  private extractConstraints(text: string): string[] {
    const constraints: string[] = [];

    const langMatches = text.matchAll(LANGUAGE_PATTERNS);
    for (const m of langMatches) constraints.push(`language:${m[0].toLowerCase()}`);

    const versionMatches = text.matchAll(VERSION_PATTERNS);
    for (const m of versionMatches) constraints.push(`version:${m[0]}`);

    const dateMatches = text.matchAll(DATE_PATTERNS);
    for (const m of dateMatches) constraints.push(`date:${m[0]}`);

    const frameworkMatches = text.matchAll(FRAMEWORK_PATTERNS);
    for (const m of frameworkMatches) constraints.push(`framework:${m[0].toLowerCase()}`);

    return [...new Set(constraints)];
  }

  // ── Exclusion extraction ───────────────────────────────────────────────────

  private extractExclusions(text: string): string[] {
    const exclusions: string[] = [];

    for (const pattern of EXCLUSION_PATTERNS) {
      const matches = text.matchAll(pattern);
      for (const m of matches) {
        const term = (m[1] ?? m[2] ?? '').trim();
        if (term) exclusions.push(term);
      }
    }

    return exclusions;
  }

  // ── Comparand extraction ──────────────────────────────────────────────────

  private extractComparands(lower: string): string[] {
    // Pattern: "X vs Y", "X versus Y", "X compared to Y"
    const vsMatch = lower.match(
      /([a-z0-9\s_-]{2,30})\s+(?:vs\.?|versus|compared\s+to)\s+([a-z0-9\s_-]{2,30})/i,
    );

    if (vsMatch) {
      return [vsMatch[1]!.trim(), vsMatch[2]!.trim()];
    }

    return [];
  }

  // ── Sub-question extraction ────────────────────────────────────────────────

  private extractSubQuestions(text: string): string[] {
    const parts = text
      .split(SUBQUESTION_SEPARATORS)
      .map(p => p.trim())
      .filter(p => p.length > 10 && p.length < 200);

    // De-duplicate and skip trivially short fragments
    return [...new Set(parts)].slice(0, 6);
  }

  // ── Intent summary ────────────────────────────────────────────────────────

  private buildIntent(text: string, lower: string): string {
    // Truncate to first sentence if long
    const firstSentence = text.split(/[.!?]/)[0]?.trim() ?? text;
    return firstSentence.length > 120
      ? firstSentence.slice(0, 120) + '…'
      : firstSentence;
  }
}
src/planning/ResearchPlanner.ts
TypeScript

import { nanoid } from 'nanoid';
import type { ResearchDepth, ResearchGoalType } from '../types/research.js';
import type {
  ParsedGoal,
  QueryPlan,
  Query,
  PlanStats,
} from '../types/planning.js';
import { GoalParser } from './GoalParser.js';
import { QUERY_TEMPLATES, type QueryTemplate } from './QueryTemplates.js';
import { PlanGenerationError } from '../types/errors.js';
import type { ResearchStore } from '../store/ResearchStore.js';

/**
 * ResearchPlanner: transforms a research goal into a persisted QueryPlan.
 *
 * Design decisions:
 * - Pure generation (no I/O beyond ResearchStore writes).
 * - Interpolates GoalParser output into QUERY_TEMPLATES.
 * - Deduplicates query texts (exact match) within a plan.
 * - Round-1 queries are the "primary sweep"; rounds 2–3 use sub-questions
 *   and gap-fill slots populated later by QueryEngine (after round 1 results).
 * - estimatedTokens uses a fixed ~150 tokens/result heuristic.
 */

const TOKENS_PER_RESULT = 150;
const COST_PER_TOKEN_USD = 0.000003; // rough composite rate

const DEPTH_TO_ROUNDS: Record<ResearchDepth, number> = {
  surface:  1,
  standard: 2,
  deep:     3,
};

export class ResearchPlanner {
  private goalParser: GoalParser;

  constructor(private readonly store: ResearchStore) {
    this.goalParser = new GoalParser();
  }

  /**
   * Generate and persist a QueryPlan for a research.
   * Returns the plan + parse stats.
   */
  async plan(
    researchId: string,
    goal: string,
    depth: ResearchDepth,
  ): Promise<{ plan: QueryPlan; parsedGoal: ParsedGoal; stats: PlanStats }> {
    let parsedGoal: ParsedGoal;

    try {
      parsedGoal = this.goalParser.parse(goal);
    } catch (err) {
      throw new PlanGenerationError(researchId, err);
    }

    const goalType = parsedGoal.goalType as ResearchGoalType;
    const rounds   = DEPTH_TO_ROUNDS[depth];
    const now      = new Date().toISOString();
    const planId   = nanoid();

    // Pick template set
    const depthTemplates = QUERY_TEMPLATES[depth];
    const templates: QueryTemplate[] = [
      ...(depthTemplates[goalType]  ?? depthTemplates.default),
    ];

    // Build queries from templates
    const queries: Query[] = [];
    const seenTexts = new Set<string>();

    for (const tmpl of templates) {
      if (tmpl.round > rounds) continue;

      const interpolated = this.interpolate(tmpl.template, parsedGoal);

      // Deduplicate by text within this plan
      if (seenTexts.has(interpolated.toLowerCase())) continue;
      seenTexts.add(interpolated.toLowerCase());

      queries.push({
        id:          nanoid(),
        planId,
        round:       tmpl.round,
        source:      tmpl.source,
        strategy:    tmpl.strategy,
        text:        interpolated,
        priority:    tmpl.priority,
        maxResults:  tmpl.maxResults,
        executed:    false,
        resultCount: 0,
      });
    }

    // Ensure at least one query per requested round
    for (let r = 1; r <= rounds; r++) {
      const hasRound = queries.some(q => q.round === r);
      if (!hasRound) {
        queries.push({
          id:          nanoid(),
          planId,
          round:       r,
          source:      'web',
          strategy:    r === 1 ? 'broad_survey' : 'gap_fill',
          text:        r === 1
            ? `${parsedGoal.keyTerms[0] ?? goal} overview`
            : `${parsedGoal.keyTerms[0] ?? goal} details`,
          priority:    5,
          maxResults:  8,
          executed:    false,
          resultCount: 0,
        });
      }
    }

    // Persist plan (queries separately)
    const plan: QueryPlan = {
      id:         planId,
      researchId,
      goalType:   parsedGoal.goalType,
      depth,
      rounds,
      queries,
      gaps:       [],
      createdAt:  now,
      updatedAt:  now,
    };

    this.store.savePlan({
      id:         plan.id,
      researchId: plan.researchId,
      goalType:   plan.goalType,
      depth:      plan.depth,
      rounds:     plan.rounds,
      gaps:       plan.gaps,
      createdAt:  plan.createdAt,
      updatedAt:  plan.updatedAt,
    });

    for (const q of queries) {
      this.store.saveQuery(q);
    }

    const stats = this.buildStats(queries);

    return { plan, parsedGoal, stats };
  }

  /**
   * After a query round completes, regenerate round N+1 queries using
   * identified knowledge gaps. Gaps are extracted by QueryEngine and passed
   * back here.
   */
  async refineWithGaps(
    planId: string,
    researchId: string,
    nextRound: number,
    gaps: string[],
  ): Promise<Query[]> {
    if (gaps.length === 0) return [];

    const now = new Date().toISOString();
    const newQueries: Query[] = [];

    for (const gap of gaps.slice(0, 6)) {
      // One web + one code query per gap
      for (const source of ['web', 'code'] as const) {
        const q: Query = {
          id:          nanoid(),
          planId,
          round:       nextRound,
          source,
          strategy:    'gap_fill',
          text:        gap,
          priority:    8,
          maxResults:  6,
          executed:    false,
          resultCount: 0,
        };
        this.store.saveQuery(q);
        newQueries.push(q);
      }
    }

    // Update plan's gaps and updatedAt
    this.store.savePlan({
      id:         planId,
      researchId,
      goalType:   '',  // no-op: ON CONFLICT only updates gaps+updated_at
      depth:      'standard',
      rounds:     nextRound,
      gaps,
      createdAt:  now,
      updatedAt:  now,
    });

    return newQueries;
  }

  // ── Interpolation ──────────────────────────────────────────────────────────

  private interpolate(template: string, goal: ParsedGoal): string {
    const primaryTerm   = goal.keyTerms[0] ?? goal.intent;
    const comparandA    = goal.comparands[0] ?? goal.keyTerms[0] ?? goal.intent;
    const comparandB    = goal.comparands[1] ?? goal.keyTerms[1] ?? 'alternatives';
    const subQ          = goal.subQuestions[0] ?? goal.intent;

    return template
      .replace(/\{TERM\}/g, primaryTerm)
      .replace(/\{COMPARAND_A\}/g, comparandA)
      .replace(/\{COMPARAND_B\}/g, comparandB)
      .replace(/\{SUBQ\}/g, subQ);
  }

  // ── Stats ─────────────────────────────────────────────────────────────────

  private buildStats(queries: Query[]): PlanStats {
    const queriesBySource: Record<string, number> = {};
    const queriesByRound:  Record<number, number> = {};
    let totalMaxResults = 0;

    for (const q of queries) {
      queriesBySource[q.source] = (queriesBySource[q.source] ?? 0) + 1;
      queriesByRound[q.round]   = (queriesByRound[q.round]   ?? 0) + 1;
      totalMaxResults += q.maxResults;
    }

    const estimatedTokens  = totalMaxResults * TOKENS_PER_RESULT;
    const estimatedCostUsd = estimatedTokens * COST_PER_TOKEN_USD;

    return {
      totalQueries:    queries.length,
      queriesBySource,
      queriesByRound,
      estimatedTokens,
      estimatedCostUsd,
    };
  }
}
src/query/WebSearchAdapter.ts
TypeScript

import { WebSearchProviderError } from '../types/errors.js';

/**
 * WebSearchAdapter: provider-agnostic web search.
 *
 * Design decisions:
 * - Supports Brave Search API (primary) and SerpAPI (fallback).
 * - Configurable via environment variables (consistent with Pass 1's .env).
 * - Returns raw SearchResult records; SourceFetcher handles content fetching.
 * - Rate-limited internally with a simple token bucket (no external dep).
 * - Results are de-ranked for common content farms and SEO spam sites.
 */

export interface WebSearchResult {
  title:    string;
  url:      string;
  snippet:  string;
  position: number;
}

export interface WebSearchConfig {
  provider:      'brave' | 'serpapi' | 'none';
  braveApiKey?:  string;
  serpApiKey?:   string;
  maxRetries?:   number;
  timeoutMs?:    number;
}

// Domains to downrank (content farms, SEO spam)
const DOWNRANKED_DOMAINS = new Set([
  'w3schools.com', 'geeksforgeeks.org', 'javatpoint.com',
  'tutorialspoint.com', 'medium.com/tag', 'quora.com',
]);

export class WebSearchAdapter {
  private lastCallAt   = 0;
  private minIntervalMs: number;
  private config: Required<WebSearchConfig>;

  constructor(config: WebSearchConfig) {
    this.config = {
      provider:     config.provider,
      braveApiKey:  config.braveApiKey  ?? process.env.BRAVE_SEARCH_API_KEY ?? '',
      serpApiKey:   config.serpApiKey   ?? process.env.SERP_API_KEY ?? '',
      maxRetries:   config.maxRetries   ?? 3,
      timeoutMs:    config.timeoutMs    ?? 10_000,
    };

    // Brave: 1 req/sec; SerpAPI: 1 req/sec; none: no throttle
    this.minIntervalMs = config.provider === 'none' ? 0 : 1_000;
  }

  /**
   * Execute a web search and return up to maxResults results.
   */
  async search(
    query: string,
    maxResults: number = 10,
  ): Promise<WebSearchResult[]> {
    if (this.config.provider === 'none') {
      // Offline / test mode: return empty results
      return [];
    }

    await this.rateLimit();

    for (let attempt = 0; attempt < this.config.maxRetries; attempt++) {
      try {
        const results = this.config.provider === 'brave'
          ? await this.searchBrave(query, maxResults)
          : await this.searchSerpApi(query, maxResults);

        return this.rerank(results);
      } catch (err) {
        if (attempt === this.config.maxRetries - 1) {
          throw new WebSearchProviderError(this.config.provider, err);
        }
        // Exponential backoff
        await this.sleep(500 * Math.pow(2, attempt));
      }
    }

    return [];
  }

  // ── Brave Search API ──────────────────────────────────────────────────────

  private async searchBrave(
    query: string,
    count: number,
  ): Promise<WebSearchResult[]> {
    if (!this.config.braveApiKey) {
      throw new WebSearchProviderError('brave', 'BRAVE_SEARCH_API_KEY not set');
    }

    const url = new URL('https://api.search.brave.com/res/v1/web/search');
    url.searchParams.set('q', query);
    url.searchParams.set('count', String(Math.min(count, 20)));
    url.searchParams.set('text_decorations', 'false');
    url.searchParams.set('result_filter', 'web');

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), this.config.timeoutMs);

    try {
      const res = await fetch(url.toString(), {
        headers: {
          'Accept':              'application/json',
          'Accept-Encoding':     'gzip',
          'X-Subscription-Token': this.config.braveApiKey,
        },
        signal: controller.signal,
      });

      if (!res.ok) {
        throw new WebSearchProviderError('brave', `HTTP ${res.status}: ${res.statusText}`);
      }

      const data = await res.json() as {
        web?: { results?: Array<{ title: string; url: string; description: string }> };
      };

      return (data.web?.results ?? []).map((r, i) => ({
        title:    r.title,
        url:      r.url,
        snippet:  r.description,
        position: i + 1,
      }));
    } finally {
      clearTimeout(timeout);
    }
  }

  // ── SerpAPI ───────────────────────────────────────────────────────────────

  private async searchSerpApi(
    query: string,
    count: number,
  ): Promise<WebSearchResult[]> {
    if (!this.config.serpApiKey) {
      throw new WebSearchProviderError('serpapi', 'SERP_API_KEY not set');
    }

    const url = new URL('https://serpapi.com/search');
    url.searchParams.set('q', query);
    url.searchParams.set('num', String(Math.min(count, 10)));
    url.searchParams.set('api_key', this.config.serpApiKey);
    url.searchParams.set('engine', 'google');

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), this.config.timeoutMs);

    try {
      const res = await fetch(url.toString(), { signal: controller.signal });

      if (!res.ok) {
        throw new WebSearchProviderError('serpapi', `HTTP ${res.status}`);
      }

      const data = await res.json() as {
        organic_results?: Array<{ title: string; link: string; snippet: string }>;
      };

      return (data.organic_results ?? []).map((r, i) => ({
        title:    r.title,
        url:      r.link,
        snippet:  r.snippet,
        position: i + 1,
      }));
    } finally {
      clearTimeout(timeout);
    }
  }

  // ── Reranking ─────────────────────────────────────────────────────────────

  private rerank(results: WebSearchResult[]): WebSearchResult[] {
    return results
      .map(r => {
        try {
          const hostname = new URL(r.url).hostname.replace('www.', '');
          const penalty  = DOWNRANKED_DOMAINS.has(hostname) ? 5 : 0;
          return { result: r, adjustedPos: r.position + penalty };
        } catch {
          return { result: r, adjustedPos: r.position };
        }
      })
      .sort((a, b) => a.adjustedPos - b.adjustedPos)
      .map(({ result }) => result);
  }

  // ── Helpers ───────────────────────────────────────────────────────────────

  private async rateLimit(): Promise<void> {
    const now     = Date.now();
    const elapsed = now - this.lastCallAt;
    if (elapsed < this.minIntervalMs) {
      await this.sleep(this.minIntervalMs - elapsed);
    }
    this.lastCallAt = Date.now();
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
src/query/CodeSearchAdapter.ts
TypeScript

import type { GraphNodeResponse } from '@locoworker/graphify';

/**
 * CodeSearchAdapter: translates natural-language search queries into
 * graphify node/edge lookups and returns structured results.
 *
 * Design decisions:
 * - Uses graphify's GraphQuerier interface (defined in Pass 4).
 * - Heuristic name extraction: CamelCase, snake_case, quoted terms.
 * - Falls back to a substring scan across node names if no identifier found.
 * - Depth is capped at 1 to keep result sets small (caller can expand).
 */

export interface CodeSearchResult {
  nodeId:    string;
  name:      string;
  type:      string;
  path:      string;
  line:      number;
  docstring: string;
  snippet:   string;    // short formatted summary
  edges?:    Array<{ targetId: string; type: string; weight: number }>;
}

export class CodeSearchAdapter {
  constructor(
    private readonly graphQuerier: {
      queryNodes: (opts: {
        name?: string;
        nodeType?: string;
        path?: string;
        includeEdges?: boolean;
        depth?: number;
      }) => Promise<GraphNodeResponse[]>;
    },
  ) {}

  /**
   * Search code graph for symbols matching the query.
   */
  async search(
    query: string,
    maxResults: number = 10,
  ): Promise<CodeSearchResult[]> {
    const identifiers = this.extractIdentifiers(query);

    if (identifiers.length === 0) {
      return [];
    }

    const results: CodeSearchResult[] = [];
    const seen = new Set<string>();

    for (const name of identifiers) {
      if (results.length >= maxResults) break;

      const nodes = await this.graphQuerier.queryNodes({
        name,
        includeEdges: true,
        depth: 1,
      });

      for (const node of nodes) {
        if (seen.has(node.id)) continue;
        seen.add(node.id);

        results.push({
          nodeId:    node.id,
          name:      node.name,
          type:      node.type,
          path:      node.path,
          line:      node.line,
          docstring: node.docstring ?? '',
          snippet:   this.formatSnippet(node),
          edges:     node.edges,
        });
      }
    }

    return results.slice(0, maxResults);
  }

  // ── Identifier extraction ─────────────────────────────────────────────────

  private extractIdentifiers(query: string): string[] {
    const ids: string[] = [];

    // Quoted terms
    for (const m of query.matchAll(/"([^"]+)"|`([^`]+)`/g)) {
      ids.push((m[1] ?? m[2])!);
    }

    // CamelCase
    for (const m of query.matchAll(/\b([A-Z][a-zA-Z]{2,})\b/g)) {
      ids.push(m[1]!);
    }

    // snake_case
    for (const m of query.matchAll(/\b([a-z][a-z0-9]*(?:_[a-z0-9]+)+)\b/g)) {
      ids.push(m[1]!);
    }

    // camelCase
    for (const m of query.matchAll(/\b([a-z][a-z0-9]+[A-Z][a-zA-Z0-9]+)\b/g)) {
      ids.push(m[1]!);
    }

    // Deduplicate while preserving order
    return [...new Set(ids)];
  }

  private formatSnippet(node: GraphNodeResponse): string {
    const edgeSummary = node.edges?.length
      ? ` — ${node.edges.length} edges`
      : '';
    return `[${node.type}] ${node.name} at ${node.path}:${node.line}${edgeSummary}`;
  }
}
src/query/WikiSearchAdapter.ts
TypeScript

/**
 * WikiSearchAdapter: wraps WikiSearch and WikiLinker from packages/wiki.
 * Returns structured wiki search results for the QueryEngine.
 */

export interface WikiSearchAdapterResult {
  slug:     string;
  title:    string;
  excerpt:  string;
  tags:     string[];
  status:   string;
  backlinks: string[];
  score:    number;  // relevance from FTS rank
}

export class WikiSearchAdapter {
  constructor(
    private readonly wikiSearch: {
      search: (opts: {
        query?: string;
        tags?: string[];
        status?: string;
        limit?: number;
      }) => Promise<Array<{
        slug: string;
        title: string;
        content: string;
        tags: string[];
        status: string;
      }>>;
    },
    private readonly wikiLinker: {
      getBacklinks: (slug: string) => Promise<string[]>;
    },
  ) {}

  async search(
    query: string,
    maxResults: number = 5,
  ): Promise<WikiSearchAdapterResult[]> {
    const pages = await this.wikiSearch.search({
      query,
      status: 'active',
      limit:  maxResults,
    });

    const results: WikiSearchAdapterResult[] = [];

    for (const page of pages) {
      const backlinks = await this.wikiLinker.getBacklinks(page.slug);
      const excerpt   = this.extractExcerpt(page.content, query, 300);

      results.push({
        slug:      page.slug,
        title:     page.title,
        excerpt,
        tags:      page.tags,
        status:    page.status,
        backlinks,
        // Score heuristic: more backlinks = more central
        score:     Math.min(1, 0.5 + backlinks.length * 0.05),
      });
    }

    return results;
  }

  /**
   * Extract a query-centred excerpt from page content.
   * Returns the first 300 chars around the first occurrence of a key term.
   */
  private extractExcerpt(
    content: string,
    query: string,
    length: number,
  ): string {
    const terms   = query.toLowerCase().split(/\s+/);
    const lower   = content.toLowerCase();
    let bestPos   = 0;
    let bestScore = 0;

    for (let i = 0; i < content.length - length; i += 50) {
      const window = lower.slice(i, i + length);
      const score  = terms.filter(t => window.includes(t)).length;
      if (score > bestScore) {
        bestScore = score;
        bestPos   = i;
      }
    }

    return content.slice(bestPos, bestPos + length).replace(/\n+/g, ' ').trim();
  }
}
src/query/MemorySearchAdapter.ts
TypeScript

/**
 * MemorySearchAdapter: queries packages/memory's MemoryManager for
 * entries relevant to the current research goal.
 */

export interface MemorySearchResult {
  entryId:    string;
  content:    string;
  tags:       string[];
  importance: string;
  timestamp:  string;
  score:      number;
}

export class MemorySearchAdapter {
  constructor(
    private readonly memoryManager: {
      query: (opts: {
        query?: string;
        tags?: string[];
        importance?: string;
        limit?: number;
      }) => Promise<Array<{
        id: string;
        content: string;
        tags: string[];
        importance: string;
        timestamp: string;
      }>>;
    },
  ) {}

  async search(
    query: string,
    maxResults: number = 10,
  ): Promise<MemorySearchResult[]> {
    const entries = await this.memoryManager.query({
      query,
      limit: maxResults,
    });

    return entries.map(e => ({
      entryId:    e.id,
      content:    e.content,
      tags:       e.tags,
      importance: e.importance,
      timestamp:  e.timestamp,
      score:      this.scoreByImportance(e.importance),
    }));
  }

  private scoreByImportance(importance: string): number {
    const scores: Record<string, number> = {
      critical: 1.0,
      high:     0.8,
      medium:   0.6,
      low:      0.4,
    };
    return scores[importance] ?? 0.5;
  }
}
src/query/QueryEngine.ts
TypeScript

import pLimit from 'p-limit';
import { nanoid } from 'nanoid';
import type { Query } from '../types/planning.js';
import type { Source } from '../types/evidence.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import { QueryExecutionError } from '../types/errors.js';
import type { WebSearchAdapter } from './WebSearchAdapter.js';
import type { CodeSearchAdapter } from './CodeSearchAdapter.js';
import type { WikiSearchAdapter } from './WikiSearchAdapter.js';
import type { MemorySearchAdapter } from './MemorySearchAdapter.js';
import { SourceFetcher } from '../evidence/SourceFetcher.js';

/**
 * QueryEngine: dispatches a set of Query objects across all registered
 * source adapters and returns Source records ready for EvidenceCollector.
 *
 * Design decisions:
 * - Concurrency-limited per source type to avoid hammering external services.
 *   Web: max 3 concurrent; Code/Wiki/Memory: max 5 concurrent.
 * - Web search results are fetched by SourceFetcher (HTML → Markdown).
 * - Code/Wiki/Memory results are converted inline (no HTTP fetch needed).
 * - Gaps are identified as sub-questions that yielded zero results across
 *   all sources (returned for ResearchPlanner.refineWithGaps).
 * - All errors are caught per-query; one bad query doesn't abort the round.
 */

export interface QueryRoundResult {
  sources: Source[];
  gaps:    string[];   // queries that returned 0 results → feed next round
  stats: {
    total:     number;
    succeeded: number;
    failed:    number;
    skipped:   number;  // duplicate fingerprints
  };
}

const WEB_CONCURRENCY    = 3;
const LOCAL_CONCURRENCY  = 5;

export class QueryEngine {
  private webLimit:   ReturnType<typeof pLimit>;
  private localLimit: ReturnType<typeof pLimit>;

  constructor(
    private readonly store:   ResearchStore,
    private readonly fetcher: SourceFetcher,
    private readonly web:     WebSearchAdapter,
    private readonly code:    CodeSearchAdapter,
    private readonly wiki:    WikiSearchAdapter,
    private readonly memory:  MemorySearchAdapter,
  ) {
    this.webLimit   = pLimit(WEB_CONCURRENCY);
    this.localLimit = pLimit(LOCAL_CONCURRENCY);
  }

  /**
   * Execute all pending queries for a given round and return sources + gaps.
   */
  async executeRound(
    researchId: string,
    round: number,
  ): Promise<QueryRoundResult> {
    const queries = this.store.getPendingQueriesForRound(researchId, round);

    const stats = { total: queries.length, succeeded: 0, failed: 0, skipped: 0 };
    const allSources: Source[] = [];
    const gaps: string[] = [];

    // Fan out queries in parallel (respecting concurrency limits per source)
    const results = await Promise.allSettled(
      queries.map(q => this.executeQuery(researchId, q)),
    );

    for (let i = 0; i < results.length; i++) {
      const result = results[i]!;
      const query  = queries[i]!;

      if (result.status === 'fulfilled') {
        const { sources, skipped } = result.value;
        stats.succeeded++;
        stats.skipped  += skipped;

        if (sources.length === 0) {
          // No results → this query text becomes a gap for next round
          gaps.push(query.text);
        }

        allSources.push(...sources);
        this.store.markQueryExecuted(query.id, sources.length);
      } else {
        stats.failed++;
        console.error(
          `[QueryEngine] Query failed: "${query.text}" (${query.source})`,
          result.reason,
        );
        // Still mark as executed to avoid infinite retry
        this.store.markQueryExecuted(query.id, 0);
      }
    }

    return { sources: allSources, gaps, stats };
  }

  // ── Per-query dispatch ────────────────────────────────────────────────────

  private executeQuery(
    researchId: string,
    query: Query,
  ): Promise<{ sources: Source[]; skipped: number }> {
    switch (query.source) {
      case 'web':    return this.webLimit(() => this.executeWebQuery(researchId, query));
      case 'code':   return this.localLimit(() => this.executeCodeQuery(researchId, query));
      case 'wiki':   return this.localLimit(() => this.executeWikiQuery(researchId, query));
      case 'memory': return this.localLimit(() => this.executeMemoryQuery(researchId, query));
      default:
        return Promise.resolve({ sources: [], skipped: 0 });
    }
  }

  // ── Web queries ───────────────────────────────────────────────────────────

  private async executeWebQuery(
    researchId: string,
    query: Query,
  ): Promise<{ sources: Source[]; skipped: number }> {
    const searchResults = await this.web.search(query.text, query.maxResults);
    const sources: Source[] = [];
    let skipped = 0;

    for (const sr of searchResults) {
      try {
        const source = await this.fetcher.fetch(researchId, {
          type:  'web',
          url:   sr.url,
          title: sr.title,
          hint:  sr.snippet,
        });

        if (source) {
          sources.push(source);
        } else {
          skipped++;
        }
      } catch (err) {
        // Log but continue — one bad URL shouldn't abort the whole query
        console.warn(`[QueryEngine] Fetch failed for ${sr.url}:`, err);
        skipped++;
      }
    }

    return { sources, skipped };
  }

  // ── Code queries ──────────────────────────────────────────────────────────

  private async executeCodeQuery(
    researchId: string,
    query: Query,
  ): Promise<{ sources: Source[]; skipped: number }> {
    const codeResults = await this.code.search(query.text, query.maxResults);
    const sources: Source[] = [];
    let skipped = 0;

    for (const cr of codeResults) {
      const content = this.formatCodeResult(cr);
      const source  = this.fetcher.buildLocalSource(researchId, {
        type:      'code',
        nodeId:    cr.nodeId,
        title:     `${cr.type}: ${cr.name}`,
        content,
        score:     0.7,
        metadata:  { path: cr.path, line: cr.line, edges: cr.edges },
      });

      const isDup = this.store.fingerprintExists(researchId, source.fingerprint);
      if (isDup) {
        skipped++;
        continue;
      }

      this.store.saveSource(source);
      this.store.incrementSourceCount(researchId);
      sources.push(source);
    }

    return { sources, skipped };
  }

  // ── Wiki queries ──────────────────────────────────────────────────────────

  private async executeWikiQuery(
    researchId: string,
    query: Query,
  ): Promise<{ sources: Source[]; skipped: number }> {
    const wikiResults = await this.wiki.search(query.text, query.maxResults);
    const sources: Source[] = [];
    let skipped = 0;

    for (const wr of wikiResults) {
      const source = this.fetcher.buildLocalSource(researchId, {
        type:     'wiki',
        wikiSlug: wr.slug,
        title:    wr.title,
        content:  wr.excerpt,
        score:    wr.score,
        metadata: { tags: wr.tags, backlinks: wr.backlinks },
      });

      const isDup = this.store.fingerprintExists(researchId, source.fingerprint);
      if (isDup) {
        skipped++;
        continue;
      }

      this.store.saveSource(source);
      this.store.incrementSourceCount(researchId);
      sources.push(source);
    }

    return { sources, skipped };
  }

  // ── Memory queries ────────────────────────────────────────────────────────

  private async executeMemoryQuery(
    researchId: string,
    query: Query,
  ): Promise<{ sources: Source[]; skipped: number }> {
    const memResults = await this.memory.search(query.text, query.maxResults);
    const sources: Source[] = [];
    let skipped = 0;

    for (const mr of memResults) {
      const source = this.fetcher.buildLocalSource(researchId, {
        type:          'memory',
        memoryEntryId: mr.entryId,
        title:         `Memory: ${mr.content.slice(0, 50)}…`,
        content:       mr.content,
        score:         mr.score,
        metadata:      { tags: mr.tags, importance: mr.importance, timestamp: mr.timestamp },
      });

      const isDup = this.store.fingerprintExists(researchId, source.fingerprint);
      if (isDup) {
        skipped++;
        continue;
      }

      this.store.saveSource(source);
      this.store.incrementSourceCount(researchId);
      sources.push(source);
    }

    return { sources, skipped };
  }

  // ── Helpers ───────────────────────────────────────────────────────────────

  private formatCodeResult(cr: {
    name:      string;
    type:      string;
    path:      string;
    line:      number;
    docstring: string;
    snippet:   string;
    edges?:    Array<{ targetId: string; type: string; weight: number }>;
  }): string {
    const lines = [
      `**${cr.type}: ${cr.name}**`,
      `*${cr.path}:${cr.line}*`,
    ];

    if (cr.docstring) lines.push(`\n${cr.docstring}`);

    if (cr.edges?.length) {
      lines.push(
        `\nEdges (${cr.edges.length}): ` +
        cr.edges.slice(0, 5).map(e => `${e.type}→${e.targetId}`).join(', '),
      );
    }

    return lines.join('\n');
  }
}
src/evidence/ContentExtractor.ts
TypeScript

import * as cheerio from 'cheerio';
import TurndownService from 'turndown';
import { ContentExtractionError } from '../types/errors.js';

/**
 * ContentExtractor: converts raw HTML into clean, token-efficient Markdown.
 *
 * Design decisions:
 * - Uses Cheerio for DOM manipulation (removes noise before Turndown sees it).
 * - TurndownService for HTML → Markdown conversion.
 * - Noise removal: navs, footers, ads, cookie banners, social widgets.
 * - Title extraction: tries <h1>, then <title>, then URL-derived.
 * - Token estimation: char / 4 heuristic (consistent with AdaptiveCompactor).
 * - Hard truncation at MAX_CLEAN_CHARS to prevent runaway token usage.
 */

const MAX_CLEAN_CHARS = 12_000;  // ~3000 tokens

const NOISE_SELECTORS = [
  'nav', 'footer', 'header', 'aside',
  '.ad', '.ads', '.advert', '.advertisement',
  '.cookie', '.cookie-banner', '.cookie-notice',
  '.social', '.social-share', '.share-buttons',
  '.sidebar', '.widget', '.popup', '.modal',
  '.menu', '.navigation',
  'script', 'style', 'noscript',
  '[role="banner"]', '[role="navigation"]', '[role="complementary"]',
  '[aria-hidden="true"]',
];

export interface ExtractedContent {
  title:     string;
  markdown:  string;
  charCount: number;
  tokenEstimate: number;
}

export class ContentExtractor {
  private turndown: TurndownService;

  constructor() {
    this.turndown = new TurndownService({
      headingStyle:   'atx',
      bulletListMarker: '-',
      codeBlockStyle: 'fenced',
    });

    // Keep code blocks intact
    this.turndown.addRule('pre', {
      filter: ['pre'],
      replacement: (content, node) => {
        const lang = (node as Element).querySelector?.('code')
          ?.className?.replace('language-', '') ?? '';
        return `\n\`\`\`${lang}\n${content.trim()}\n\`\`\`\n`;
      },
    });
  }

  /**
   * Extract clean Markdown from raw HTML.
   */
  extract(html: string, sourceUrl?: string): ExtractedContent {
    let $: cheerio.CheerioAPI;

    try {
      $ = cheerio.load(html);
    } catch (err) {
      throw new ContentExtractionError(sourceUrl ?? 'unknown', err);
    }

    // 1. Remove noise elements
    for (const selector of NOISE_SELECTORS) {
      $(selector).remove();
    }

    // 2. Extract title
    const title =
      $('h1').first().text().trim() ||
      $('title').text().trim()       ||
      this.titleFromUrl(sourceUrl);

    // 3. Get main content area
    const mainContent =
      $('main').html() ??
      $('article').html() ??
      $('[role="main"]').html() ??
      $('body').html() ??
      '';

    // 4. Convert to Markdown
    let markdown: string;

    try {
      markdown = this.turndown.turndown(mainContent);
    } catch (err) {
      // Fallback: strip all tags
      markdown = mainContent.replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim();
    }

    // 5. Clean up whitespace + truncate
    markdown = markdown
      .replace(/\n{3,}/g, '\n\n')   // collapse triple+ newlines
      .replace(/\t/g, '  ')          // tabs → spaces
      .trim()
      .slice(0, MAX_CLEAN_CHARS);

    const charCount     = markdown.length;
    const tokenEstimate = Math.ceil(charCount / 4);

    return { title, markdown, charCount, tokenEstimate };
  }

  private titleFromUrl(url?: string): string {
    if (!url) return 'Untitled';
    try {
      const pathname = new URL(url).pathname;
      return pathname
        .split('/')
        .filter(Boolean)
        .pop()
        ?.replace(/[-_]/g, ' ')
        ?.replace(/\.\w+$/, '')
        ?? 'Untitled';
    } catch {
      return 'Untitled';
    }
  }
}
src/evidence/SourceFetcher.ts
TypeScript

import { createHash } from 'node:crypto';
import { nanoid } from 'nanoid';
import pRetry from 'p-retry';
import type { Source, SourceType } from '../types/evidence.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import { ContentExtractor } from './ContentExtractor.js';
import { SourceFetchError } from '../types/errors.js';

/**
 * SourceFetcher: fetches URLs, extracts content, fingerprints it, and
 * returns a persisted Source record.
 *
 * Design decisions:
 * - Deduplication by SHA-256 fingerprint of clean content (not URL).
 *   Same content served at different URLs is deduplicated correctly.
 * - Per-domain rate limiting (1 req/sec) via simple per-domain timestamp map.
 * - User-Agent rotation to reduce bot detection.
 * - fetchedAt is recorded for freshness tracking in the report.
 * - buildLocalSource() creates in-process Source records for
 *   code/wiki/memory adapters (no HTTP fetch needed).
 */

const MAX_RESPONSE_BYTES = 2 * 1024 * 1024; // 2 MB
const FETCH_TIMEOUT_MS   = 15_000;
const MIN_FETCH_INTERVAL = 1_000; // per-domain rate limit (ms)

const USER_AGENTS = [
  'Mozilla/5.0 (compatible; LocoWorker/0.1; +https://locoworker.dev/bot)',
  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
  'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36',
];

/**
 * Input to buildLocalSource (no HTTP, in-process source types).
 */
export interface LocalSourceInput {
  type:           SourceType;
  nodeId?:        string;
  wikiSlug?:      string;
  memoryEntryId?: string;
  filePath?:      string;
  title:          string;
  content:        string;
  score:          number;
  metadata?:      Record<string, unknown>;
}

export class SourceFetcher {
  private extractor:      ContentExtractor;
  private domainLastFetch = new Map<string, number>();
  private uaIndex         = 0;

  constructor(private readonly store: ResearchStore) {
    this.extractor = new ContentExtractor();
  }

  /**
   * Fetch a URL, extract content, fingerprint, deduplicate, and persist.
   * Returns null if the content is a duplicate.
   */
  async fetch(
    researchId: string,
    opts: {
      type:   'web';
      url:    string;
      title:  string;
      hint?:  string;
    },
  ): Promise<Source | null> {
    await this.rateLimit(opts.url);

    const response = await pRetry(
      () => this.fetchWithTimeout(opts.url),
      {
        retries:         3,
        minTimeout:      500,
        maxTimeout:      3_000,
        shouldRetry:     (err) => {
          // Retry on network errors and 5xx, not on 4xx
          const status = (err as any).status;
          return !status || status >= 500;
        },
      },
    );

    const { html, httpStatus } = response;

    // Extract clean content
    const extracted = this.extractor.extract(html, opts.url);

    // Compute fingerprint
    const fingerprint = this.fingerprint(extracted.markdown);

    // Deduplicate
    if (this.store.fingerprintExists(researchId, fingerprint)) {
      return null;
    }

    const source: Source = {
      id:             nanoid(),
      researchId,
      type:           'web',
      url:            opts.url,
      title:          extracted.title || opts.title,
      rawContent:     html.slice(0, 50_000),   // store first 50k chars of raw HTML
      cleanContent:   extracted.markdown,
      fingerprint,
      tokenCount:     extracted.tokenEstimate,
      relevanceScore: this.estimateRelevance(extracted, opts.hint),
      fetchedAt:      new Date().toISOString(),
      httpStatus,
    };

    this.store.saveSource(source);
    this.store.incrementSourceCount(researchId);

    return source;
  }

  /**
   * Build a Source record for in-process content (code/wiki/memory/file).
   * Does NOT persist — caller is responsible for calling store.saveSource().
   */
  buildLocalSource(
    researchId: string,
    input: LocalSourceInput,
  ): Source {
    const fingerprint = this.fingerprint(input.content);

    return {
      id:             nanoid(),
      researchId,
      type:           input.type,
      nodeId:         input.nodeId,
      wikiSlug:       input.wikiSlug,
      memoryEntryId:  input.memoryEntryId,
      filePath:       input.filePath,
      title:          input.title,
      rawContent:     input.content,
      cleanContent:   input.content,
      fingerprint,
      tokenCount:     Math.ceil(input.content.length / 4),
      relevanceScore: input.score,
      fetchedAt:      new Date().toISOString(),
      metadata:       input.metadata,
    };
  }

  // ── HTTP fetch ────────────────────────────────────────────────────────────

  private async fetchWithTimeout(
    url: string,
  ): Promise<{ html: string; httpStatus: number }> {
    const controller = new AbortController();
    const timeout    = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);
    const ua         = USER_AGENTS[this.uaIndex % USER_AGENTS.length]!;
    this.uaIndex++;

    try {
      const res = await fetch(url, {
        headers: {
          'User-Agent':      ua,
          'Accept':          'text/html,application/xhtml+xml,*/*;q=0.8',
          'Accept-Language': 'en-US,en;q=0.9',
          'Accept-Encoding': 'gzip, deflate, br',
        },
        signal: controller.signal,
        redirect: 'follow',
      });

      if (!res.ok) {
        const err = new SourceFetchError(url, res.status);
        (err as any).status = res.status;
        throw err;
      }

      // Guard against huge responses
      const contentLength = Number(res.headers.get('content-length') ?? 0);
      if (contentLength > MAX_RESPONSE_BYTES) {
        throw new SourceFetchError(url, res.status, `Response too large: ${contentLength}`);
      }

      const html = await res.text();
      return { html, httpStatus: res.status };

    } finally {
      clearTimeout(timeout);
    }
  }

  // ── Domain rate limiting ──────────────────────────────────────────────────

  private async rateLimit(url: string): Promise<void> {
    let domain: string;

    try {
      domain = new URL(url).hostname;
    } catch {
      return;
    }

    const lastFetch = this.domainLastFetch.get(domain) ?? 0;
    const elapsed   = Date.now() - lastFetch;

    if (elapsed < MIN_FETCH_INTERVAL) {
      await new Promise(resolve => setTimeout(resolve, MIN_FETCH_INTERVAL - elapsed));
    }

    this.domainLastFetch.set(domain, Date.now());
  }

  // ── Fingerprinting ────────────────────────────────────────────────────────

  fingerprint(content: string): string {
    // Normalise before fingerprinting (collapse whitespace, lowercase)
    const normalised = content
      .toLowerCase()
      .replace(/\s+/g, ' ')
      .trim();

    return createHash('sha256').update(normalised).digest('hex');
  }

  // ── Relevance heuristic ───────────────────────────────────────────────────

  private estimateRelevance(
    extracted: { title: string; markdown: string },
    hint?: string,
  ): number {
    if (!hint) return 0.5;

    const hintTerms  = hint.toLowerCase().split(/\s+/);
    const fullText   = (extracted.title + ' ' + extracted.markdown).toLowerCase();
    const matchCount = hintTerms.filter(t => fullText.includes(t)).length;

    return Math.min(1, 0.4 + (matchCount / Math.max(hintTerms.length, 1)) * 0.6);
  }
}
src/evidence/EvidenceCollector.ts
TypeScript

import { nanoid } from 'nanoid';
import type { Source, Evidence, EvidenceGraph, DeduplicationResult } from '../types/evidence.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import { EvidenceCollectionError } from '../types/errors.js';

/**
 * EvidenceCollector: transforms raw Sources into scored, de-duplicated
 * Evidence entries and assembles the EvidenceGraph.
 *
 * Design decisions:
 * - Evidence extraction is pattern-based (no LLM in Part 1).
 *   LLM-assisted evidence extraction lives in SynthesisEngine (Part 2).
 * - Claim extraction uses sentence-boundary detection + heuristic typing.
 * - Confidence is derived from source relevance + corroboration count.
 * - Contradiction detection: Jaccard similarity on claim terms ≥ 0.4
 *   + presence of negation terms in one but not the other.
 * - Corroboration: two claims share >60% of key terms → support link.
 * - Graph clusters: keyword co-occurrence grouping (greedy, O(n²) acceptable
 *   for n ≤ 500 evidence entries per research).
 */

const MAX_CLAIMS_PER_SOURCE = 8;
const MIN_CLAIM_LENGTH      = 40;
const MAX_CLAIM_LENGTH      = 400;

// Heuristic patterns for evidence typing
const EVIDENCE_TYPE_PATTERNS: Array<{
  type: Evidence['type'];
  pattern: RegExp;
}> = [
  { type: 'warning',      pattern: /\b(warning|caution|careful|danger|avoid|pitfall|gotcha)\b/i },
  { type: 'definition',   pattern: /\b(is defined as|refers to|means|is a|is an)\b/i },
  { type: 'comparison',   pattern: /\b(compared to|unlike|whereas|in contrast|better than|worse than)\b/i },
  { type: 'procedure',    pattern: /^\s*\d+\.|^step\s+\d|^first[,\s]|^then[,\s]|^finally[,\s]/im },
  { type: 'code_pattern', pattern: /```|`[^`]+`|\bfunction\b|\bclass\b|\bconst\b|\bimport\b/i },
  { type: 'example',      pattern: /\b(for example|for instance|e\.g\.|such as|like)\b/i },
  { type: 'fact',         pattern: /./ },  // default fallback
];

// Negation terms used for contradiction detection
const NEGATION_TERMS = new Set([
  'not', 'no', 'never', 'without', 'cannot', "can't", 'impossible',
  'incorrect', 'wrong', 'false', 'fails', 'broken',
]);

export class EvidenceCollector {
  constructor(private readonly store: ResearchStore) {}

  /**
   * Process all sources for a research round and produce Evidence + EvidenceGraph.
   */
  async collect(
    researchId: string,
    sources:    Source[],
    round:      number,
  ): Promise<{
    evidence:  Evidence[];
    graph:     EvidenceGraph;
    dedup:     DeduplicationResult;
  }> {
    try {
      // 1. Extract raw claims from each source
      const allEvidence: Evidence[] = [];
      const seenClaims  = new Set<string>();
      let dupCount = 0;

      for (const source of sources) {
        const claims = this.extractClaims(source, round);

        for (const claim of claims) {
          // Deduplicate claims by normalised text
          const norm = this.normaliseClaim(claim.claim);
          if (seenClaims.has(norm)) {
            dupCount++;
            continue;
          }
          seenClaims.add(norm);

          this.store.saveEvidence(claim);
          allEvidence.push(claim);
        }
      }

      // 2. Link contradictions + corroborations
      this.linkRelationships(allEvidence);

      // 3. Boost confidence of corroborated claims
      const scored = this.scoreConfidence(allEvidence);

      // 4. Build graph
      const graph = this.buildGraph(researchId, scored);

      const dedup: DeduplicationResult = {
        total:      allEvidence.length + dupCount,
        unique:     allEvidence.length,
        duplicates: dupCount,
        merged:     0,
      };

      return { evidence: scored, graph, dedup };
    } catch (err) {
      throw new EvidenceCollectionError(researchId, err);
    }
  }

  // ── Claim extraction ───────────────────────────────────────────────────────

  private extractClaims(source: Source, round: number): Evidence[] {
    const sentences  = this.splitIntoSentences(source.cleanContent);
    const claims: Evidence[] = [];

    for (const sentence of sentences.slice(0, MAX_CLAIMS_PER_SOURCE)) {
      const trimmed = sentence.trim();

      if (trimmed.length < MIN_CLAIM_LENGTH || trimmed.length > MAX_CLAIM_LENGTH) {
        continue;
      }

      const type = this.classifyType(trimmed);
      const tags = this.extractTags(trimmed);

      claims.push({
        id:          nanoid(),
        researchId:  source.researchId,
        sourceIds:   [source.id],
        type,
        claim:       trimmed,
        context:     this.buildContext(source, trimmed),
        tags,
        confidence:  source.relevanceScore,
        relevance:   source.relevanceScore,
        round,
        contradicts: [],
        supports:    [],
        createdAt:   new Date().toISOString(),
      });
    }

    return claims;
  }

  // ── Sentence splitting ────────────────────────────────────────────────────

  private splitIntoSentences(text: string): string[] {
    // Split on sentence-ending punctuation followed by whitespace + capital.
    // Handles common abbreviations poorly, but good enough for evidence extraction.
    return text
      .split(/(?<=[.!?])\s+(?=[A-Z])|(?<=\n)\s*[-•*]\s+/)
      .flatMap(s => s.length > MAX_CLAIM_LENGTH
        ? s.split(/(?<=\n)/)    // split at newlines if still too long
        : [s])
      .filter(s => s.trim().length >= MIN_CLAIM_LENGTH);
  }

  // ── Type classification ───────────────────────────────────────────────────

  private classifyType(sentence: string): Evidence['type'] {
    for (const { type, pattern } of EVIDENCE_TYPE_PATTERNS) {
      if (pattern.test(sentence)) return type;
    }
    return 'fact';
  }

  // ── Tag extraction ────────────────────────────────────────────────────────

  private extractTags(sentence: string): string[] {
    const tags: string[] = [];

    // Extract CamelCase identifiers as tags
    for (const m of sentence.matchAll(/\b([A-Z][a-zA-Z]{2,})\b/g)) {
      tags.push(m[1]!.toLowerCase());
    }

    // Extract quoted terms
    for (const m of sentence.matchAll(/"([^"]{3,20})"/g)) {
      tags.push(m[1]!.toLowerCase());
    }

    return [...new Set(tags)].slice(0, 6);
  }

  // ── Context building ──────────────────────────────────────────────────────

  private buildContext(source: Source, claim: string): string {
    const pos = source.cleanContent.indexOf(claim);
    if (pos === -1) return source.title;

    const start   = Math.max(0, pos - 100);
    const end     = Math.min(source.cleanContent.length, pos + claim.length + 100);
    const context = source.cleanContent.slice(start, end).trim();

    return `[${source.title}] ${context}`;
  }

  // ── Relationship detection ────────────────────────────────────────────────

  private linkRelationships(evidence: Evidence[]): void {
    for (let i = 0; i < evidence.length; i++) {
      for (let j = i + 1; j < evidence.length; j++) {
        const a = evidence[i]!;
        const b = evidence[j]!;

        const similarity = this.jaccardSimilarity(
          this.tokenSet(a.claim),
          this.tokenSet(b.claim),
        );

        if (similarity < 0.3) continue;

        const aHasNeg = this.hasNegation(a.claim);
        const bHasNeg = this.hasNegation(b.claim);

        if (aHasNeg !== bHasNeg && similarity >= 0.4) {
          // One negates the other → contradiction
          a.contradicts.push(b.id);
          b.contradicts.push(a.id);

          // Mark evidence type
          if (a.type !== 'contradiction') a.type = 'contradiction';
        } else if (similarity >= 0.6) {
          // High overlap, same polarity → corroboration
          a.supports.push(b.id);
          b.supports.push(a.id);
        }
      }
    }

    // Persist updated relationships
    for (const ev of evidence) {
      if (ev.contradicts.length > 0 || ev.supports.length > 0) {
        this.store.linkEvidenceRelationships(ev.id, ev.contradicts, ev.supports);
      }
    }
  }

  // ── Confidence scoring ────────────────────────────────────────────────────

  private scoreConfidence(evidence: Evidence[]): Evidence[] {
    for (const ev of evidence) {
      const corrobBoost  = Math.min(0.2, ev.supports.length * 0.05);
      const contradPenal = Math.min(0.1, ev.contradicts.length * 0.03);
      ev.confidence = Math.min(1, Math.max(0,
        ev.confidence + corrobBoost - contradPenal,
      ));
    }
    return evidence;
  }

  // ── Graph assembly ────────────────────────────────────────────────────────

  private buildGraph(researchId: string, evidence: Evidence[]): EvidenceGraph {
    const nodes      = evidence.map(e => e.id);
    const supports   = evidence.flatMap(e =>
      e.supports.map(tid => ({ from: e.id, to: tid, weight: e.confidence })),
    );
    const contradicts = evidence.flatMap(e =>
      e.contradicts.map(tid => ({ from: e.id, to: tid, weight: e.confidence })),
    );

    // Greedy keyword clustering
    const clusters = this.clusterEvidence(evidence);

    return { researchId, nodes, supports, contradicts, clusters };
  }

  private clusterEvidence(
    evidence: Evidence[],
  ): Array<{ label: string; nodeIds: string[] }> {
    // Group by most common tag (simple, O(n) pass)
    const tagGroups = new Map<string, string[]>();

    for (const ev of evidence) {
      for (const tag of ev.tags.slice(0, 2)) {
        if (!tagGroups.has(tag)) tagGroups.set(tag, []);
        tagGroups.get(tag)!.push(ev.id);
      }
    }

    // Return clusters of ≥ 2 nodes, top 10 clusters
    return Array.from(tagGroups.entries())
      .filter(([, ids]) => ids.length >= 2)
      .sort(([, a], [, b]) => b.length - a.length)
      .slice(0, 10)
      .map(([label, nodeIds]) => ({ label, nodeIds }));
  }

  // ── Utility ───────────────────────────────────────────────────────────────

  private normaliseClaim(claim: string): string {
    return claim.toLowerCase().replace(/\s+/g, ' ').trim();
  }

  private tokenSet(text: string): Set<string> {
    return new Set(
      text.toLowerCase()
        .split(/\W+/)
        .filter(t => t.length > 3),
    );
  }

  private jaccardSimilarity(a: Set<string>, b: Set<string>): number {
    if (a.size === 0 && b.size === 0) return 0;
    let intersection = 0;
    for (const t of a) if (b.has(t)) intersection++;
    const union = a.size + b.size - intersection;
    return intersection / union;
  }

  private hasNegation(text: string): boolean {
    const words = text.toLowerCase().split(/\W+/);
    return words.some(w => NEGATION_TERMS.has(w));
  }
}
src/index.ts
TypeScript

/**
 * @locoworker/autoresearch — Part 1 exports
 *
 * Part 2 will add: SynthesisEngine, ReportWriter, AutoResearchLoop,
 * ResearchToolSet, and Kairos/EventBus integration.
 */

// Types
export * from './types/index.js';

// Store
export { ResearchStore }     from './store/ResearchStore.js';

// Planning
export { GoalParser }        from './planning/GoalParser.js';
export { ResearchPlanner }   from './planning/ResearchPlanner.js';
export { QUERY_TEMPLATES }   from './planning/QueryTemplates.js';

// Query adapters
export { WebSearchAdapter }  from './query/WebSearchAdapter.js';
export { CodeSearchAdapter } from './query/CodeSearchAdapter.js';
export { WikiSearchAdapter } from './query/WikiSearchAdapter.js';
export { MemorySearchAdapter } from './query/MemorySearchAdapter.js';
export { QueryEngine }       from './query/QueryEngine.js';

// Evidence pipeline
export { ContentExtractor }  from './evidence/ContentExtractor.js';
export { SourceFetcher }     from './evidence/SourceFetcher.js';
export { EvidenceCollector } from './evidence/EvidenceCollector.js';
tests/store.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore } from '../src/store/ResearchStore.js';
import { ResearchNotFoundError } from '../src/types/errors.js';

describe('ResearchStore', () => {
  let tempDir: string;
  let store: ResearchStore;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'autoresearch-store-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  // ── Research CRUD ──────────────────────────────────────────────────────────

  test('creates research', () => {
    const r = store.createResearch({
      goal:     'How does React reconciliation work?',
      goalType: 'technical',
      depth:    'standard',
      tags:     ['react', 'reconciliation'],
    });

    expect(r.id).toBeTruthy();
    expect(r.status).toBe('pending');
    expect(r.goal).toBe('How does React reconciliation work?');
    expect(r.totalSources).toBe(0);
  });

  test('gets research by id', () => {
    const created = store.createResearch({
      goal:     'Test goal',
      goalType: 'exploratory',
      depth:    'surface',
      tags:     [],
    });

    const fetched = store.getResearch(created.id);
    expect(fetched?.id).toBe(created.id);
  });

  test('returns null for missing research', () => {
    expect(store.getResearch('nonexistent')).toBeNull();
  });

  test('throws for missing research via getOrThrow', () => {
    expect(() => store.getResearchOrThrow('nonexistent'))
      .toThrow(ResearchNotFoundError);
  });

  test('updates status', () => {
    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.updateStatus(r.id, 'planning', { startedAt: new Date().toISOString() });

    const updated = store.getResearch(r.id);
    expect(updated?.status).toBe('planning');
    expect(updated?.startedAt).toBeTruthy();
  });

  test('accumulates cost', () => {
    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.accumulateCost(r.id, 1000, 0.003);
    store.accumulateCost(r.id, 500, 0.0015);

    const updated = store.getResearch(r.id);
    expect(updated?.totalTokens).toBe(1500);
    expect(updated?.costUsd).toBeCloseTo(0.0045);
  });

  test('lists researches with status filter', () => {
    store.createResearch({ goal: 'A', goalType: 'factual', depth: 'surface', tags: [] });
    store.createResearch({ goal: 'B', goalType: 'factual', depth: 'surface', tags: [] });

    const all = store.listResearches();
    expect(all.length).toBe(2);

    const pending = store.listResearches({ status: 'pending' });
    expect(pending.length).toBe(2);

    const complete = store.listResearches({ status: 'complete' });
    expect(complete.length).toBe(0);
  });

  // ── Sources ────────────────────────────────────────────────────────────────

  test('saves source and detects duplicate fingerprint', () => {
    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const fingerprint = 'abc123fingerprint';

    store.saveSource({
      id:             'src-1',
      researchId:     r.id,
      type:           'web',
      url:            'https://example.com',
      title:          'Example',
      rawContent:     '<html>...</html>',
      cleanContent:   'Some content here',
      fingerprint,
      tokenCount:     10,
      relevanceScore: 0.8,
      fetchedAt:      new Date().toISOString(),
      httpStatus:     200,
    });

    expect(store.fingerprintExists(r.id, fingerprint)).toBe(true);
    expect(store.fingerprintExists(r.id, 'different')).toBe(false);
  });

  test('getSources respects minRelevance filter', () => {
    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.saveSource({
      id: 'src-hi', researchId: r.id, type: 'web',
      title: 'High', rawContent: '', cleanContent: 'hi',
      fingerprint: 'fp-hi', tokenCount: 5, relevanceScore: 0.9,
      fetchedAt: new Date().toISOString(),
    });

    store.saveSource({
      id: 'src-lo', researchId: r.id, type: 'web',
      title: 'Low', rawContent: '', cleanContent: 'lo',
      fingerprint: 'fp-lo', tokenCount: 5, relevanceScore: 0.2,
      fetchedAt: new Date().toISOString(),
    });

    const highOnly = store.getSources(r.id, { minRelevance: 0.5 });
    expect(highOnly).toHaveLength(1);
    expect(highOnly[0]?.title).toBe('High');
  });

  // ── Evidence ───────────────────────────────────────────────────────────────

  test('saves and retrieves evidence', () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    store.saveEvidence({
      id:          'ev-1',
      researchId:  r.id,
      sourceIds:   ['src-1'],
      type:        'fact',
      claim:       'React uses a virtual DOM for efficient updates',
      context:     'From React docs',
      tags:        ['react', 'virtualdom'],
      confidence:  0.9,
      relevance:   0.85,
      round:       1,
      contradicts: [],
      supports:    [],
      createdAt:   new Date().toISOString(),
    });

    const evidence = store.getEvidence(r.id);
    expect(evidence).toHaveLength(1);
    expect(evidence[0]?.claim).toContain('React');
    expect(evidence[0]?.type).toBe('fact');
  });

  test('links evidence relationships', () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    store.saveEvidence({
      id: 'ev-1', researchId: r.id, sourceIds: [],
      type: 'fact', claim: 'X is fast', context: '',
      tags: [], confidence: 0.8, relevance: 0.8,
      round: 1, contradicts: [], supports: [],
      createdAt: new Date().toISOString(),
    });

    store.saveEvidence({
      id: 'ev-2', researchId: r.id, sourceIds: [],
      type: 'fact', claim: 'X is slow', context: '',
      tags: [], confidence: 0.7, relevance: 0.7,
      round: 1, contradicts: [], supports: [],
      createdAt: new Date().toISOString(),
    });

    store.linkEvidenceRelationships('ev-1', ['ev-2'], []);
    store.linkEvidenceRelationships('ev-2', ['ev-1'], []);

    const evidence = store.getEvidence(r.id);
    const ev1 = evidence.find(e => e.id === 'ev-1');
    expect(ev1?.contradicts).toContain('ev-2');
  });
});
tests/planner.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore } from '../src/store/ResearchStore.js';
import { GoalParser } from '../src/planning/GoalParser.js';
import { ResearchPlanner } from '../src/planning/ResearchPlanner.js';

describe('GoalParser', () => {
  const parser = new GoalParser();

  test('classifies technical goal', () => {
    const result = parser.parse('How does the React reconciliation algorithm work?');
    expect(result.goalType).toBe('technical');
  });

  test('classifies comparative goal', () => {
    const result = parser.parse('React vs Vue: which is better for large apps?');
    expect(result.goalType).toBe('comparative');
    expect(result.comparands).toHaveLength(2);
    expect(result.comparands[0]).toContain('react');
    expect(result.comparands[1]).toContain('vue');
  });

  test('classifies procedural goal', () => {
    const result = parser.parse('How to set up a Turborepo monorepo with pnpm?');
    expect(result.goalType).toBe('procedural');
  });

  test('classifies investigative goal', () => {
    const result = parser.parse('Why does my React app re-render so many times?');
    expect(result.goalType).toBe('investigative');
  });

  test('extracts CamelCase key terms', () => {
    const result = parser.parse('How does QueryEngine dispatch to WebSearchAdapter?');
    expect(result.keyTerms).toContain('QueryEngine');
    expect(result.keyTerms).toContain('WebSearchAdapter');
  });

  test('extracts quoted terms', () => {
    const result = parser.parse('What is "token bucket" rate limiting?');
    expect(result.keyTerms).toContain('token bucket');
  });

  test('extracts constraints', () => {
    const result = parser.parse('How to use React v18.2 with TypeScript?');
    expect(result.constraints.some(c => c.startsWith('language:typescript'))).toBe(true);
    expect(result.constraints.some(c => c.startsWith('version:'))).toBe(true);
  });

  test('extracts sub-questions', () => {
    const result = parser.parse(
      'How does React work? Why is it fast? What is the virtual DOM?',
    );
    expect(result.subQuestions.length).toBeGreaterThan(1);
  });
});

describe('ResearchPlanner', () => {
  let tempDir: string;
  let store: ResearchStore;
  let planner: ResearchPlanner;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'planner-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
    planner = new ResearchPlanner(store);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  test('generates surface plan with round 1 queries only', async () => {
    const r = store.createResearch({
      goal:     'What is SQLite WAL mode?',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const { plan, stats } = await planner.plan(r.id, r.goal, 'surface');

    expect(plan.rounds).toBe(1);
    expect(stats.totalQueries).toBeGreaterThan(0);
    expect(plan.queries.every(q => q.round === 1)).toBe(true);
  });

  test('generates standard plan with rounds 1 and 2', async () => {
    const r = store.createResearch({
      goal:     'How does Turborepo caching work?',
      goalType: 'technical',
      depth:    'standard',
      tags:     [],
    });

    const { plan } = await planner.plan(r.id, r.goal, 'standard');

    expect(plan.rounds).toBe(2);

    const rounds = new Set(plan.queries.map(q => q.round));
    expect(rounds.has(1)).toBe(true);
    expect(rounds.has(2)).toBe(true);
  });

  test('generates deep plan with up to 3 rounds', async () => {
    const r = store.createResearch({
      goal:     'React vs Vue for enterprise apps',
      goalType: 'comparative',
      depth:    'deep',
      tags:     [],
    });

    const { plan } = await planner.plan(r.id, r.goal, 'deep');

    expect(plan.rounds).toBe(3);
  });

  test('deduplicates query texts within plan', async () => {
    const r = store.createResearch({
      goal:     'What is TypeScript?',
      goalType: 'factual',
      depth:    'standard',
      tags:     [],
    });

    const { plan } = await planner.plan(r.id, r.goal, 'standard');

    const texts = plan.queries.map(q => q.text.toLowerCase());
    const unique = new Set(texts);
    expect(unique.size).toBe(texts.length);
  });

  test('persists queries to store', async () => {
    const r = store.createResearch({
      goal:     'How does pLimit work?',
      goalType: 'technical',
      depth:    'surface',
      tags:     [],
    });

    const { plan } = await planner.plan(r.id, r.goal, 'surface');

    const pending = store.getPendingQueriesForRound(r.id, 1);
    expect(pending.length).toBe(plan.queries.filter(q => q.round === 1).length);
  });

  test('refineWithGaps generates follow-up queries', async () => {
    const r = store.createResearch({
      goal:     'How does WAL mode work?',
      goalType: 'technical',
      depth:    'standard',
      tags:     [],
    });

    const { plan } = await planner.plan(r.id, r.goal, 'standard');

    const newQueries = await planner.refineWithGaps(
      plan.id,
      r.id,
      2,
      ['WAL checkpointing', 'WAL reader concurrency'],
    );

    expect(newQueries.length).toBeGreaterThan(0);
    expect(newQueries.every(q => q.round === 2)).toBe(true);
  });
});
tests/evidence-collector.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore } from '../src/store/ResearchStore.js';
import { EvidenceCollector } from '../src/evidence/EvidenceCollector.js';
import type { Source } from '../src/types/evidence.js';

describe('EvidenceCollector', () => {
  let tempDir: string;
  let store: ResearchStore;
  let collector: EvidenceCollector;

  beforeEach(() => {
    tempDir  = mkdtempSync(join(tmpdir(), 'evidence-test-'));
    store    = new ResearchStore(join(tempDir, 'research.db'));
    collector = new EvidenceCollector(store);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  const makeSource = (
    researchId: string,
    content: string,
    score = 0.8,
  ): Source => ({
    id:             `src-${Math.random().toString(36).slice(2)}`,
    researchId,
    type:           'web',
    url:            'https://example.com',
    title:          'Test Source',
    rawContent:     content,
    cleanContent:   content,
    fingerprint:    `fp-${Math.random()}`,
    tokenCount:     Math.ceil(content.length / 4),
    relevanceScore: score,
    fetchedAt:      new Date().toISOString(),
  });

  test('extracts evidence from sources', async () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    const sources: Source[] = [
      makeSource(r.id,
        'React uses a virtual DOM for efficient updates to the real DOM. ' +
        'This allows React to batch multiple DOM changes together. ' +
        'For example, when state changes, React re-renders the component tree.',
      ),
    ];

    const { evidence, dedup } = await collector.collect(r.id, sources, 1);

    expect(evidence.length).toBeGreaterThan(0);
    expect(dedup.total).toBeGreaterThanOrEqual(dedup.unique);
  });

  test('classifies warning evidence type', async () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    const sources: Source[] = [
      makeSource(r.id,
        'Warning: avoid calling hooks conditionally because this violates ' +
        'the rules of hooks and will cause unexpected behaviour in your components.',
      ),
    ];

    const { evidence } = await collector.collect(r.id, sources, 1);

    const warnings = evidence.filter(e => e.type === 'warning');
    expect(warnings.length).toBeGreaterThan(0);
  });

  test('classifies definition evidence type', async () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    const sources: Source[] = [
      makeSource(r.id,
        'A hook is defined as a function that allows you to use React state ' +
        'and lifecycle features from function components.',
      ),
    ];

    const { evidence } = await collector.collect(r.id, sources, 1);

    const defs = evidence.filter(e => e.type === 'definition');
    expect(defs.length).toBeGreaterThan(0);
  });

  test('builds evidence graph with nodes', async () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    const sources: Source[] = [
      makeSource(r.id,
        'React is fast because it uses a virtual DOM for diffing. ' +
        'The reconciler compares old and new virtual DOM trees. ' +
        'React batches state updates for better performance.',
      ),
      makeSource(r.id,
        'React Fiber is the modern reconciliation algorithm in React. ' +
        'Fiber allows React to pause and resume rendering work. ' +
        'This makes React more responsive on slow devices.',
      ),
    ];

    const { graph } = await collector.collect(r.id, sources, 1);

    expect(graph.nodes.length).toBeGreaterThan(0);
    expect(graph.researchId).toBe(r.id);
  });

  test('deduplicates identical claims across sources', async () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'factual', depth: 'surface', tags: [],
    });

    const repeatedContent =
      'React uses hooks to manage state in functional components. ' +
      'The useState hook returns a tuple of the current value and setter.';

    const sources: Source[] = [
      makeSource(r.id, repeatedContent, 0.9),
      makeSource(r.id, repeatedContent, 0.8),  // same content, different source
    ];

    const { dedup } = await collector.collect(r.id, sources, 1);

    expect(dedup.duplicates).toBeGreaterThan(0);
    expect(dedup.unique).toBeLessThan(dedup.total);
  });
});
tests/source-fetcher.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore } from '../src/store/ResearchStore.js';
import { SourceFetcher } from '../src/evidence/SourceFetcher.js';
import { ContentExtractor } from '../src/evidence/ContentExtractor.js';

describe('SourceFetcher.buildLocalSource', () => {
  let tempDir: string;
  let store: ResearchStore;
  let fetcher: SourceFetcher;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'fetcher-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
    fetcher = new SourceFetcher(store);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  test('builds code source with correct fingerprint', () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'technical', depth: 'surface', tags: [],
    });

    const s = fetcher.buildLocalSource(r.id, {
      type:    'code',
      nodeId:  'node-123',
      title:   'function: queryLoop',
      content: 'async function* queryLoop(sessionId, message) { yield... }',
      score:   0.9,
    });

    expect(s.type).toBe('code');
    expect(s.nodeId).toBe('node-123');
    expect(s.fingerprint).toBeTruthy();
    expect(s.tokenCount).toBeGreaterThan(0);
  });

  test('same content produces same fingerprint', () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'technical', depth: 'surface', tags: [],
    });

    const content = 'This is deterministic content for testing';
    const s1 = fetcher.buildLocalSource(r.id, { type: 'wiki', wikiSlug: 'test', title: 'T', content, score: 0.5 });
    const s2 = fetcher.buildLocalSource(r.id, { type: 'wiki', wikiSlug: 'test', title: 'T', content, score: 0.7 });

    expect(s1.fingerprint).toBe(s2.fingerprint);
  });

  test('different content produces different fingerprints', () => {
    const r = store.createResearch({
      goal: 'Test', goalType: 'technical', depth: 'surface', tags: [],
    });

    const s1 = fetcher.buildLocalSource(r.id, { type: 'memory', title: 'A', content: 'Content A version 1', score: 0.5 });
    const s2 = fetcher.buildLocalSource(r.id, { type: 'memory', title: 'B', content: 'Content B version 2', score: 0.5 });

    expect(s1.fingerprint).not.toBe(s2.fingerprint);
  });
});

describe('ContentExtractor', () => {
  const extractor = new ContentExtractor();

  test('extracts title from h1', () => {
    const html = '<html><body><h1>My Article</h1><p>Some content here that is long enough to matter.</p></body></html>';
    const result = extractor.extract(html);
    expect(result.title).toBe('My Article');
  });

  test('falls back to title tag', () => {
    const html = '<html><head><title>Page Title</title></head><body><p>content</p></body></html>';
    const result = extractor.extract(html);
    expect(result.title).toBe('Page Title');
  });

  test('removes nav and footer noise', () => {
    const html = `
      <html><body>
        <nav>Navigation links here</nav>
        <main><p>This is the real content about React hooks.</p></main>
        <footer>Footer content</footer>
      </body></html>
    `;
    const result = extractor.extract(html);
    expect(result.markdown).not.toContain('Navigation links');
    expect(result.markdown).not.toContain('Footer content');
    expect(result.markdown).toContain('React hooks');
  });

  test('estimates token count', () => {
    const html = '<html><body><p>' + 'word '.repeat(100) + '</p></body></html>';
    const result = extractor.extract(html);
    expect(result.tokenEstimate).toBeGreaterThan(0);
    expect(result.tokenEstimate).toBeLessThanOrEqual(Math.ceil(result.charCount / 4) + 1);
  });

  test('truncates very long content', () => {
    const longContent = 'This is a very long sentence with meaningful content. '.repeat(500);
    const html = `<html><body><main>${longContent}</main></body></html>`;
    const result = extractor.extract(html);
    expect(result.charCount).toBeLessThanOrEqual(12_000);
  });
});
Summary: Pass 9 Part 1
What Part 1 delivers
Layer	Components	Key design decisions
Types	research.ts, evidence.ts, planning.ts, errors.ts	Status lifecycle, depth levels, goal types, Zod schemas throughout
Store	ResearchStore, schema.sql	Single-writer SQLite, WAL mode, cascade deletes, fingerprint de-dup, consistent with WikiStore/GatewayStore/OrchestratorStore pattern
Planning	GoalParser, ResearchPlanner, QueryTemplates	Pattern-based goal parsing (zero LLM), template library per depth×goalType, round-based query generation, gap-fill refinement
Query adapters	WebSearchAdapter, CodeSearchAdapter, WikiSearchAdapter, MemorySearchAdapter	Provider-agnostic web (Brave/SerpAPI), per-domain rate limiting, graphify/wiki/memory adapters are pure interface wrappers
Query engine	QueryEngine	p-limit concurrency control (web=3, local=5), per-query error isolation, gap detection, fan-out across all source types
Evidence pipeline	ContentExtractor, SourceFetcher, EvidenceCollector	Cheerio+Turndown HTML→Markdown, SHA-256 fingerprint de-dup, p-retry on fetch, sentence-boundary evidence extraction, Jaccard contradiction/corroboration detection, keyword cluster graph
Tests	5 test files	Store CRUD, GoalParser classification, ResearchPlanner round generation, EvidenceCollector typing/dedup/graph, SourceFetcher fingerprinting/ContentExtractor noise removal



Pass 9 Part 2: packages/autoresearch — Synthesis, Reports, Loop, Kairos & Tools
Building directly on Part 1's foundation (store, planning, query engine, evidence pipeline), Part 2 delivers the synthesis layer, report writing, the top-level async-generator research loop, Kairos scheduling integration, the agent tool set, CLI commands, and the full test suite.

New/Updated File Tree (Part 2 additions)
text

packages/autoresearch/
├── src/
│   ├── synthesis/
│   │   ├── SynthesisEngine.ts       ← multi-doc synthesis via agent loop
│   │   ├── PromptBuilder.ts         ← token-budgeted synthesis prompts
│   │   └── SynthesisValidator.ts    ← validate synthesis completeness
│   ├── report/
│   │   ├── ReportWriter.ts          ← Markdown report generation
│   │   ├── ReportFormatter.ts       ← section builders + citation renderer
│   │   └── WikiPusher.ts            ← push report to wiki package
│   ├── loop/
│   │   ├── AutoResearchLoop.ts      ← top-level async-generator orchestrator
│   │   └── LoopSupervisor.ts        ← cost/time/depth guards
│   ├── kairos/
│   │   └── KairosIntegration.ts     ← register job handler + emit observations
│   ├── tools/
│   │   └── ResearchToolSet.ts       ← tools exposed via core ToolRegistry
│   ├── cli/
│   │   └── commands.ts              ← research:run + research:report CLI
│   └── index.ts                     ← updated full exports
└── tests/
    ├── synthesis.test.ts
    ├── report.test.ts
    ├── loop.test.ts
    ├── kairos-integration.test.ts
    └── tools.test.ts
src/synthesis/PromptBuilder.ts
TypeScript

import type { Evidence, Source, EvidenceGraph } from '../types/evidence.js';
import type { Research } from '../types/research.js';

/**
 * PromptBuilder: assembles token-budgeted synthesis prompts.
 *
 * Design decisions:
 * - Hard token budget enforced by char/4 heuristic (consistent with
 *   AdaptiveCompactor in packages/core Pass 2).
 * - Evidence is ranked by relevance DESC then confidence DESC before
 *   inclusion; low-signal evidence is dropped when budget is tight.
 * - Contradiction pairs are explicitly surfaced so the model can
 *   reason about conflicting claims rather than silently ignoring one.
 * - Source citations use [S:N] shorthand expanded in ReportFormatter.
 * - Prompts are goal-type-specific: comparative prompts request a
 *   structured trade-off table; procedural prompts request ordered steps.
 */

const CHARS_PER_TOKEN = 4;

export interface SynthesisPrompt {
  system:       string;
  user:         string;
  evidenceUsed: Evidence[];
  sourcesUsed:  Source[];
  tokenEstimate: number;
}

const GOAL_TYPE_INSTRUCTIONS: Record<string, string> = {
  technical:
    'Focus on implementation details, architecture decisions, and code patterns. ' +
    'Highlight trade-offs, performance implications, and known pitfalls.',
  factual:
    'Provide a clear, accurate answer with supporting evidence. ' +
    'Distinguish well-established facts from uncertain claims.',
  exploratory:
    'Produce a comprehensive survey covering key concepts, major themes, ' +
    'and notable examples. Organise by topic rather than by source.',
  investigative:
    'Identify root causes, contributing factors, and evidence chains. ' +
    'Explicitly note where evidence is incomplete or contradictory.',
  comparative:
    'Produce a structured comparison with a summary table. ' +
    'Address each dimension (performance, DX, ecosystem, maturity) separately.',
  procedural:
    'Produce numbered, actionable steps. ' +
    'Highlight prerequisites, common mistakes, and verification checkpoints.',
};

export class PromptBuilder {
  /**
   * Build a synthesis prompt within the given token budget.
   */
  build(
    research:  Research,
    evidence:  Evidence[],
    sources:   Source[],
    graph:     EvidenceGraph,
    opts: {
      tokenBudget:        number;    // total tokens available for prompt
      systemReserve?:     number;    // tokens reserved for system prompt (default 500)
      responseReserve?:   number;    // tokens reserved for model response (default 2000)
    },
  ): SynthesisPrompt {
    const systemReserve  = opts.systemReserve  ?? 500;
    const responseReserve = opts.responseReserve ?? 2000;
    const evidenceBudget = opts.tokenBudget - systemReserve - responseReserve;

    // Sort evidence: relevance DESC then confidence DESC
    const ranked = [...evidence].sort(
      (a, b) => b.relevance - a.relevance || b.confidence - a.confidence,
    );

    // Build source index (id → number) for [S:N] citations
    const sourceIndex = new Map<string, number>();
    sources.forEach((s, i) => sourceIndex.set(s.id, i + 1));

    // Select evidence within budget
    const { selected, budgetUsed } = this.selectEvidence(ranked, evidenceBudget);

    // Identify contradiction pairs from graph
    const contradictions = this.buildContradictionPairs(selected, graph);

    // Build corroboration clusters
    const clusters = graph.clusters.slice(0, 5);

    // Assemble system prompt
    const system = this.buildSystemPrompt(research, selected.length);

    // Assemble user prompt
    const user = this.buildUserPrompt(
      research,
      selected,
      sources,
      sourceIndex,
      contradictions,
      clusters,
    );

    const tokenEstimate = Math.ceil(
      (system.length + user.length) / CHARS_PER_TOKEN,
    );

    // Map selected evidence back to their sources
    const usedSourceIds = new Set(selected.flatMap(e => e.sourceIds));
    const sourcesUsed   = sources.filter(s => usedSourceIds.has(s.id));

    return { system, user, evidenceUsed: selected, sourcesUsed, tokenEstimate };
  }

  // ── Evidence selection ────────────────────────────────────────────────────

  private selectEvidence(
    ranked:  Evidence[],
    budget:  number,
  ): { selected: Evidence[]; budgetUsed: number } {
    const selected: Evidence[] = [];
    let budgetUsed = 0;
    const budgetChars = budget * CHARS_PER_TOKEN;

    for (const ev of ranked) {
      const cost = Math.ceil(
        (ev.claim.length + ev.context.length) / CHARS_PER_TOKEN,
      );

      if (budgetUsed + cost > budget) break;

      selected.push(ev);
      budgetUsed += cost;
    }

    return { selected, budgetUsed };
  }

  // ── Contradiction pairs ───────────────────────────────────────────────────

  private buildContradictionPairs(
    evidence: Evidence[],
    graph:    EvidenceGraph,
  ): Array<{ a: Evidence; b: Evidence }> {
    const idMap = new Map(evidence.map(e => [e.id, e]));
    const pairs: Array<{ a: Evidence; b: Evidence }> = [];
    const seen  = new Set<string>();

    for (const edge of graph.contradicts) {
      const key = [edge.from, edge.to].sort().join('|');
      if (seen.has(key)) continue;
      seen.add(key);

      const a = idMap.get(edge.from);
      const b = idMap.get(edge.to);
      if (a && b) pairs.push({ a, b });
    }

    return pairs.slice(0, 6); // cap at 6 pairs to avoid prompt bloat
  }

  // ── System prompt ─────────────────────────────────────────────────────────

  private buildSystemPrompt(research: Research, evidenceCount: number): string {
    const instruction = GOAL_TYPE_INSTRUCTIONS[research.goalType] ??
      GOAL_TYPE_INSTRUCTIONS.exploratory!;

    return [
      'You are a research synthesis assistant for LocoWorker.',
      'You have been given a collection of evidence gathered from multiple sources.',
      '',
      `Research goal: "${research.goal}"`,
      `Goal type: ${research.goalType}`,
      `Depth: ${research.depth} (${evidenceCount} evidence items provided)`,
      '',
      'TASK:',
      instruction,
      '',
      'RULES:',
      '- Cite every factual claim with [S:N] where N is the source number.',
      '- If evidence contradicts itself, acknowledge both views and note the conflict.',
      '- Do not fabricate information not present in the evidence.',
      '- Keep the response concise but complete.',
      '- Use Markdown formatting with clear section headers.',
    ].join('\n');
  }

  // ── User prompt ───────────────────────────────────────────────────────────

  private buildUserPrompt(
    research:      Research,
    evidence:      Evidence[],
    sources:       Source[],
    sourceIndex:   Map<string, number>,
    contradictions: Array<{ a: Evidence; b: Evidence }>,
    clusters:      Array<{ label: string; nodeIds: string[] }>,
  ): string {
    const parts: string[] = [];

    // Evidence block
    parts.push('## Evidence\n');
    for (const ev of evidence) {
      const srcNums = ev.sourceIds
        .map(id => sourceIndex.get(id))
        .filter(Boolean)
        .map(n => `[S:${n}]`)
        .join(' ');

      parts.push(
        `**[${ev.type.toUpperCase()}]** (relevance: ${ev.relevance.toFixed(2)}, ` +
        `confidence: ${ev.confidence.toFixed(2)}) ${srcNums}\n` +
        `${ev.claim}\n`,
      );
    }

    // Contradictions block
    if (contradictions.length > 0) {
      parts.push('\n## Identified Contradictions\n');
      for (const { a, b } of contradictions) {
        const aNum = sourceIndex.get(a.sourceIds[0] ?? '') ?? '?';
        const bNum = sourceIndex.get(b.sourceIds[0] ?? '') ?? '?';
        parts.push(
          `- **CONFLICT**: [S:${aNum}] "${a.claim.slice(0, 120)}" ` +
          `contradicts [S:${bNum}] "${b.claim.slice(0, 120)}"\n`,
        );
      }
    }

    // Theme clusters
    if (clusters.length > 0) {
      parts.push('\n## Identified Themes\n');
      parts.push(clusters.map(c => `- ${c.label} (${c.nodeIds.length} items)`).join('\n'));
    }

    // Source reference list
    parts.push('\n## Sources\n');
    const usedIds = new Set(evidence.flatMap(e => e.sourceIds));
    for (const source of sources.filter(s => usedIds.has(s.id))) {
      const n = sourceIndex.get(source.id) ?? '?';
      const ref = source.url ?? source.wikiSlug ?? source.nodeId ?? source.filePath ?? 'local';
      parts.push(`[S:${n}] **${source.title}** — ${ref}`);
    }

    // Final instruction
    parts.push(`\n---\nSynthesize the above evidence into a complete answer for: "${research.goal}"`);

    return parts.join('\n');
  }
}
src/synthesis/SynthesisValidator.ts
TypeScript

import type { Research } from '../types/research.js';

/**
 * SynthesisValidator: checks whether a synthesis output is sufficiently
 * complete and well-formed before it is handed to ReportWriter.
 *
 * Design decisions:
 * - Pure logic, no I/O.
 * - Checks are heuristic and permissive: we want to catch catastrophic
 *   failures (empty output, refused response) not stylistic issues.
 * - Validation result includes specific failure reasons for retry logic
 *   in SynthesisEngine.
 */

export interface ValidationResult {
  valid:    boolean;
  score:    number;           // 0.0–1.0
  reasons:  string[];         // failure reasons
  warnings: string[];         // non-fatal issues
}

const MIN_SYNTHESIS_CHARS   = 200;
const MIN_CITATION_COUNT    = 1;
const REFUSAL_PATTERNS      = [
  /i (cannot|can't|am unable to) (help|assist|provide)/i,
  /this (request|topic) (is|seems) (inappropriate|harmful)/i,
  /i don't have (access|information) about/i,
];

export class SynthesisValidator {
  validate(
    synthesis: string,
    research:  Research,
  ): ValidationResult {
    const reasons:  string[] = [];
    const warnings: string[] = [];

    // 1. Not empty
    if (!synthesis || synthesis.trim().length === 0) {
      return { valid: false, score: 0, reasons: ['Empty synthesis output'], warnings };
    }

    // 2. Minimum length
    if (synthesis.length < MIN_SYNTHESIS_CHARS) {
      reasons.push(
        `Synthesis too short: ${synthesis.length} chars (min ${MIN_SYNTHESIS_CHARS})`,
      );
    }

    // 3. No refusal patterns
    for (const pattern of REFUSAL_PATTERNS) {
      if (pattern.test(synthesis)) {
        reasons.push('Model refused to synthesize');
        break;
      }
    }

    // 4. Has at least one citation
    const citationCount = (synthesis.match(/\[S:\d+\]/g) ?? []).length;
    if (citationCount < MIN_CITATION_COUNT) {
      warnings.push('No source citations found — synthesis may be fabricated');
    }

    // 5. Has at least one markdown header
    if (!/^#{1,3}\s/m.test(synthesis)) {
      warnings.push('No markdown headers found — synthesis may be poorly structured');
    }

    // 6. Goal-type specific checks
    if (research.goalType === 'comparative') {
      if (!/\|\s*[-:]+\s*\|/.test(synthesis) && !/(vs|versus|compared)/i.test(synthesis)) {
        warnings.push('Comparative synthesis lacks a comparison table or explicit comparison');
      }
    }

    if (research.goalType === 'procedural') {
      if (!/^\s*\d+\./m.test(synthesis)) {
        warnings.push('Procedural synthesis lacks numbered steps');
      }
    }

    // 7. Compute score
    const penaltyPerReason = 0.3;
    const penaltyPerWarning = 0.05;
    const score = Math.max(
      0,
      1.0 - reasons.length * penaltyPerReason - warnings.length * penaltyPerWarning,
    );

    return {
      valid:    reasons.length === 0,
      score,
      reasons,
      warnings,
    };
  }
}
src/synthesis/SynthesisEngine.ts
TypeScript

import type { Research } from '../types/research.js';
import type { Evidence, Source, EvidenceGraph } from '../types/evidence.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import { PromptBuilder } from './PromptBuilder.js';
import { SynthesisValidator } from './SynthesisValidator.js';
import { AutoResearchError } from '../types/errors.js';

/**
 * SynthesisEngine: calls the agent loop (packages/core ProviderRouter)
 * with a token-budgeted synthesis prompt and returns the synthesized text.
 *
 * Design decisions:
 * - Uses ProviderRouter directly (not queryLoop) because synthesis is a
 *   single-turn call, not a multi-turn session. This keeps cost minimal.
 * - Up to MAX_RETRIES attempts if SynthesisValidator rejects the output.
 * - Retry uses a slightly more direct prompt each time.
 * - Token usage is recorded back to the Research via store.accumulateCost().
 * - Falls back to a "best-effort" concatenation of top evidence if
 *   all retries fail (so the report writer always has something to work with).
 */

const MAX_RETRIES       = 2;
const SYNTHESIS_BUDGET  = 8_000;   // tokens available for the synthesis prompt
const FALLBACK_MAX_EVIDENCE = 10;

export interface SynthesisResult {
  text:          string;
  tokensUsed:    number;
  costUsd:       number;
  attempts:      number;
  validationScore: number;
  isFallback:    boolean;
}

export class SynthesisEngine {
  private promptBuilder: PromptBuilder;
  private validator:     SynthesisValidator;

  constructor(
    private readonly store:          ResearchStore,
    private readonly providerRouter: {
      call: (opts: {
        messages:    Array<{ role: string; content: string }>;
        model?:      string;
        maxTokens?:  number;
        system?:     string;
      }) => Promise<{
        content:  string;
        usage?:   { inputTokens: number; outputTokens: number; costUsd?: number };
      }>;
    },
    private readonly modelConfig?: {
      model?:     string;
      maxTokens?: number;
    },
  ) {
    this.promptBuilder = new PromptBuilder();
    this.validator     = new SynthesisValidator();
  }

  /**
   * Synthesize evidence into a coherent research answer.
   */
  async synthesize(
    research:  Research,
    evidence:  Evidence[],
    sources:   Source[],
    graph:     EvidenceGraph,
  ): Promise<SynthesisResult> {
    if (evidence.length === 0) {
      return this.buildFallback(evidence, sources, 0);
    }

    let lastText            = '';
    let lastValidationScore = 0;

    for (let attempt = 1; attempt <= MAX_RETRIES + 1; attempt++) {
      const prompt = this.promptBuilder.build(
        research,
        evidence,
        sources,
        graph,
        { tokenBudget: SYNTHESIS_BUDGET },
      );

      let response: { content: string; usage?: { inputTokens: number; outputTokens: number; costUsd?: number } };

      try {
        response = await this.providerRouter.call({
          system:    prompt.system,
          messages:  [{ role: 'user', content: prompt.user }],
          model:     this.modelConfig?.model,
          maxTokens: this.modelConfig?.maxTokens ?? 4096,
        });
      } catch (err) {
        if (attempt > MAX_RETRIES) {
          // All attempts failed at the provider level
          return this.buildFallback(evidence, sources, attempt);
        }
        continue;
      }

      const tokensUsed = (response.usage?.inputTokens ?? 0) +
                         (response.usage?.outputTokens ?? 0);
      const costUsd    = response.usage?.costUsd ?? 0;

      // Record cost even if we retry
      if (tokensUsed > 0) {
        this.store.accumulateCost(research.id, tokensUsed, costUsd);
      }

      // Validate
      const validation = this.validator.validate(response.content, research);

      if (validation.valid || attempt > MAX_RETRIES) {
        return {
          text:            response.content,
          tokensUsed,
          costUsd,
          attempts:        attempt,
          validationScore: validation.score,
          isFallback:      false,
        };
      }

      // Log validation failures and retry
      console.warn(
        `[SynthesisEngine] Attempt ${attempt} failed validation:`,
        validation.reasons,
      );

      lastText            = response.content;
      lastValidationScore = validation.score;
    }

    // Exceeded retries: return the last attempt (even if invalid)
    return {
      text:            lastText,
      tokensUsed:      0,
      costUsd:         0,
      attempts:        MAX_RETRIES + 1,
      validationScore: lastValidationScore,
      isFallback:      false,
    };
  }

  /**
   * Fallback synthesis: concatenate top evidence as a best-effort response.
   * Used when provider calls fail or evidence is empty.
   */
  private buildFallback(
    evidence: Evidence[],
    sources:  Source[],
    attempts: number,
  ): SynthesisResult {
    const top = evidence
      .sort((a, b) => b.relevance - a.relevance)
      .slice(0, FALLBACK_MAX_EVIDENCE);

    const text = top.length > 0
      ? [
          '## Research Summary (Fallback — Synthesis Unavailable)\n',
          ...top.map((ev, i) =>
            `**${i + 1}. [${ev.type}]** ${ev.claim}`,
          ),
        ].join('\n')
      : '## Research Summary\n\nNo evidence was collected for this research.';

    return {
      text,
      tokensUsed:      0,
      costUsd:         0,
      attempts,
      validationScore: 0.3,
      isFallback:      true,
    };
  }
}
src/report/ReportFormatter.ts
TypeScript

import type { Research } from '../types/research.js';
import type { Source, Evidence } from '../types/evidence.js';
import type { QueryPlan } from '../types/planning.js';

/**
 * ReportFormatter: assembles the final Markdown report from synthesis +
 * metadata sections.
 *
 * Design decisions:
 * - Consistent with LocoWorker's wiki Markdown conventions (Pass 5):
 *   YAML frontmatter, heading hierarchy, wikilink syntax.
 * - Source appendix uses numbered reference list ([S:N] citations).
 * - Report includes a "Research Trail" section (plan stats + gaps) so the
 *   reader can understand how the evidence was gathered.
 * - Token estimate is included for context-budget awareness.
 */

export interface FormattedReport {
  frontmatter: string;
  body:        string;
  full:        string;         // frontmatter + body combined
  slug:        string;         // kebab-case wiki slug
  title:       string;
  wordCount:   number;
  charCount:   number;
}

export class ReportFormatter {
  /**
   * Format a complete research report.
   */
  format(opts: {
    research:   Research;
    synthesis:  string;
    sources:    Source[];
    evidence:   Evidence[];
    plan?:      QueryPlan;
    gaps?:      string[];
  }): FormattedReport {
    const {
      research, synthesis, sources, evidence, plan, gaps = [],
    } = opts;

    const title = this.buildTitle(research.goal);
    const slug  = this.goalToSlug(research.goal);
    const now   = new Date().toISOString();

    // 1. YAML frontmatter (wiki-compatible)
    const frontmatter = this.buildFrontmatter({
      title,
      slug,
      research,
      now,
      sourceCount: sources.length,
      evidenceCount: evidence.length,
    });

    // 2. Body sections
    const sections: string[] = [];

    // Executive summary (first paragraph of synthesis)
    const summary = this.extractSummary(synthesis);
    if (summary) {
      sections.push('## Executive Summary\n');
      sections.push(summary + '\n');
    }

    // Main synthesis body
    sections.push('## Research Findings\n');
    sections.push(synthesis + '\n');

    // Contradictions callout
    const contradictions = evidence.filter(e => e.type === 'contradiction');
    if (contradictions.length > 0) {
      sections.push('## ⚠️ Conflicting Evidence\n');
      for (const c of contradictions.slice(0, 5)) {
        sections.push(`> ${c.claim}\n`);
      }
    }

    // Research trail
    sections.push(this.buildResearchTrail(research, plan, gaps));

    // Source appendix
    sections.push(this.buildSourceAppendix(sources));

    const body = sections.join('\n');

    return {
      frontmatter,
      body,
      full:       frontmatter + '\n' + body,
      slug,
      title,
      wordCount:  body.split(/\s+/).length,
      charCount:  body.length,
    };
  }

  // ── Frontmatter ───────────────────────────────────────────────────────────

  private buildFrontmatter(opts: {
    title:         string;
    slug:          string;
    research:      Research;
    now:           string;
    sourceCount:   number;
    evidenceCount: number;
  }): string {
    const tags = [
      'autoresearch',
      opts.research.goalType,
      opts.research.depth,
      ...opts.research.tags,
    ];

    return [
      '---',
      `title: "${opts.title}"`,
      `slug: ${opts.slug}`,
      `status: active`,
      `author: autoresearch`,
      `tags: [${tags.map(t => `"${t}"`).join(', ')}]`,
      `research_id: ${opts.research.id}`,
      `research_goal: "${opts.research.goal.replace(/"/g, "'")}"`,
      `research_depth: ${opts.research.depth}`,
      `research_goal_type: ${opts.research.goalType}`,
      `sources_total: ${opts.sourceCount}`,
      `evidence_total: ${opts.evidenceCount}`,
      `cost_usd: ${opts.research.costUsd.toFixed(6)}`,
      `created_at: "${opts.now}"`,
      '---',
      '',
    ].join('\n');
  }

  // ── Research trail section ────────────────────────────────────────────────

  private buildResearchTrail(
    research: Research,
    plan?:    QueryPlan,
    gaps:     string[] = [],
  ): string {
    const lines = [
      '## Research Trail\n',
      `| Field | Value |`,
      `|---|---|`,
      `| Goal | ${research.goal} |`,
      `| Type | ${research.goalType} |`,
      `| Depth | ${research.depth} |`,
      `| Sources Fetched | ${research.totalSources} |`,
      `| Sources Used | ${research.usedSources} |`,
      `| Total Tokens | ${research.totalTokens.toLocaleString()} |`,
      `| Est. Cost | $${research.costUsd.toFixed(4)} |`,
      '',
    ];

    if (plan) {
      lines.push(
        `**Queries executed:** ${plan.queries.filter(q => q.executed).length} / ` +
        `${plan.queries.length} across ${plan.rounds} round(s)\n`,
      );

      // Per-source breakdown
      const bySource: Record<string, number> = {};
      for (const q of plan.queries.filter(q => q.executed)) {
        bySource[q.source] = (bySource[q.source] ?? 0) + q.resultCount;
      }
      lines.push('**Results by source:**\n');
      for (const [src, count] of Object.entries(bySource)) {
        lines.push(`- ${src}: ${count} results`);
      }
      lines.push('');
    }

    if (gaps.length > 0) {
      lines.push('**Knowledge gaps identified:**\n');
      for (const gap of gaps) {
        lines.push(`- ${gap}`);
      }
      lines.push('');
    }

    return lines.join('\n') + '\n';
  }

  // ── Source appendix ───────────────────────────────────────────────────────

  private buildSourceAppendix(sources: Source[]): string {
    if (sources.length === 0) return '';

    const lines = ['## Sources\n'];

    sources.forEach((s, i) => {
      const n   = i + 1;
      const ref = s.url
        ? `[${s.title}](${s.url})`
        : `**${s.title}**`;

      const type = s.type.toUpperCase();
      const score = (s.relevanceScore * 100).toFixed(0);

      lines.push(`**[S:${n}]** [${type}] ${ref} — relevance: ${score}%`);
    });

    return lines.join('\n') + '\n';
  }

  // ── Helpers ───────────────────────────────────────────────────────────────

  private buildTitle(goal: string): string {
    // Capitalise first letter, strip trailing punctuation
    const clean = goal.trim().replace(/[?!.]+$/, '');
    return clean.charAt(0).toUpperCase() + clean.slice(1);
  }

  goalToSlug(goal: string): string {
    return goal
      .toLowerCase()
      .replace(/[^a-z0-9\s-]/g, '')
      .replace(/\s+/g, '-')
      .replace(/-+/g, '-')
      .slice(0, 60)
      .replace(/^-+|-+$/g, '');
  }

  private extractSummary(synthesis: string): string | null {
    // Extract first paragraph that isn't a header
    const paragraphs = synthesis.split(/\n\n+/);
    for (const para of paragraphs) {
      if (!para.trim().startsWith('#') && para.trim().length > 50) {
        return para.trim().slice(0, 600);
      }
    }
    return null;
  }
}
src/report/WikiPusher.ts
TypeScript

import type { FormattedReport } from './ReportFormatter.js';
import type { Research } from '../types/research.js';

/**
 * WikiPusher: pushes a formatted research report to packages/wiki.
 *
 * Design decisions:
 * - Uses the same WikiStore.upsertPage interface established in Pass 5.
 * - Author is always 'autoresearch' for audit trail visibility.
 * - If a wiki page for this research already exists (same slug), it is
 *   updated rather than creating a duplicate.
 * - Returns the final slug so AutoResearchLoop can store it on Research.
 */
export class WikiPusher {
  constructor(
    private readonly wikiStore: {
      upsertPage: (opts: {
        slug:           string;
        title:          string;
        content:        string;
        tags:           string[];
        status:         string;
        author:         string;
        graphifyNode?:  string;
      }) => Promise<{ slug: string }>;
    } | null,  // null = wiki integration disabled
  ) {}

  /**
   * Push report to wiki. Returns slug if successful, null if disabled/failed.
   */
  async push(report: FormattedReport, research: Research): Promise<string | null> {
    if (!this.wikiStore) return null;

    const tags = [
      'autoresearch',
      research.goalType,
      research.depth,
      ...research.tags,
    ];

    try {
      const page = await this.wikiStore.upsertPage({
        slug:    report.slug,
        title:   report.title,
        content: report.full,
        tags,
        status:  'active',
        author:  'autoresearch',
      });

      return page.slug;
    } catch (err) {
      console.error('[WikiPusher] Failed to push report to wiki:', err);
      return null;
    }
  }
}
src/report/ReportWriter.ts
TypeScript

import { writeFile, mkdir } from 'node:fs/promises';
import { join, dirname } from 'node:path';
import type { Research } from '../types/research.js';
import type { Source, Evidence } from '../types/evidence.js';
import type { EvidenceGraph } from '../types/evidence.js';
import type { QueryPlan } from '../types/planning.js';
import type { SynthesisResult } from '../synthesis/SynthesisEngine.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import { ReportFormatter } from './ReportFormatter.js';
import { WikiPusher } from './WikiPusher.js';

/**
 * ReportWriter: orchestrates formatting + filesystem write + wiki push.
 *
 * Design decisions:
 * - Reports are written to .locoworker/research/<researchId>/<slug>.md
 *   (consistent with .locoworker/ runtime folder from Pass 1).
 * - Filesystem write is always attempted; wiki push is best-effort.
 * - Both reportPath and wikiSlug are written back to the Research record.
 */

export interface ReportResult {
  reportPath:  string;
  wikiSlug:    string | null;
  wordCount:   number;
  charCount:   number;
  title:       string;
}

export class ReportWriter {
  private formatter: ReportFormatter;
  private wikiPusher: WikiPusher;

  constructor(
    private readonly store:      ResearchStore,
    private readonly outputDir:  string,
    wikiStore: ConstructorParameters<typeof WikiPusher>[0],
  ) {
    this.formatter  = new ReportFormatter();
    this.wikiPusher = new WikiPusher(wikiStore);
  }

  /**
   * Write the research report to disk and optionally push to wiki.
   */
  async write(opts: {
    research:   Research;
    synthesis:  SynthesisResult;
    sources:    Source[];
    evidence:   Evidence[];
    graph:      EvidenceGraph;
    plan?:      QueryPlan;
    gaps?:      string[];
  }): Promise<ReportResult> {
    const { research, synthesis, sources, evidence, plan, gaps } = opts;

    // Format report
    const report = this.formatter.format({
      research,
      synthesis:  synthesis.text,
      sources:    sources.filter(s => synthesis.sourcesUsed?.find?.(u => u.id === s.id)),
      evidence,
      plan,
      gaps,
    });

    // Write to filesystem
    const reportPath = join(
      this.outputDir,
      research.id,
      `${report.slug}.md`,
    );

    await mkdir(dirname(reportPath), { recursive: true });
    await writeFile(reportPath, report.full, 'utf8');

    // Push to wiki (best-effort)
    const wikiSlug = await this.wikiPusher.push(report, research);

    // Update store
    this.store.updateStatus(research.id, 'complete', {
      completedAt: new Date().toISOString(),
      reportPath,
      wikiSlug:    wikiSlug ?? undefined,
      usedSources: synthesis.sourcesUsed?.length ?? 0,
    });

    return {
      reportPath,
      wikiSlug,
      wordCount: report.wordCount,
      charCount: report.charCount,
      title:     report.title,
    };
  }
}
src/loop/LoopSupervisor.ts
TypeScript

import type { Research } from '../types/research.js';
import { ResearchCostCapError } from '../types/errors.js';

/**
 * LoopSupervisor: enforces cost, time, and depth guards during the research loop.
 *
 * Design decisions:
 * - Cost cap is checked before each round starts (not mid-round).
 * - Time cap is wall-clock elapsed since loop started.
 * - Depth cap is just the rounds field from QueryPlan (already enforced by planner),
 *   but LoopSupervisor provides a runtime double-check.
 * - All limits are configurable; defaults are conservative.
 */

export interface SupervisorConfig {
  maxCostUsd:    number;    // default: 0.50
  maxMinutes:    number;    // default: 30
  maxRounds:     number;    // default: 3
  maxSources:    number;    // default: 60
}

const DEFAULTS: SupervisorConfig = {
  maxCostUsd: 0.50,
  maxMinutes: 30,
  maxRounds:  3,
  maxSources: 60,
};

export interface SupervisorCheckResult {
  allowed:   boolean;
  reason?:   string;
}

export class LoopSupervisor {
  private config:    SupervisorConfig;
  private startedAt: number;

  constructor(config: Partial<SupervisorConfig> = {}) {
    this.config    = { ...DEFAULTS, ...config };
    this.startedAt = Date.now();
  }

  reset(): void {
    this.startedAt = Date.now();
  }

  /**
   * Check if a new round is allowed to start.
   */
  checkBeforeRound(research: Research, round: number): SupervisorCheckResult {
    // Cost cap
    if (research.costUsd >= this.config.maxCostUsd) {
      throw new ResearchCostCapError(
        research.id,
        research.costUsd,
        this.config.maxCostUsd,
      );
    }

    // Time cap
    const elapsedMin = (Date.now() - this.startedAt) / 60_000;
    if (elapsedMin >= this.config.maxMinutes) {
      return {
        allowed: false,
        reason:  `Time cap reached: ${elapsedMin.toFixed(1)} min > ${this.config.maxMinutes} min`,
      };
    }

    // Depth cap
    if (round > this.config.maxRounds) {
      return {
        allowed: false,
        reason:  `Round cap reached: round ${round} > max ${this.config.maxRounds}`,
      };
    }

    // Source cap
    if (research.totalSources >= this.config.maxSources) {
      return {
        allowed: false,
        reason:  `Source cap reached: ${research.totalSources} >= ${this.config.maxSources}`,
      };
    }

    return { allowed: true };
  }

  elapsedSeconds(): number {
    return Math.floor((Date.now() - this.startedAt) / 1000);
  }
}
src/loop/AutoResearchLoop.ts
TypeScript

import type { ResearchStore } from '../store/ResearchStore.js';
import type { ResearchPlanner } from '../planning/ResearchPlanner.js';
import type { QueryEngine } from '../query/QueryEngine.js';
import type { EvidenceCollector } from '../evidence/EvidenceCollector.js';
import type { SynthesisEngine } from '../synthesis/SynthesisEngine.js';
import type { ReportWriter } from '../report/ReportWriter.js';
import { LoopSupervisor } from './LoopSupervisor.js';
import type { CreateResearch, Research } from '../types/research.js';
import type { EvidenceGraph } from '../types/evidence.js';
import {
  AutoResearchError,
  ResearchAlreadyRunningError,
} from '../types/errors.js';
import type { SupervisorConfig } from './LoopSupervisor.js';

/**
 * AutoResearchLoop: the top-level async-generator orchestrator.
 *
 * Design decisions:
 * - Async generator pattern (mirrors queryLoop from packages/core Pass 2):
 *   yields typed events that CLI, dashboard, and KairosIntegration can consume.
 * - All subsystems are injected (no singletons), enabling clean testing.
 * - The loop phases map 1:1 to ResearchStatus values:
 *     planning → querying (round N) → collecting → synthesizing → writing → complete
 * - Between rounds the supervisor checks cost/time/depth before proceeding.
 * - Gaps from QueryEngine feed ResearchPlanner.refineWithGaps for the next round.
 * - On any unrecoverable error, status is set to 'failed' and a 'loop:error'
 *   event is yielded (not thrown), so callers don't need try/catch.
 */

export type LoopEventType =
  | 'loop:started'
  | 'loop:planning'
  | 'loop:plan_ready'
  | 'loop:round_start'
  | 'loop:round_complete'
  | 'loop:collecting'
  | 'loop:collection_complete'
  | 'loop:synthesizing'
  | 'loop:synthesis_complete'
  | 'loop:writing'
  | 'loop:complete'
  | 'loop:error'
  | 'loop:cancelled'
  | 'loop:supervisor_stop';

export interface LoopEvent {
  type:        LoopEventType;
  researchId:  string;
  timestamp:   string;
  data:        Record<string, unknown>;
}

export interface AutoResearchLoopConfig {
  supervisor?:  Partial<SupervisorConfig>;
  outputDir:    string;    // for ReportWriter
}

export class AutoResearchLoop {
  constructor(
    private readonly store:       ResearchStore,
    private readonly planner:     ResearchPlanner,
    private readonly queryEngine: QueryEngine,
    private readonly collector:   EvidenceCollector,
    private readonly synthesizer: SynthesisEngine,
    private readonly reporter:    ReportWriter,
    private readonly config:      AutoResearchLoopConfig,
  ) {}

  /**
   * Run a research loop from start to finish.
   * Yields LoopEvent objects; completes when the report is written or an
   * unrecoverable error occurs.
   */
  async *run(
    input: CreateResearch & { goalType?: string },
  ): AsyncGenerator<LoopEvent, void, unknown> {
    // ── Create Research record ──────────────────────────────────────────────
    let research: Research;

    try {
      const parsed = await this.parseGoalType(input);
      research = this.store.createResearch({
        ...input,
        goalType: parsed,
      });
    } catch (err) {
      yield this.event('loop:error', 'pre-create', {
        error: (err as Error).message,
      });
      return;
    }

    const supervisor = new LoopSupervisor(this.config.supervisor);
    const researchId = research.id;

    yield this.event('loop:started', researchId, { goal: research.goal });

    try {
      // ── Phase 1: Planning ────────────────────────────────────────────────
      this.store.updateStatus(researchId, 'planning', {
        startedAt: new Date().toISOString(),
      });

      yield this.event('loop:planning', researchId, {
        goal: research.goal, depth: research.depth,
      });

      const { plan, parsedGoal, stats } = await this.planner.plan(
        researchId,
        research.goal,
        research.depth,
      );

      yield this.event('loop:plan_ready', researchId, {
        rounds:          plan.rounds,
        totalQueries:    stats.totalQueries,
        estimatedCostUsd: stats.estimatedCostUsd,
        queriesBySource: stats.queriesBySource,
      });

      // ── Phase 2: Query rounds ────────────────────────────────────────────
      this.store.updateStatus(researchId, 'querying');

      const allSources: ReturnType<ResearchStore['getSources']> = [];
      let   accumulatedGaps: string[] = [];

      for (let round = 1; round <= plan.rounds; round++) {
        // Supervisor check before each round
        const freshResearch = this.store.getResearchOrThrow(researchId);
        const check = supervisor.checkBeforeRound(freshResearch, round);

        if (!check.allowed) {
          yield this.event('loop:supervisor_stop', researchId, {
            round,
            reason: check.reason,
          });
          break;
        }

        yield this.event('loop:round_start', researchId, {
          round,
          totalRounds: plan.rounds,
        });

        // Refine plan with gaps from previous round (rounds 2+)
        if (round > 1 && accumulatedGaps.length > 0) {
          await this.planner.refineWithGaps(
            plan.id,
            researchId,
            round,
            accumulatedGaps,
          );
        }

        const roundResult = await this.queryEngine.executeRound(researchId, round);
        allSources.push(...roundResult.sources);
        accumulatedGaps = roundResult.gaps;

        yield this.event('loop:round_complete', researchId, {
          round,
          sourcesFound:  roundResult.sources.length,
          skipped:       roundResult.stats.skipped,
          gaps:          roundResult.gaps,
          statsTotal:    roundResult.stats.total,
          statsFailed:   roundResult.stats.failed,
        });
      }

      // ── Phase 3: Evidence collection ─────────────────────────────────────
      this.store.updateStatus(researchId, 'collecting');

      yield this.event('loop:collecting', researchId, {
        totalSources: allSources.length,
      });

      // Get all persisted sources (includes ones from all rounds)
      const persistedSources = this.store.getSources(researchId, {
        minRelevance: 0.3,
        limit:        100,
      });

      const { evidence, graph, dedup } = await this.collector.collect(
        researchId,
        persistedSources,
        plan.rounds,
      );

      yield this.event('loop:collection_complete', researchId, {
        evidenceCount: evidence.length,
        dedup,
        graphNodes:    graph.nodes.length,
        clusters:      graph.clusters.length,
      });

      // ── Phase 4: Synthesis ────────────────────────────────────────────────
      this.store.updateStatus(researchId, 'synthesizing');

      yield this.event('loop:synthesizing', researchId, {
        evidenceCount: evidence.length,
        sourcesCount:  persistedSources.length,
      });

      const synthesis = await this.synthesizer.synthesize(
        research,
        evidence,
        persistedSources,
        graph,
      );

      yield this.event('loop:synthesis_complete', researchId, {
        tokensUsed:      synthesis.tokensUsed,
        costUsd:         synthesis.costUsd,
        attempts:        synthesis.attempts,
        validationScore: synthesis.validationScore,
        isFallback:      synthesis.isFallback,
      });

      // ── Phase 5: Report writing ───────────────────────────────────────────
      this.store.updateStatus(researchId, 'writing');

      yield this.event('loop:writing', researchId, {});

      const currentPlan = this.store.getPlan(researchId);

      const reportResult = await this.reporter.write({
        research,
        synthesis,
        sources:  persistedSources,
        evidence,
        graph,
        plan:     currentPlan ?? undefined,
        gaps:     accumulatedGaps,
      });

      // ── Complete ─────────────────────────────────────────────────────────
      yield this.event('loop:complete', researchId, {
        reportPath:   reportResult.reportPath,
        wikiSlug:     reportResult.wikiSlug,
        wordCount:    reportResult.wordCount,
        elapsedSec:   supervisor.elapsedSeconds(),
        totalSources: persistedSources.length,
        evidenceCount: evidence.length,
      });

    } catch (err) {
      const message = (err as Error).message ?? 'Unknown error';

      this.store.updateStatus(researchId, 'failed', {
        errorMessage: message,
        completedAt:  new Date().toISOString(),
      });

      yield this.event('loop:error', researchId, {
        error: message,
        stack: (err as Error).stack,
      });
    }
  }

  // ── Helpers ───────────────────────────────────────────────────────────────

  private async parseGoalType(input: CreateResearch & { goalType?: string }): Promise<string> {
    // If caller provided goalType, use it; otherwise classify via planner
    if (input.goalType) return input.goalType;

    // Import lazily to avoid circular dep
    const { GoalParser } = await import('../planning/GoalParser.js');
    return new GoalParser().parse(input.goal).goalType;
  }

  private event(
    type:       LoopEventType,
    researchId: string,
    data:       Record<string, unknown>,
  ): LoopEvent {
    return {
      type,
      researchId,
      timestamp: new Date().toISOString(),
      data,
    };
  }
}
src/kairos/KairosIntegration.ts
TypeScript

import type { AutoResearchLoop } from '../loop/AutoResearchLoop.js';
import type { ResearchStore } from '../store/ResearchStore.js';

/**
 * KairosIntegration: registers the autoresearch job handler with Kairos
 * and emits structured observations to the ObservationLog.
 *
 * Design decisions:
 * - Follows the job-handler registration pattern defined in Pass 6
 *   (KairosAgent.registerJobHandler).
 * - Observations are appended for every major loop phase (start, rounds,
 *   complete/failed) so KairosAgent's TemporalContext stays informed.
 * - EventBus events are emitted for real-time WebSocket consumers
 *   (Gateway EventStreamer, Pass 8).
 * - Kairos runs AutoDream after a successful deep research to consolidate
 *   the new knowledge into memory.
 */

export interface KairosIntegrationDeps {
  loop:             AutoResearchLoop;
  store:            ResearchStore;
  kairosAgent: {
    registerJobHandler: (
      jobType:  string,
      handler:  (taskId: string, payload: Record<string, unknown>) => Promise<void>,
    ) => void;
  } | null;
  observationLog: {
    append: (entry: {
      type:     string;
      content:  string;
      metadata?: Record<string, unknown>;
    }) => Promise<void>;
  } | null;
  eventBus: {
    emit: (event: string, payload: Record<string, unknown>) => void;
  } | null;
  autoDream: {
    run: () => Promise<unknown>;
  } | null;
}

export class KairosIntegration {
  constructor(private readonly deps: KairosIntegrationDeps) {}

  /**
   * Register with KairosAgent.
   * After this call, Kairos can schedule 'autoresearch' jobs.
   */
  register(): void {
    if (!this.deps.kairosAgent) return;

    this.deps.kairosAgent.registerJobHandler(
      'autoresearch',
      async (taskId, payload) => {
        await this.runFromKairos(taskId, payload);
      },
    );
  }

  /**
   * Run a research triggered by Kairos.
   */
  private async runFromKairos(
    taskId:  string,
    payload: Record<string, unknown>,
  ): Promise<void> {
    const goal  = String(payload.goal ?? '');
    const depth = (payload.depth as any) ?? 'standard';
    const tags  = (payload.tags as string[]) ?? [];

    if (!goal) {
      console.warn('[KairosIntegration] Kairos triggered autoresearch with empty goal');
      return;
    }

    await this.deps.observationLog?.append({
      type:    'kairos_trigger',
      content: `AutoResearch triggered by Kairos task ${taskId}: "${goal}"`,
      metadata: { taskId, goal, depth },
    });

    let researchId: string | undefined;
    let success = false;

    // Consume the async generator
    for await (const event of this.deps.loop.run({
      goal,
      depth,
      tags,
      kairosTaskId: taskId,
    })) {
      // Extract researchId from first event
      if (!researchId) researchId = event.researchId;

      // Emit to EventBus for WebSocket consumers
      this.deps.eventBus?.emit(`autoresearch:${event.type}`, {
        ...event.data,
        researchId: event.researchId,
      });

      // Append key phase observations
      switch (event.type) {
        case 'loop:plan_ready':
          await this.deps.observationLog?.append({
            type:    'agent_insight',
            content: `Research plan ready for "${goal}": ` +
              `${event.data.totalQueries} queries, ` +
              `${event.data.rounds} rounds`,
            metadata: { researchId, ...event.data },
          });
          break;

        case 'loop:round_complete':
          await this.deps.observationLog?.append({
            type:    'agent_insight',
            content: `Research round ${event.data.round}/${event.data.rounds ?? '?'} ` +
              `complete for "${goal}": ${event.data.sourcesFound} sources found`,
            metadata: { researchId, ...event.data },
          });
          break;

        case 'loop:complete':
          success = true;
          await this.deps.observationLog?.append({
            type:    'agent_insight',
            content: `Research complete for "${goal}": ` +
              `${event.data.wordCount} words, ` +
              `${event.data.totalSources} sources, ` +
              `elapsed ${event.data.elapsedSec}s`,
            metadata: { researchId, ...event.data },
          });
          break;

        case 'loop:error':
          await this.deps.observationLog?.append({
            type:    'tool_error',
            content: `Research failed for "${goal}": ${event.data.error}`,
            metadata: { researchId, error: event.data.error },
          });
          break;
      }
    }

    // Trigger AutoDream after a successful deep research to consolidate knowledge
    if (success && depth === 'deep' && this.deps.autoDream) {
      try {
        await this.deps.autoDream.run();
        await this.deps.observationLog?.append({
          type:    'agent_insight',
          content: `AutoDream triggered after deep research: "${goal}"`,
          metadata: { researchId },
        });
      } catch (err) {
        console.warn('[KairosIntegration] AutoDream post-research failed:', err);
      }
    }
  }
}
src/tools/ResearchToolSet.ts
TypeScript

import type { AutoResearchLoop } from '../loop/AutoResearchLoop.js';
import type { ResearchStore } from '../store/ResearchStore.js';

/**
 * ResearchToolSet: exposes autoresearch capabilities as agent tools
 * compatible with packages/core ToolRegistry.
 *
 * Design decisions:
 * - All tools follow the ToolDefinition schema from Pass 2:
 *   { name, description, inputSchema (JSON Schema), parallelSafe,
 *     requiredPermission, execute }.
 * - research_run: streams loop events and returns final report path.
 *   Uses an async generator adapter that collects events synchronously
 *   (the tool interface is synchronous; streaming is via EventBus).
 * - research_status: read-only lookup, parallelSafe = true.
 * - research_list: read-only listing, parallelSafe = true.
 * - research_get_evidence: returns top-N evidence for in-context use.
 * - All write tools require 'STANDARD' permission tier (consistent with
 *   the permission ladder from Pass 2 PermissionGate).
 */

export interface ToolDefinition {
  name:               string;
  description:        string;
  inputSchema:        Record<string, unknown>;
  parallelSafe:       boolean;
  requiredPermission: string;
  execute:            (input: Record<string, unknown>) => Promise<unknown>;
}

export class ResearchToolSet {
  constructor(
    private readonly loop:  AutoResearchLoop,
    private readonly store: ResearchStore,
    private readonly eventBus?: {
      emit: (event: string, payload: Record<string, unknown>) => void;
    } | null,
  ) {}

  /**
   * Return all tool definitions for registration with ToolRegistry.
   */
  getTools(): ToolDefinition[] {
    return [
      this.researchRunTool(),
      this.researchStatusTool(),
      this.researchListTool(),
      this.researchGetEvidenceTool(),
      this.researchCancelTool(),
    ];
  }

  // ── research_run ──────────────────────────────────────────────────────────

  private researchRunTool(): ToolDefinition {
    return {
      name: 'research_run',
      description:
        'Start an autonomous research loop on a given goal. ' +
        'Returns the research ID and report path when complete. ' +
        'Use depth "surface" for quick lookups, "standard" for thorough research, ' +
        '"deep" for exhaustive cross-validated research.',
      inputSchema: {
        type: 'object',
        required: ['goal'],
        properties: {
          goal:  { type: 'string', description: 'The research question or topic.' },
          depth: {
            type:    'string',
            enum:    ['surface', 'standard', 'deep'],
            default: 'standard',
            description: 'Research depth.',
          },
          tags: {
            type:        'array',
            items:       { type: 'string' },
            description: 'Optional tags for the research.',
          },
        },
      },
      parallelSafe:       false,
      requiredPermission: 'STANDARD',
      execute: async (input) => {
        const goal  = String(input.goal ?? '');
        const depth = (input.depth as any) ?? 'standard';
        const tags  = (input.tags as string[]) ?? [];

        if (!goal) {
          throw new Error('research_run: goal is required');
        }

        // Run the loop and collect events
        const events: Record<string, unknown>[] = [];
        let lastEvent: Record<string, unknown> = {};

        for await (const event of this.loop.run({ goal, depth, tags })) {
          // Emit each event to EventBus for live streaming
          this.eventBus?.emit(`autoresearch:${event.type}`, {
            ...event.data,
            researchId: event.researchId,
          });

          events.push({ type: event.type, data: event.data });
          lastEvent = event;

          // Abort collection if research failed
          if (event.type === 'loop:error') {
            return {
              status:    'failed',
              error:     event.data.error,
              researchId: event.researchId,
            };
          }
        }

        // Return final summary
        const completeEvent = events.find(e => e.type === 'loop:complete');
        const data = (completeEvent?.data ?? {}) as Record<string, unknown>;

        return {
          status:      'complete',
          researchId:  lastEvent.researchId,
          reportPath:  data.reportPath,
          wikiSlug:    data.wikiSlug,
          wordCount:   data.wordCount,
          elapsedSec:  data.elapsedSec,
          totalSources: data.totalSources,
          evidenceCount: data.evidenceCount,
        };
      },
    };
  }

  // ── research_status ───────────────────────────────────────────────────────

  private researchStatusTool(): ToolDefinition {
    return {
      name:        'research_status',
      description: 'Get the current status of a research by ID.',
      inputSchema: {
        type:     'object',
        required: ['researchId'],
        properties: {
          researchId: { type: 'string' },
        },
      },
      parallelSafe:       true,
      requiredPermission: 'READ_ONLY',
      execute: async (input) => {
        const research = this.store.getResearch(String(input.researchId ?? ''));
        if (!research) {
          return { error: `Research not found: ${input.researchId}` };
        }

        return {
          id:           research.id,
          status:       research.status,
          goal:         research.goal,
          goalType:     research.goalType,
          depth:        research.depth,
          totalSources: research.totalSources,
          usedSources:  research.usedSources,
          costUsd:      research.costUsd,
          totalTokens:  research.totalTokens,
          wikiSlug:     research.wikiSlug,
          reportPath:   research.reportPath,
          createdAt:    research.createdAt,
          completedAt:  research.completedAt,
          errorMessage: research.errorMessage,
        };
      },
    };
  }

  // ── research_list ─────────────────────────────────────────────────────────

  private researchListTool(): ToolDefinition {
    return {
      name:        'research_list',
      description: 'List recent research items with optional status filter.',
      inputSchema: {
        type: 'object',
        properties: {
          status: {
            type: 'string',
            enum: [
              'pending', 'planning', 'querying', 'fetching',
              'collecting', 'synthesizing', 'writing',
              'complete', 'failed', 'cancelled',
            ],
          },
          limit:  { type: 'number', default: 10 },
        },
      },
      parallelSafe:       true,
      requiredPermission: 'READ_ONLY',
      execute: async (input) => {
        const items = this.store.listResearches({
          status: input.status as any,
          limit:  Number(input.limit ?? 10),
        });

        return {
          researches: items.map(r => ({
            id:          r.id,
            goal:        r.goal,
            goalType:    r.goalType,
            depth:       r.depth,
            status:      r.status,
            costUsd:     r.costUsd,
            wikiSlug:    r.wikiSlug,
            createdAt:   r.createdAt,
            completedAt: r.completedAt,
          })),
          total: items.length,
        };
      },
    };
  }

  // ── research_get_evidence ─────────────────────────────────────────────────

  private researchGetEvidenceTool(): ToolDefinition {
    return {
      name:        'research_get_evidence',
      description:
        'Retrieve top evidence items from a completed research for use in context. ' +
        'Returns ranked evidence with claims, types, and confidence scores.',
      inputSchema: {
        type:     'object',
        required: ['researchId'],
        properties: {
          researchId:  { type: 'string' },
          limit:       { type: 'number', default: 20 },
          minRelevance:{ type: 'number', default: 0.5 },
          type: {
            type: 'string',
            enum: [
              'fact', 'definition', 'example', 'comparison',
              'procedure', 'opinion', 'code_pattern', 'warning', 'contradiction',
            ],
          },
        },
      },
      parallelSafe:       true,
      requiredPermission: 'READ_ONLY',
      execute: async (input) => {
        const research = this.store.getResearch(String(input.researchId ?? ''));
        if (!research) {
          return { error: `Research not found: ${input.researchId}` };
        }

        const evidence = this.store.getEvidence(research.id, {
          type:         input.type as any,
          minRelevance: Number(input.minRelevance ?? 0.5),
          limit:        Number(input.limit ?? 20),
        });

        return {
          researchId: research.id,
          goal:       research.goal,
          evidence: evidence.map(e => ({
            id:         e.id,
            type:       e.type,
            claim:      e.claim,
            confidence: e.confidence,
            relevance:  e.relevance,
            round:      e.round,
            tags:       e.tags,
            contradicts: e.contradicts.length,
            supports:    e.supports.length,
          })),
          total: evidence.length,
        };
      },
    };
  }

  // ── research_cancel ───────────────────────────────────────────────────────

  private researchCancelTool(): ToolDefinition {
    return {
      name:        'research_cancel',
      description: 'Cancel a pending or running research.',
      inputSchema: {
        type:     'object',
        required: ['researchId'],
        properties: {
          researchId: { type: 'string' },
        },
      },
      parallelSafe:       false,
      requiredPermission: 'STANDARD',
      execute: async (input) => {
        const id = String(input.researchId ?? '');
        const research = this.store.getResearch(id);
        if (!research) {
          return { error: `Research not found: ${id}` };
        }

        if (['complete', 'failed', 'cancelled'].includes(research.status)) {
          return {
            error: `Cannot cancel research in status: ${research.status}`,
          };
        }

        this.store.updateStatus(id, 'cancelled', {
          completedAt: new Date().toISOString(),
        });

        return { message: 'Research cancelled', researchId: id };
      },
    };
  }
}
src/cli/commands.ts
TypeScript

import { parseArgs } from 'node:util';
import { join } from 'node:path';
import type { AutoResearchLoop } from '../loop/AutoResearchLoop.js';
import type { ResearchStore } from '../store/ResearchStore.js';
import type { ReportFormatter } from '../report/ReportFormatter.js';

/**
 * CLI commands for autoresearch.
 *
 * Usage (from root package.json scripts):
 *   pnpm research:run  -- --goal "How does X work?" --depth standard
 *   pnpm research:report -- --id <researchId>
 *
 * Design decisions:
 * - Uses Node's built-in parseArgs (no external CLI framework dep).
 * - Streams loop events to stdout for real-time feedback.
 * - Exits with code 1 on error (standard CLI convention).
 * - Output is human-readable (coloured where possible) not JSON.
 */

// ── ANSI colour helpers ───────────────────────────────────────────────────────

const isTTY = process.stdout.isTTY;
const c = {
  reset:  isTTY ? '\x1b[0m'  : '',
  bold:   isTTY ? '\x1b[1m'  : '',
  dim:    isTTY ? '\x1b[2m'  : '',
  green:  isTTY ? '\x1b[32m' : '',
  yellow: isTTY ? '\x1b[33m' : '',
  cyan:   isTTY ? '\x1b[36m' : '',
  red:    isTTY ? '\x1b[31m' : '',
  blue:   isTTY ? '\x1b[34m' : '',
};

function log(msg: string): void { process.stdout.write(msg + '\n'); }
function err(msg: string): void { process.stderr.write(msg + '\n'); }

function pad(s: string, width: number): string {
  return s.padEnd(width).slice(0, width);
}

// ── research:run ─────────────────────────────────────────────────────────────

export async function researchRunCommand(
  loop:  AutoResearchLoop,
  argv:  string[] = process.argv.slice(2),
): Promise<void> {
  const { values } = parseArgs({
    args: argv,
    options: {
      goal:  { type: 'string', short: 'g' },
      depth: { type: 'string', short: 'd', default: 'standard' },
      tags:  { type: 'string', short: 't', multiple: true, default: [] },
      quiet: { type: 'boolean', short: 'q', default: false },
    },
    allowPositionals: false,
  });

  if (!values.goal) {
    err(`${c.red}Error: --goal is required${c.reset}`);
    err(`Usage: research:run --goal "your research question" [--depth surface|standard|deep]`);
    process.exit(1);
  }

  const depth = (values.depth as any) ?? 'standard';
  const tags  = (values.tags as string[]) ?? [];

  if (!values.quiet) {
    log(`\n${c.bold}${c.cyan}🔬 AutoResearch${c.reset}`);
    log(`${c.dim}Goal:${c.reset}  ${values.goal}`);
    log(`${c.dim}Depth:${c.reset} ${depth}`);
    log('');
  }

  let exitCode = 0;

  for await (const event of loop.run({ goal: values.goal!, depth, tags })) {
    if (values.quiet) continue;

    switch (event.type) {
      case 'loop:started':
        log(`${c.blue}▶${c.reset} Research started (${event.researchId.slice(0, 8)}…)`);
        break;

      case 'loop:plan_ready':
        log(
          `${c.blue}📋${c.reset} Plan ready — ` +
          `${event.data.totalQueries} queries, ` +
          `${event.data.rounds} round(s), ` +
          `est. cost $${Number(event.data.estimatedCostUsd).toFixed(4)}`,
        );
        break;

      case 'loop:round_start':
        log(`\n${c.yellow}⟳${c.reset} Round ${event.data.round}/${event.data.totalRounds} starting…`);
        break;

      case 'loop:round_complete': {
        const gaps = (event.data.gaps as string[]) ?? [];
        log(
          `${c.green}✓${c.reset} Round ${event.data.round} complete — ` +
          `${event.data.sourcesFound} sources, ` +
          `${event.data.skipped} skipped` +
          (gaps.length > 0 ? `, ${gaps.length} gaps identified` : ''),
        );
        break;
      }

      case 'loop:collecting':
        log(`${c.blue}🗂️${c.reset}  Collecting evidence from ${event.data.totalSources} sources…`);
        break;

      case 'loop:collection_complete':
        log(
          `${c.green}✓${c.reset} Evidence collected — ` +
          `${event.data.evidenceCount} items, ` +
          `${event.data.clusters} clusters, ` +
          `${event.data.dedup?.duplicates ?? 0} duplicates removed`,
        );
        break;

      case 'loop:synthesizing':
        log(`${c.blue}🧠${c.reset} Synthesizing ${event.data.evidenceCount} evidence items…`);
        break;

      case 'loop:synthesis_complete':
        log(
          `${c.green}✓${c.reset} Synthesis complete — ` +
          `${event.data.tokensUsed} tokens, ` +
          `$${Number(event.data.costUsd).toFixed(4)}, ` +
          `score: ${Number(event.data.validationScore).toFixed(2)}` +
          (event.data.isFallback ? ` ${c.yellow}(fallback)${c.reset}` : ''),
        );
        break;

      case 'loop:writing':
        log(`${c.blue}✍️${c.reset}  Writing report…`);
        break;

      case 'loop:complete':
        log(`\n${c.green}${c.bold}✅ Research complete!${c.reset}`);
        log(`${c.dim}Report:${c.reset}   ${event.data.reportPath}`);
        if (event.data.wikiSlug) {
          log(`${c.dim}Wiki:${c.reset}     [[${event.data.wikiSlug}]]`);
        }
        log(
          `${c.dim}Summary:${c.reset}  ` +
          `${event.data.wordCount} words, ` +
          `${event.data.totalSources} sources, ` +
          `${event.data.elapsedSec}s elapsed`,
        );
        log('');
        break;

      case 'loop:supervisor_stop':
        log(`\n${c.yellow}⚠️  Supervisor stopped: ${event.data.reason}${c.reset}`);
        break;

      case 'loop:error':
        log(`\n${c.red}${c.bold}✗ Research failed:${c.reset} ${event.data.error}`);
        exitCode = 1;
        break;

      case 'loop:cancelled':
        log(`\n${c.yellow}Research cancelled.${c.reset}`);
        exitCode = 1;
        break;
    }
  }

  process.exit(exitCode);
}

// ── research:report ───────────────────────────────────────────────────────────

export async function researchReportCommand(
  store:     ResearchStore,
  formatter: ReportFormatter,
  argv:      string[] = process.argv.slice(2),
): Promise<void> {
  const { values } = parseArgs({
    args: argv,
    options: {
      id:     { type: 'string', short: 'i' },
      list:   { type: 'boolean', short: 'l', default: false },
      limit:  { type: 'string', default: '10' },
      status: { type: 'string' },
      format: { type: 'string', default: 'table' },  // table | json
    },
    allowPositionals: false,
  });

  // List mode
  if (values.list) {
    const items = store.listResearches({
      status: values.status as any,
      limit:  parseInt(values.limit!, 10),
    });

    if (values.format === 'json') {
      log(JSON.stringify(items, null, 2));
    } else {
      log(`\n${c.bold}Recent Researches${c.reset}\n`);
      log(
        `${c.dim}${pad('ID', 10)} ${pad('STATUS', 12)} ${pad('DEPTH', 8)} ` +
        `${pad('SOURCES', 8)} ${pad('COST', 8)} GOAL${c.reset}`,
      );
      log('─'.repeat(80));
      for (const r of items) {
        const statusColor = r.status === 'complete'
          ? c.green : r.status === 'failed' ? c.red : c.yellow;
        log(
          `${pad(r.id.slice(0, 8), 10)} ` +
          `${statusColor}${pad(r.status, 12)}${c.reset} ` +
          `${pad(r.depth, 8)} ` +
          `${pad(String(r.totalSources), 8)} ` +
          `${pad('$' + r.costUsd.toFixed(3), 8)} ` +
          `${r.goal.slice(0, 40)}`,
        );
      }
      log('');
    }
    return;
  }

  // Single research report
  if (!values.id) {
    err(`${c.red}Error: --id or --list is required${c.reset}`);
    process.exit(1);
  }

  const research = store.getResearch(values.id);
  if (!research) {
    err(`${c.red}Research not found: ${values.id}${c.reset}`);
    process.exit(1);
  }

  if (values.format === 'json') {
    log(JSON.stringify(research, null, 2));
    return;
  }

  // Human-readable summary
  log(`\n${c.bold}Research Report${c.reset}\n`);
  log(`${c.dim}ID:${c.reset}          ${research.id}`);
  log(`${c.dim}Goal:${c.reset}        ${research.goal}`);
  log(`${c.dim}Type:${c.reset}        ${research.goalType}`);
  log(`${c.dim}Depth:${c.reset}       ${research.depth}`);

  const statusColor = research.status === 'complete'
    ? c.green : research.status === 'failed' ? c.red : c.yellow;
  log(`${c.dim}Status:${c.reset}      ${statusColor}${research.status}${c.reset}`);
  log(`${c.dim}Sources:${c.reset}     ${research.totalSources} fetched, ${research.usedSources} used`);
  log(`${c.dim}Tokens:${c.reset}      ${research.totalTokens.toLocaleString()}`);
  log(`${c.dim}Cost:${c.reset}        $${research.costUsd.toFixed(6)}`);
  log(`${c.dim}Created:${c.reset}     ${research.createdAt}`);

  if (research.completedAt) {
    log(`${c.dim}Completed:${c.reset}   ${research.completedAt}`);
  }

  if (research.wikiSlug) {
    log(`${c.dim}Wiki:${c.reset}        [[${research.wikiSlug}]]`);
  }

  if (research.reportPath) {
    log(`${c.dim}Report path:${c.reset} ${research.reportPath}`);
  }

  if (research.errorMessage) {
    log(`${c.dim}Error:${c.reset}       ${c.red}${research.errorMessage}${c.reset}`);
  }

  // Evidence summary
  const evidence = store.getEvidence(research.id, { limit: 5 });
  if (evidence.length > 0) {
    log(`\n${c.bold}Top Evidence${c.reset}\n`);
    for (const ev of evidence) {
      log(
        `${c.dim}[${ev.type.toUpperCase()}]${c.reset} ` +
        `(rel: ${ev.relevance.toFixed(2)}) ` +
        ev.claim.slice(0, 100),
      );
    }
  }

  log('');
}
src/index.ts (updated — full package exports)
TypeScript

/**
 * @locoworker/autoresearch — complete package exports (Parts 1 + 2)
 */

// Types
export * from './types/index.js';

// Store
export { ResearchStore } from './store/ResearchStore.js';

// Planning
export { GoalParser }        from './planning/GoalParser.js';
export { ResearchPlanner }   from './planning/ResearchPlanner.js';
export { QUERY_TEMPLATES }   from './planning/QueryTemplates.js';

// Query adapters
export { WebSearchAdapter }    from './query/WebSearchAdapter.js';
export { CodeSearchAdapter }   from './query/CodeSearchAdapter.js';
export { WikiSearchAdapter }   from './query/WikiSearchAdapter.js';
export { MemorySearchAdapter } from './query/MemorySearchAdapter.js';
export { QueryEngine }         from './query/QueryEngine.js';

// Evidence pipeline
export { ContentExtractor }  from './evidence/ContentExtractor.js';
export { SourceFetcher }     from './evidence/SourceFetcher.js';
export { EvidenceCollector } from './evidence/EvidenceCollector.js';

// Synthesis
export { PromptBuilder }       from './synthesis/PromptBuilder.js';
export { SynthesisValidator }  from './synthesis/SynthesisValidator.js';
export { SynthesisEngine }     from './synthesis/SynthesisEngine.js';
export type { SynthesisResult } from './synthesis/SynthesisEngine.js';

// Report
export { ReportFormatter } from './report/ReportFormatter.js';
export { WikiPusher }      from './report/WikiPusher.js';
export { ReportWriter }    from './report/ReportWriter.js';
export type { FormattedReport } from './report/ReportFormatter.js';
export type { ReportResult }    from './report/ReportWriter.js';

// Loop
export { LoopSupervisor }     from './loop/LoopSupervisor.js';
export { AutoResearchLoop }   from './loop/AutoResearchLoop.js';
export type {
  LoopEvent,
  LoopEventType,
  AutoResearchLoopConfig,
} from './loop/AutoResearchLoop.js';

// Kairos
export { KairosIntegration } from './kairos/KairosIntegration.js';

// Tools
export { ResearchToolSet } from './tools/ResearchToolSet.js';

// CLI
export {
  researchRunCommand,
  researchReportCommand,
} from './cli/commands.js';
tests/synthesis.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore }     from '../src/store/ResearchStore.js';
import { PromptBuilder }     from '../src/synthesis/PromptBuilder.js';
import { SynthesisValidator } from '../src/synthesis/SynthesisValidator.js';
import { SynthesisEngine }   from '../src/synthesis/SynthesisEngine.js';
import type { Evidence, Source, EvidenceGraph } from '../src/types/evidence.js';
import type { Research } from '../src/types/research.js';

// ── Fixtures ──────────────────────────────────────────────────────────────────

function makeResearch(overrides: Partial<Research> = {}): Research {
  return {
    id:           'res-1',
    goal:         'How does React reconciliation work?',
    goalType:     'technical',
    depth:        'standard',
    status:       'synthesizing',
    tags:         ['react'],
    totalSources: 5,
    usedSources:  5,
    totalTokens:  1000,
    costUsd:      0.003,
    createdAt:    new Date().toISOString(),
    ...overrides,
  };
}

function makeEvidence(id: string, claim: string): Evidence {
  return {
    id,
    researchId:  'res-1',
    sourceIds:   [`src-${id}`],
    type:        'fact',
    claim,
    context:     `Context for ${id}`,
    tags:        ['react'],
    confidence:  0.8,
    relevance:   0.85,
    round:       1,
    contradicts: [],
    supports:    [],
    createdAt:   new Date().toISOString(),
  };
}

function makeSource(id: string): Source {
  return {
    id,
    researchId:     'res-1',
    type:           'web',
    url:            `https://example.com/${id}`,
    title:          `Source ${id}`,
    rawContent:     '',
    cleanContent:   `Content for source ${id}`,
    fingerprint:    `fp-${id}`,
    tokenCount:     100,
    relevanceScore: 0.8,
    fetchedAt:      new Date().toISOString(),
  };
}

function makeGraph(): EvidenceGraph {
  return {
    researchId: 'res-1',
    nodes:      ['ev-1', 'ev-2'],
    supports:   [],
    contradicts: [],
    clusters:   [{ label: 'react', nodeIds: ['ev-1', 'ev-2'] }],
  };
}

// ── PromptBuilder tests ───────────────────────────────────────────────────────

describe('PromptBuilder', () => {
  const builder = new PromptBuilder();

  test('builds prompt within token budget', () => {
    const evidence = Array.from({ length: 20 }, (_, i) =>
      makeEvidence(`ev-${i}`, `Claim number ${i} about React reconciliation algorithm performance.`),
    );
    const sources = Array.from({ length: 20 }, (_, i) => makeSource(`src-ev-${i}`));

    const prompt = builder.build(
      makeResearch(),
      evidence,
      sources,
      makeGraph(),
      { tokenBudget: 4096 },
    );

    expect(prompt.tokenEstimate).toBeLessThanOrEqual(4096 + 100); // small tolerance
    expect(prompt.system).toContain('LocoWorker');
    expect(prompt.user).toContain('Evidence');
    expect(prompt.user).toContain('[S:1]');
  });

  test('surfaces contradictions in prompt', () => {
    const evidence = [
      makeEvidence('ev-a', 'React is fast because of virtual DOM diffing'),
      makeEvidence('ev-b', 'React is slow because of virtual DOM diffing overhead'),
    ];
    const sources  = [makeSource('src-ev-a'), makeSource('src-ev-b')];
    const graph: EvidenceGraph = {
      researchId:  'res-1',
      nodes:       ['ev-a', 'ev-b'],
      supports:    [],
      contradicts: [{ from: 'ev-a', to: 'ev-b', weight: 0.8 }],
      clusters:    [],
    };

    const prompt = builder.build(makeResearch(), evidence, sources, graph, {
      tokenBudget: 4096,
    });

    expect(prompt.user).toContain('Contradictions');
  });

  test('comparative goal type includes comparison instruction', () => {
    const prompt = builder.build(
      makeResearch({ goalType: 'comparative', goal: 'React vs Vue' }),
      [makeEvidence('ev-1', 'React uses JSX; Vue uses templates for component definition')],
      [makeSource('src-ev-1')],
      makeGraph(),
      { tokenBudget: 4096 },
    );

    expect(prompt.system).toContain('comparison');
  });

  test('empty evidence builds minimal prompt', () => {
    const prompt = builder.build(
      makeResearch(),
      [],
      [],
      { researchId: 'res-1', nodes: [], supports: [], contradicts: [], clusters: [] },
      { tokenBudget: 4096 },
    );

    expect(prompt.user).toBeTruthy();
    expect(prompt.evidenceUsed).toHaveLength(0);
  });
});

// ── SynthesisValidator tests ──────────────────────────────────────────────────

describe('SynthesisValidator', () => {
  const validator = new SynthesisValidator();

  test('valid synthesis passes', () => {
    const synthesis = [
      '## React Reconciliation\n',
      'React uses a virtual DOM to efficiently update the UI [S:1]. ',
      'The reconciler compares old and new trees using a diffing algorithm [S:2].\n',
      '## How It Works\n',
      'When state changes, React re-renders the component tree [S:1]. ',
      'It then commits only the changed parts to the real DOM [S:3].',
    ].join('');

    const result = validator.validate(synthesis, makeResearch());
    expect(result.valid).toBe(true);
    expect(result.score).toBeGreaterThan(0.8);
  });

  test('empty synthesis fails', () => {
    const result = validator.validate('', makeResearch());
    expect(result.valid).toBe(false);
    expect(result.reasons).toContain('Empty synthesis output');
  });

  test('refusal pattern fails', () => {
    const result = validator.validate(
      'I cannot help with this research request.',
      makeResearch(),
    );
    expect(result.valid).toBe(false);
    expect(result.reasons.some(r => r.includes('refused'))).toBe(true);
  });

  test('too-short synthesis fails', () => {
    const result = validator.validate('Short response.', makeResearch());
    expect(result.valid).toBe(false);
  });

  test('missing citations produces warning not failure', () => {
    const synthesis =
      '## React Overview\n\n' +
      'React is a JavaScript library for building user interfaces. ' +
      'It was created by Facebook and has been widely adopted in the industry. ' +
      'React uses components as building blocks for UI development.';

    const result = validator.validate(synthesis, makeResearch());
    expect(result.valid).toBe(true); // still valid — citations are a warning
    expect(result.warnings.some(w => w.includes('citation'))).toBe(true);
  });
});

// ── SynthesisEngine tests ─────────────────────────────────────────────────────

describe('SynthesisEngine', () => {
  let tempDir: string;
  let store:   ResearchStore;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'synthesis-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  test('returns synthesis from provider', async () => {
    const mockProvider = {
      call: async () => ({
        content: [
          '## React Reconciliation\n\n',
          'React uses a virtual DOM for efficient updates [S:1]. ',
          'The diffing algorithm compares trees recursively [S:2].\n\n',
          '## Conclusion\n\nReact reconciliation is efficient and well-documented.',
        ].join(''),
        usage: { inputTokens: 500, outputTokens: 200, costUsd: 0.002 },
      }),
    };

    const engine = new SynthesisEngine(store, mockProvider);

    const r = store.createResearch({
      goal:     'How does React reconciliation work?',
      goalType: 'technical',
      depth:    'standard',
      tags:     [],
    });

    const result = await engine.synthesize(
      store.getResearchOrThrow(r.id),
      [makeEvidence('ev-1', 'React uses virtual DOM')],
      [makeSource('src-ev-1')],
      makeGraph(),
    );

    expect(result.text).toContain('React');
    expect(result.tokensUsed).toBe(700);
    expect(result.isFallback).toBe(false);
    expect(result.attempts).toBe(1);
  });

  test('returns fallback when evidence is empty', async () => {
    const mockProvider = {
      call: async () => ({ content: '' }),
    };

    const engine = new SynthesisEngine(store, mockProvider);

    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const result = await engine.synthesize(
      store.getResearchOrThrow(r.id),
      [],
      [],
      { researchId: r.id, nodes: [], supports: [], contradicts: [], clusters: [] },
    );

    expect(result.isFallback).toBe(true);
  });

  test('accumulates cost in store', async () => {
    const mockProvider = {
      call: async () => ({
        content: '## Finding\n\nSome content with [S:1] citation.\nMore content here.',
        usage: { inputTokens: 300, outputTokens: 100, costUsd: 0.0012 },
      }),
    };

    const engine = new SynthesisEngine(store, mockProvider);

    const r = store.createResearch({
      goal:     'Test cost tracking',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    await engine.synthesize(
      store.getResearchOrThrow(r.id),
      [makeEvidence('ev-1', 'Some relevant fact about the topic here')],
      [makeSource('src-ev-1')],
      makeGraph(),
    );

    const updated = store.getResearchOrThrow(r.id);
    expect(updated.totalTokens).toBe(400);
    expect(updated.costUsd).toBeCloseTo(0.0012);
  });

  test('retries on invalid response and returns best attempt', async () => {
    let callCount = 0;

    const mockProvider = {
      call: async () => {
        callCount++;
        // Always return invalid (too short) response
        return {
          content: 'Short.',
          usage: { inputTokens: 100, outputTokens: 10, costUsd: 0.0001 },
        };
      },
    };

    const engine = new SynthesisEngine(store, mockProvider);

    const r = store.createResearch({
      goal:     'Test retry',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const result = await engine.synthesize(
      store.getResearchOrThrow(r.id),
      [makeEvidence('ev-1', 'Test claim here')],
      [makeSource('src-ev-1')],
      makeGraph(),
    );

    // Should have retried MAX_RETRIES + 1 times
    expect(callCount).toBeGreaterThan(1);
    // Still returns something (last attempt)
    expect(result.text).toBeTruthy();
  });
});
tests/report.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync, existsSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore }    from '../src/store/ResearchStore.js';
import { ReportFormatter }  from '../src/report/ReportFormatter.js';
import { ReportWriter }     from '../src/report/ReportWriter.js';
import type { Research }    from '../src/types/research.js';
import type { Source, Evidence, EvidenceGraph } from '../src/types/evidence.js';

// ── Fixtures ──────────────────────────────────────────────────────────────────

function makeResearch(): Research {
  return {
    id:           'res-1',
    goal:         'How does SQLite WAL mode work?',
    goalType:     'technical',
    depth:        'standard',
    status:       'writing',
    tags:         ['sqlite', 'wal'],
    totalSources: 3,
    usedSources:  3,
    totalTokens:  800,
    costUsd:      0.0024,
    createdAt:    new Date().toISOString(),
  };
}

function makeSources(): Source[] {
  return [
    {
      id: 's1', researchId: 'res-1', type: 'web',
      url: 'https://sqlite.org/wal.html', title: 'SQLite WAL',
      rawContent: '', cleanContent: 'WAL mode uses a separate log file',
      fingerprint: 'fp1', tokenCount: 50, relevanceScore: 0.9,
      fetchedAt: new Date().toISOString(), httpStatus: 200,
    },
    {
      id: 's2', researchId: 'res-1', type: 'wiki',
      wikiSlug: 'sqlite-wal', title: 'SQLite WAL — Project Wiki',
      rawContent: '', cleanContent: 'WAL provides better concurrency',
      fingerprint: 'fp2', tokenCount: 40, relevanceScore: 0.85,
      fetchedAt: new Date().toISOString(),
    },
  ];
}

function makeEvidence(): Evidence[] {
  return [
    {
      id: 'ev1', researchId: 'res-1', sourceIds: ['s1'],
      type: 'fact',
      claim: 'SQLite WAL mode uses a write-ahead log for better concurrency',
      context: 'From SQLite docs',
      tags: ['sqlite', 'wal'], confidence: 0.9, relevance: 0.9,
      round: 1, contradicts: [], supports: [],
      createdAt: new Date().toISOString(),
    },
    {
      id: 'ev2', researchId: 'res-1', sourceIds: ['s2'],
      type: 'warning',
      claim: 'Warning: WAL mode does not work well with network filesystems',
      context: 'From wiki',
      tags: ['sqlite', 'warning'], confidence: 0.85, relevance: 0.8,
      round: 1, contradicts: [], supports: [],
      createdAt: new Date().toISOString(),
    },
  ];
}

function makeGraph(): EvidenceGraph {
  return {
    researchId: 'res-1',
    nodes:      ['ev1', 'ev2'],
    supports:   [],
    contradicts: [],
    clusters:   [{ label: 'sqlite', nodeIds: ['ev1', 'ev2'] }],
  };
}

// ── ReportFormatter tests ─────────────────────────────────────────────────────

describe('ReportFormatter', () => {
  const formatter = new ReportFormatter();

  test('generates YAML frontmatter', () => {
    const report = formatter.format({
      research:  makeResearch(),
      synthesis: '## Findings\n\nSQLite WAL mode uses a write-ahead log [S:1].',
      sources:   makeSources(),
      evidence:  makeEvidence(),
    });

    expect(report.frontmatter).toContain('---');
    expect(report.frontmatter).toContain('title:');
    expect(report.frontmatter).toContain('autoresearch');
    expect(report.frontmatter).toContain('research_id:');
    expect(report.frontmatter).toContain('cost_usd:');
  });

  test('includes synthesis in body', () => {
    const synthesis = '## Findings\n\nSQLite WAL mode uses a write-ahead log [S:1].';
    const report    = formatter.format({
      research: makeResearch(), synthesis, sources: makeSources(), evidence: makeEvidence(),
    });

    expect(report.body).toContain('Research Findings');
    expect(report.body).toContain('WAL mode');
  });

  test('includes source appendix', () => {
    const report = formatter.format({
      research:  makeResearch(),
      synthesis: '## Findings\n\nContent [S:1].',
      sources:   makeSources(),
      evidence:  makeEvidence(),
    });

    expect(report.body).toContain('[S:1]');
    expect(report.body).toContain('SQLite WAL');
    expect(report.body).toContain('sqlite.org');
  });

  test('includes conflicting evidence callout', () => {
    const ev = makeEvidence();
    ev.push({
      id: 'ev3', researchId: 'res-1', sourceIds: ['s1'],
      type: 'contradiction',
      claim: 'Some sources claim WAL has worse read performance',
      context: '', tags: [], confidence: 0.6, relevance: 0.6,
      round: 2, contradicts: ['ev1'], supports: [],
      createdAt: new Date().toISOString(),
    });

    const report = formatter.format({
      research:  makeResearch(),
      synthesis: '## Findings\n\nContent [S:1].',
      sources:   makeSources(),
      evidence:  ev,
    });

    expect(report.body).toContain('Conflicting Evidence');
  });

  test('generates correct slug', () => {
    const report = formatter.format({
      research:  makeResearch(),
      synthesis: '## Test\n\nContent.',
      sources:   [],
      evidence:  [],
    });

    expect(report.slug).toMatch(/^[a-z0-9-]+$/);
    expect(report.slug).toContain('sqlite');
  });

  test('reports word count', () => {
    const synthesis = 'Word '.repeat(200);
    const report    = formatter.format({
      research:  makeResearch(),
      synthesis,
      sources:   makeSources(),
      evidence:  makeEvidence(),
    });

    expect(report.wordCount).toBeGreaterThan(150);
  });

  test('full report combines frontmatter + body', () => {
    const report = formatter.format({
      research:  makeResearch(),
      synthesis: '## Test\n\nContent [S:1].',
      sources:   makeSources(),
      evidence:  makeEvidence(),
    });

    expect(report.full).toContain('---');      // frontmatter
    expect(report.full).toContain('##');       // body headers
    expect(report.full).toContain('[S:1]');    // citation
  });
});

// ── ReportWriter tests ────────────────────────────────────────────────────────

describe('ReportWriter', () => {
  let tempDir:  string;
  let store:    ResearchStore;
  let writer:   ReportWriter;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'report-writer-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
    writer  = new ReportWriter(store, join(tempDir, 'reports'), null);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  test('writes report to filesystem', async () => {
    const r = store.createResearch({
      goal:     'How does SQLite WAL work?',
      goalType: 'technical',
      depth:    'standard',
      tags:     ['sqlite'],
    });

    store.updateStatus(r.id, 'writing');

    const result = await writer.write({
      research: store.getResearchOrThrow(r.id),
      synthesis: {
        text:            '## Findings\n\nSQLite WAL uses a log file [S:1].',
        tokensUsed:      300,
        costUsd:         0.0009,
        attempts:        1,
        validationScore: 0.9,
        isFallback:      false,
        sourcesUsed:     makeSources(),
      },
      sources:   makeSources(),
      evidence:  makeEvidence(),
      graph:     makeGraph(),
    });

    expect(existsSync(result.reportPath)).toBe(true);
    expect(result.reportPath).toContain('.md');
    expect(result.wordCount).toBeGreaterThan(0);
    expect(result.wikiSlug).toBeNull(); // no wiki store
  });

  test('updates research status to complete after write', async () => {
    const r = store.createResearch({
      goal:     'Test report writing',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.updateStatus(r.id, 'writing');

    await writer.write({
      research: store.getResearchOrThrow(r.id),
      synthesis: {
        text:            '## Result\n\nThis is the research result [S:1].',
        tokensUsed:      100,
        costUsd:         0.0003,
        attempts:        1,
        validationScore: 1.0,
        isFallback:      false,
        sourcesUsed:     [],
      },
      sources:  [],
      evidence: [],
      graph:    { researchId: r.id, nodes: [], supports: [], contradicts: [], clusters: [] },
    });

    const updated = store.getResearchOrThrow(r.id);
    expect(updated.status).toBe('complete');
    expect(updated.completedAt).toBeTruthy();
    expect(updated.reportPath).toBeTruthy();
  });

  test('pushes to wiki when wikiStore provided', async () => {
    let pushedSlug: string | null = null;

    const mockWikiStore = {
      upsertPage: async (opts: any) => {
        pushedSlug = opts.slug;
        return { slug: opts.slug };
      },
    };

    const wikiWriter = new ReportWriter(
      store,
      join(tempDir, 'reports2'),
      mockWikiStore,
    );

    const r = store.createResearch({
      goal:     'Test wiki push',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.updateStatus(r.id, 'writing');

    const result = await wikiWriter.write({
      research: store.getResearchOrThrow(r.id),
      synthesis: {
        text:            '## Result\n\nThis is a test result [S:1].',
        tokensUsed:      50,
        costUsd:         0.0001,
        attempts:        1,
        validationScore: 1.0,
        isFallback:      false,
        sourcesUsed:     [],
      },
      sources:  [],
      evidence: [],
      graph:    { researchId: r.id, nodes: [], supports: [], contradicts: [], clusters: [] },
    });

    expect(result.wikiSlug).toBeTruthy();
    expect(pushedSlug).toBeTruthy();
  });
});
tests/loop.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore }   from '../src/store/ResearchStore.js';
import { ResearchPlanner } from '../src/planning/ResearchPlanner.js';
import { LoopSupervisor }  from '../src/loop/LoopSupervisor.js';
import { AutoResearchLoop, type LoopEvent } from '../src/loop/AutoResearchLoop.js';

// ── Stub factories ────────────────────────────────────────────────────────────

function makeStubQueryEngine() {
  return {
    executeRound: async (_researchId: string, _round: number) => ({
      sources: [
        {
          id: `src-${Math.random()}`, researchId: _researchId, type: 'web' as const,
          url: 'https://example.com', title: 'Test', rawContent: '',
          cleanContent: 'React uses virtual DOM for diffing.',
          fingerprint: Math.random().toString(36), tokenCount: 20,
          relevanceScore: 0.8, fetchedAt: new Date().toISOString(),
        },
      ],
      gaps:  ['What about concurrent mode?'],
      stats: { total: 1, succeeded: 1, failed: 0, skipped: 0 },
    }),
  };
}

function makeStubCollector() {
  return {
    collect: async (researchId: string, sources: any[], round: number) => ({
      evidence: [{
        id: 'ev-1', researchId, sourceIds: [sources[0]?.id ?? 'src-1'],
        type: 'fact' as const,
        claim: 'React uses virtual DOM for efficient UI updates',
        context: 'Test context', tags: ['react'], confidence: 0.9,
        relevance: 0.9, round, contradicts: [], supports: [],
        createdAt: new Date().toISOString(),
      }],
      graph:  {
        researchId, nodes: ['ev-1'], supports: [],
        contradicts: [], clusters: [{ label: 'react', nodeIds: ['ev-1'] }],
      },
      dedup: { total: 1, unique: 1, duplicates: 0, merged: 0 },
    }),
  };
}

function makeStubSynthesizer() {
  return {
    synthesize: async () => ({
      text:
        '## Findings\n\nReact uses virtual DOM for efficient UI updates [S:1].\n' +
        'This allows React to batch multiple DOM operations [S:1].',
      tokensUsed:      200,
      costUsd:         0.0006,
      attempts:        1,
      validationScore: 0.95,
      isFallback:      false,
      sourcesUsed:     [],
    }),
  };
}

function makeStubReporter() {
  return {
    write: async (opts: any) => {
      opts.research && opts.research;
      return {
        reportPath:  '/tmp/test-report.md',
        wikiSlug:    'test-research',
        wordCount:   120,
        charCount:   600,
        title:       'Test Research',
      };
    },
  };
}

// ── Tests ─────────────────────────────────────────────────────────────────────

describe('LoopSupervisor', () => {
  test('allows rounds within limits', () => {
    const sv = new LoopSupervisor({ maxCostUsd: 1.0, maxMinutes: 60, maxRounds: 3 });

    const research: any = {
      id: 'r1', costUsd: 0.01, totalSources: 0,
    };

    expect(sv.checkBeforeRound(research, 1).allowed).toBe(true);
    expect(sv.checkBeforeRound(research, 3).allowed).toBe(true);
  });

  test('blocks round beyond maxRounds', () => {
    const sv = new LoopSupervisor({ maxRounds: 2 });

    const research: any = { id: 'r1', costUsd: 0.0, totalSources: 0 };

    expect(sv.checkBeforeRound(research, 3).allowed).toBe(false);
    expect(sv.checkBeforeRound(research, 3).reason).toContain('Round cap');
  });

  test('throws on cost cap exceeded', () => {
    const sv = new LoopSupervisor({ maxCostUsd: 0.10 });
    const research: any = { id: 'r1', costUsd: 0.15, totalSources: 0 };

    expect(() => sv.checkBeforeRound(research, 1)).toThrow();
  });

  test('blocks when source cap reached', () => {
    const sv = new LoopSupervisor({ maxSources: 5 });
    const research: any = { id: 'r1', costUsd: 0.0, totalSources: 10 };

    const result = sv.checkBeforeRound(research, 1);
    expect(result.allowed).toBe(false);
    expect(result.reason).toContain('Source cap');
  });
});

describe('AutoResearchLoop', () => {
  let tempDir: string;
  let store:   ResearchStore;
  let planner: ResearchPlanner;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'loop-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
    planner = new ResearchPlanner(store);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  function makeLoop() {
    return new AutoResearchLoop(
      store,
      planner,
      makeStubQueryEngine() as any,
      makeStubCollector() as any,
      makeStubSynthesizer() as any,
      makeStubReporter() as any,
      { outputDir: join(tempDir, 'reports') },
    );
  }

  test('yields loop:started event', async () => {
    const loop   = makeLoop();
    const events: LoopEvent[] = [];

    for await (const ev of loop.run({ goal: 'How does React work?', depth: 'surface' })) {
      events.push(ev);
    }

    expect(events[0]?.type).toBe('loop:started');
  });

  test('yields loop:complete event for successful run', async () => {
    const loop   = makeLoop();
    const events: LoopEvent[] = [];

    for await (const ev of loop.run({ goal: 'How does SQLite work?', depth: 'surface' })) {
      events.push(ev);
    }

    const types = events.map(e => e.type);
    expect(types).toContain('loop:complete');
    expect(types).not.toContain('loop:error');
  });

  test('yields all expected phase events in order', async () => {
    const loop   = makeLoop();
    const types: string[] = [];

    for await (const ev of loop.run({ goal: 'What is WAL mode?', depth: 'surface' })) {
      types.push(ev.type);
    }

    const expectedPhases = [
      'loop:started',
      'loop:planning',
      'loop:plan_ready',
      'loop:round_start',
      'loop:round_complete',
      'loop:collecting',
      'loop:collection_complete',
      'loop:synthesizing',
      'loop:synthesis_complete',
      'loop:writing',
      'loop:complete',
    ];

    for (const phase of expectedPhases) {
      expect(types).toContain(phase);
    }
  });

  test('creates research record in store', async () => {
    const loop = makeLoop();

    for await (const _ of loop.run({ goal: 'Test goal', depth: 'surface' })) {
      // consume
    }

    const all = store.listResearches();
    expect(all.length).toBe(1);
    expect(all[0]?.status).toBe('complete');
  });

  test('yields loop:error and sets status to failed on provider error', async () => {
    const failingSynthesizer = {
      synthesize: async () => {
        throw new Error('Provider unavailable');
      },
    };

    const loop = new AutoResearchLoop(
      store,
      planner,
      makeStubQueryEngine() as any,
      makeStubCollector() as any,
      failingSynthesizer as any,
      makeStubReporter() as any,
      { outputDir: join(tempDir, 'reports') },
    );

    const events: LoopEvent[] = [];
    for await (const ev of loop.run({ goal: 'Test failure', depth: 'surface' })) {
      events.push(ev);
    }

    const errorEvent = events.find(e => e.type === 'loop:error');
    expect(errorEvent).toBeTruthy();
    expect(errorEvent?.data.error).toContain('Provider unavailable');

    const all = store.listResearches();
    expect(all[0]?.status).toBe('failed');
  });

  test('loop:complete event contains report path and word count', async () => {
    const loop   = makeLoop();
    const events: LoopEvent[] = [];

    for await (const ev of loop.run({ goal: 'How does pnpm work?', depth: 'surface' })) {
      events.push(ev);
    }

    const complete = events.find(e => e.type === 'loop:complete');
    expect(complete?.data.reportPath).toBeTruthy();
    expect(complete?.data.wordCount).toBeGreaterThan(0);
  });

  test('forwards tags to research record', async () => {
    const loop = makeLoop();

    for await (const _ of loop.run({
      goal:  'Test with tags',
      depth: 'surface',
      tags:  ['typescript', 'testing'],
    })) {
      // consume
    }

    const all = store.listResearches();
    // tags are stored in JSON; find via direct store call
    const r = store.getResearch(all[0]!.id);
    expect(r?.tags).toContain('typescript');
    expect(r?.tags).toContain('testing');
  });
});
tests/kairos-integration.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore }    from '../src/store/ResearchStore.js';
import { ResearchPlanner }  from '../src/planning/ResearchPlanner.js';
import { AutoResearchLoop } from '../src/loop/AutoResearchLoop.js';
import { KairosIntegration } from '../src/kairos/KairosIntegration.js';

describe('KairosIntegration', () => {
  let tempDir: string;
  let store:   ResearchStore;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'kairos-int-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  function makeLoop(): AutoResearchLoop {
    const planner = new ResearchPlanner(store);
    return new AutoResearchLoop(
      store,
      planner,
      {
        executeRound: async (researchId: string) => ({
          sources: [],
          gaps:    [],
          stats:   { total: 0, succeeded: 0, failed: 0, skipped: 0 },
        }),
      } as any,
      {
        collect: async (researchId: string) => ({
          evidence: [],
          graph:    {
            researchId, nodes: [], supports: [],
            contradicts: [], clusters: [],
          },
          dedup: { total: 0, unique: 0, duplicates: 0, merged: 0 },
        }),
      } as any,
      {
        synthesize: async () => ({
          text:            '## Findings\n\nNo evidence was collected [S:1].',
          tokensUsed:      50,
          costUsd:         0.0001,
          attempts:        1,
          validationScore: 0.8,
          isFallback:      false,
          sourcesUsed:     [],
        }),
      } as any,
      {
        write: async (opts: any) => ({
          reportPath:  join(tempDir, 'test.md'),
          wikiSlug:    null,
          wordCount:   10,
          charCount:   50,
          title:       'Test',
        }),
      } as any,
      { outputDir: join(tempDir, 'reports') },
    );
  }

  test('registers job handler with Kairos', () => {
    const registeredJobs: string[] = [];

    const mockKairos = {
      registerJobHandler: (jobType: string, _: any) => {
        registeredJobs.push(jobType);
      },
    };

    const integration = new KairosIntegration({
      loop:          makeLoop(),
      store,
      kairosAgent:   mockKairos,
      observationLog: null,
      eventBus:      null,
      autoDream:     null,
    });

    integration.register();
    expect(registeredJobs).toContain('autoresearch');
  });

  test('does not throw when kairosAgent is null', () => {
    const integration = new KairosIntegration({
      loop:          makeLoop(),
      store,
      kairosAgent:   null,
      observationLog: null,
      eventBus:      null,
      autoDream:     null,
    });

    expect(() => integration.register()).not.toThrow();
  });

  test('emits EventBus events during loop run', async () => {
    const emittedEvents: string[] = [];
    let jobHandler: ((taskId: string, payload: any) => Promise<void>) | null = null;

    const mockKairos = {
      registerJobHandler: (_: string, handler: any) => {
        jobHandler = handler;
      },
    };

    const mockEventBus = {
      emit: (event: string) => emittedEvents.push(event),
    };

    const integration = new KairosIntegration({
      loop:          makeLoop(),
      store,
      kairosAgent:   mockKairos,
      observationLog: null,
      eventBus:      mockEventBus,
      autoDream:     null,
    });

    integration.register();

    // Trigger the handler as Kairos would
    await jobHandler!('task-123', { goal: 'How does WAL work?', depth: 'surface' });

    expect(emittedEvents.length).toBeGreaterThan(0);
    expect(emittedEvents.some(e => e.startsWith('autoresearch:'))).toBe(true);
  });

  test('appends observations to observation log', async () => {
    const observations: string[] = [];
    let jobHandler: ((taskId: string, payload: any) => Promise<void>) | null = null;

    const mockKairos = {
      registerJobHandler: (_: string, handler: any) => { jobHandler = handler; },
    };

    const mockObservationLog = {
      append: async (entry: { content: string }) => {
        observations.push(entry.content);
      },
    };

    const integration = new KairosIntegration({
      loop:          makeLoop(),
      store,
      kairosAgent:   mockKairos,
      observationLog: mockObservationLog,
      eventBus:      null,
      autoDream:     null,
    });

    integration.register();
    await jobHandler!('task-456', { goal: 'Test observation logging', depth: 'surface' });

    expect(observations.length).toBeGreaterThan(0);
    expect(observations.some(o => o.includes('triggered'))).toBe(true);
  });
});
tests/tools.test.ts
TypeScript

import { describe, test, expect, beforeEach, afterEach } from 'bun:test';
import { mkdtempSync, rmSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { ResearchStore }   from '../src/store/ResearchStore.js';
import { ResearchPlanner } from '../src/planning/ResearchPlanner.js';
import { AutoResearchLoop } from '../src/loop/AutoResearchLoop.js';
import { ResearchToolSet } from '../src/tools/ResearchToolSet.js';

describe('ResearchToolSet', () => {
  let tempDir: string;
  let store:   ResearchStore;
  let toolSet: ResearchToolSet;

  beforeEach(() => {
    tempDir = mkdtempSync(join(tmpdir(), 'tools-test-'));
    store   = new ResearchStore(join(tempDir, 'research.db'));

    const planner = new ResearchPlanner(store);

    const loop = new AutoResearchLoop(
      store,
      planner,
      {
        executeRound: async (researchId: string) => ({
          sources: [], gaps: [], stats: { total: 0, succeeded: 0, failed: 0, skipped: 0 },
        }),
      } as any,
      {
        collect: async (researchId: string) => ({
          evidence: [],
          graph:    { researchId, nodes: [], supports: [], contradicts: [], clusters: [] },
          dedup:    { total: 0, unique: 0, duplicates: 0, merged: 0 },
        }),
      } as any,
      {
        synthesize: async () => ({
          text:            '## Test\n\nSynthesis result [S:1].',
          tokensUsed:      100, costUsd: 0.0003, attempts: 1,
          validationScore: 0.9, isFallback: false, sourcesUsed: [],
        }),
      } as any,
      {
        write: async () => ({
          reportPath:  join(tempDir, 'report.md'),
          wikiSlug:    null,
          wordCount:   50,
          charCount:   300,
          title:       'Test Report',
        }),
      } as any,
      { outputDir: join(tempDir, 'reports') },
    );

    toolSet = new ResearchToolSet(loop, store, null);
  });

  afterEach(() => {
    store.close();
    rmSync(tempDir, { recursive: true, force: true });
  });

  test('getTools returns 5 tools', () => {
    const tools = toolSet.getTools();
    expect(tools).toHaveLength(5);
  });

  test('all tools have required fields', () => {
    const tools = toolSet.getTools();
    for (const tool of tools) {
      expect(tool.name).toBeTruthy();
      expect(tool.description).toBeTruthy();
      expect(tool.inputSchema).toBeTruthy();
      expect(typeof tool.parallelSafe).toBe('boolean');
      expect(tool.requiredPermission).toBeTruthy();
      expect(typeof tool.execute).toBe('function');
    }
  });

  test('research_status returns not found for missing id', async () => {
    const tool = toolSet.getTools().find(t => t.name === 'research_status')!;
    const result = await tool.execute({ researchId: 'nonexistent' }) as any;
    expect(result.error).toContain('not found');
  });

  test('research_list returns empty list initially', async () => {
    const tool   = toolSet.getTools().find(t => t.name === 'research_list')!;
    const result = await tool.execute({ limit: 10 }) as any;
    expect(Array.isArray(result.researches)).toBe(true);
    expect(result.total).toBe(0);
  });

  test('research_list returns researches after creation', async () => {
    store.createResearch({
      goal:     'Test research',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const tool   = toolSet.getTools().find(t => t.name === 'research_list')!;
    const result = await tool.execute({ limit: 10 }) as any;
    expect(result.total).toBe(1);
    expect(result.researches[0].goal).toBe('Test research');
  });

  test('research_get_evidence returns empty for no evidence', async () => {
    const r = store.createResearch({
      goal:     'Test',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const tool   = toolSet.getTools().find(t => t.name === 'research_get_evidence')!;
    const result = await tool.execute({ researchId: r.id }) as any;
    expect(result.evidence).toHaveLength(0);
  });

  test('research_cancel marks research as cancelled', async () => {
    const r = store.createResearch({
      goal:     'Test cancel',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    const tool   = toolSet.getTools().find(t => t.name === 'research_cancel')!;
    const result = await tool.execute({ researchId: r.id }) as any;
    expect(result.message).toContain('cancelled');

    const updated = store.getResearch(r.id);
    expect(updated?.status).toBe('cancelled');
  });

  test('research_cancel rejects already-complete research', async () => {
    const r = store.createResearch({
      goal:     'Test complete cancel',
      goalType: 'factual',
      depth:    'surface',
      tags:     [],
    });

    store.updateStatus(r.id, 'complete', { completedAt: new Date().toISOString() });

    const tool   = toolSet.getTools().find(t => t.name === 'research_cancel')!;
    const result = await tool.execute({ researchId: r.id }) as any;
    expect(result.error).toContain('Cannot cancel');
  });

  test('research_run completes and returns report path', async () => {
    const tool   = toolSet.getTools().find(t => t.name === 'research_run')!;
    const result = await tool.execute({
      goal:  'What is SQLite WAL mode?',
      depth: 'surface',
    }) as any;

    expect(result.status).toBe('complete');
    expect(result.reportPath).toBeTruthy();
    expect(result.researchId).toBeTruthy();
  });

  test('research_run is not parallelSafe', () => {
    const tool = toolSet.getTools().find(t => t.name === 'research_run')!;
    expect(tool.parallelSafe).toBe(false);
  });

  test('research_status is parallelSafe', () => {
    const tool = toolSet.getTools().find(t => t.name === 'research_status')!;
    expect(tool.parallelSafe).toBe(true);
  });
});
Summary: Pass 9 Complete (packages/autoresearch)
What Part 2 delivered
Layer	Components	Key design decisions
Synthesis	PromptBuilder, SynthesisValidator, SynthesisEngine	Token-budgeted prompts, goal-type specific instructions, contradiction surfacing, [S:N] citation system, up to 3 retry attempts, cost accumulation per attempt, fallback to evidence concatenation
Report	ReportFormatter, WikiPusher, ReportWriter	Wiki-compatible YAML frontmatter, source appendix, research trail section, conflict callout, goal-to-slug normalisation, filesystem write + wiki push (best-effort)
Loop	LoopSupervisor, AutoResearchLoop	Async-generator orchestrator (mirrors queryLoop from Pass 2), 5-phase lifecycle, supervisor guards (cost/time/depth/sources), gap-driven plan refinement between rounds, error isolation (yields loop:error, never throws)
Kairos	KairosIntegration	Registers autoresearch job handler, emits EventBus events for WebSocket streaming (Gateway Pass 8), appends observations to ObservationLog, triggers AutoDream after deep research
Tools	ResearchToolSet	5 tools: research_run, research_status, research_list, research_get_evidence, research_cancel; parallel-safe flags correct; permission tiers consistent with Pass 2
CLI	commands.ts	research:run (streaming coloured output) + research:report (table/JSON list + detail view); built on Node parseArgs (no extra deps)
Tests	5 test files	PromptBuilder budgeting/contradictions, SynthesisValidator pass/fail/warn, SynthesisEngine provider call/retry/fallback/cost, ReportFormatter sections/slug/citations, ReportWriter filesystem+wiki, LoopSupervisor guards, AutoResearchLoop phase ordering/failure/tags, KairosIntegration registration/observation, ResearchToolSet all 5 tools
Full packages/autoresearch architecture
text

CreateResearch input
       │
       ▼
[GoalParser] ──────────────► goalType
       │
       ▼
[ResearchPlanner] ─────────► QueryPlan (rounds × sources × strategies)
       │                      persisted via ResearchStore
       ▼
[QueryEngine] ──fan-out────► WebSearchAdapter → SourceFetcher → Source
               per round      CodeSearchAdapter → buildLocalSource
                               WikiSearchAdapter → buildLocalSource
                               MemorySearchAdapter → buildLocalSource
                               ↓
                              EvidenceCollector → Evidence + EvidenceGraph
       │
       ▼ (gap-fill refinement between rounds)
[LoopSupervisor] ──guards──► cost / time / depth / sources
       │
       ▼
[SynthesisEngine] ─────────► PromptBuilder → ProviderRouter → validated text
       │                      SynthesisValidator (retry up to 3×)
       ▼
[ReportWriter] ────────────► ReportFormatter → .md file + wiki push
       │                      ResearchStore.updateStatus('complete')
       ▼
[LoopEvent stream] ─────────► CLI / KairosIntegration / EventBus / ResearchToolSet
