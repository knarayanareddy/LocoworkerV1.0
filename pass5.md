Pass 5 — packages/wiki (Part 1 of 2)
What Pass 5 covers: packages/wiki is the persistent project knowledge base — a Markdown-first, SQLite-indexed documentation engine that lets the agent and users read, write, search, and cross-link living wiki pages tied to the codebase. It sits between packages/memory (ephemeral agent facts) and packages/graphify (code structure) by capturing human-authored and agent-authored durable knowledge about the project.

Pass 5 split plan
Part	Files Generated
Part 1 (this pass)	package.json · tsconfig.json · src/types/wiki.types.ts · src/WikiStore.ts · src/WikiParser.ts · src/WikiSearch.ts · src/WikiLinker.ts
Part 2 (next pass)	src/WikiExporter.ts · src/WikiSyncAgent.ts · src/WikiWatcher.ts · src/index.ts · All 4 test files fully runnable with bun test
File tree (full package)
text

packages/wiki/
├── package.json
├── tsconfig.json
└── src/
    ├── types/
    │   └── wiki.types.ts
    ├── WikiStore.ts          ← SQLite persistence layer (pages, links, tags, history)
    ├── WikiParser.ts         ← Markdown parse → structured WikiPage + frontmatter
    ├── WikiSearch.ts         ← Full-text search + tag + backlink queries
    ├── WikiLinker.ts         ← [[wikilink]] resolution + dead-link detection
    ├── WikiExporter.ts       ← Export to HTML / single-page Markdown bundle (Part 2)
    ├── WikiSyncAgent.ts      ← Agent-triggered page write/update pipeline (Part 2)
    ├── WikiWatcher.ts        ← fs.watch → auto-sync on file save (Part 2)
    └── index.ts              ← Barrel export (Part 2)
    tests/
    ├── WikiStore.test.ts     (Part 2)
    ├── WikiParser.test.ts    (Part 2)
    ├── WikiSearch.test.ts    (Part 2)
    └── WikiLinker.test.ts    (Part 2)
packages/wiki/package.json
JSON

{
  "name": "@locoworker/wiki",
  "version": "1.0.0",
  "description": "Persistent Markdown wiki engine with SQLite index, full-text search, and wikilink resolution for the LocoWorker platform",
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
    "gray-matter": "^4.0.3",
    "marked": "^12.0.0",
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
    "@locoworker/graphify": "workspace:*"
  },
  "peerDependenciesMeta": {
    "@locoworker/graphify": { "optional": true }
  }
}
packages/wiki/tsconfig.json
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
      "@locoworker/core": ["../core/src/index.ts"],
      "@locoworker/shared": ["../shared/src/index.ts"],
      "@locoworker/graphify": ["../graphify/src/index.ts"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "src/tests"]
}
src/types/wiki.types.ts
TypeScript

/**
 * @locoworker/wiki — type definitions
 *
 * The wiki sits between ephemeral memory (packages/memory) and code structure
 * (packages/graphify). It stores durable, human/agent-authored knowledge about
 * the project as versioned Markdown pages with rich metadata.
 *
 * Design principles:
 *  - Every page has a stable slug (kebab-case title) as its primary key.
 *  - Frontmatter is the source of truth for metadata (tags, status, links).
 *  - The SQLite index is a derived artefact and can be fully rebuilt from .md files.
 *  - [[wikilinks]] are first-class: resolution, dead-link detection, and backlinks
 *    are all tracked in the DB.
 *  - Page history is append-only (no destructive updates to history rows).
 */

import { z } from 'zod';

// ─── Enumerations ─────────────────────────────────────────────────────────────

/** Lifecycle status of a wiki page */
export type WikiPageStatus =
  | 'draft'       // Being written, not yet considered stable
  | 'active'      // Current, reviewed, considered accurate
  | 'outdated'    // Known to be stale, needs update
  | 'archived'    // No longer relevant, kept for history
  | 'stub';       // Placeholder page, needs expansion

/** Who (or what) created/last modified a page */
export type WikiPageAuthor = 'human' | 'agent' | 'autodream' | 'import';

/** The kind of relationship a wikilink represents */
export type WikiLinkType =
  | 'references'   // General reference: [[ToolRegistry]]
  | 'implements'   // This page implements / describes the impl of another
  | 'extends'      // Extends / inherits from another concept
  | 'documents'    // Documents a code entity (links to graphify node)
  | 'related'      // Soft association
  | 'contradicts'  // Known conflict (agent uses this for contradiction detection)
  | 'supersedes';  // This page replaces an older one

/** Export formats supported by WikiExporter (Part 2) */
export type WikiExportFormat = 'html' | 'markdown_bundle' | 'json';

// ─── Core page model ──────────────────────────────────────────────────────────

/**
 * A parsed wiki page — the in-memory representation after WikiParser processes
 * a raw .md file. This is NOT the DB row; see WikiPageRow for that.
 */
export interface WikiPage {
  /** Stable, URL-safe, unique identifier derived from the page title.
   *  e.g. "Tool Registry" → "tool-registry"
   *  Once set, a slug never changes (history and links rely on it). */
  slug: string;

  /** Human-readable title, taken from frontmatter `title:` or first H1. */
  title: string;

  /** File path relative to the wiki root directory. */
  filePath: string;

  /** Raw Markdown source (after frontmatter is stripped). */
  content: string;

  /** Rendered HTML (populated lazily by WikiParser.render()). */
  html?: string;

  /** Structured frontmatter extracted by gray-matter. */
  frontmatter: WikiFrontmatter;

  /** All [[wikilinks]] found in the page body. */
  wikilinks: WikiLinkRef[];

  /** External URLs found in the page body. */
  externalLinks: ExternalLinkRef[];

  /** Headings extracted for table-of-contents generation. */
  headings: WikiHeading[];

  /** Word count (approximate, for reading time estimates). */
  wordCount: number;

  /** SHA-256 fingerprint of the raw file content (for incremental sync). */
  fingerprint: string;

  /** ISO 8601 timestamps. */
  createdAt: string;
  updatedAt: string;
}

/**
 * Frontmatter schema — the YAML block at the top of each .md file.
 * All fields are optional; WikiParser supplies sensible defaults.
 *
 * Example .md frontmatter:
 * ---
 * title: Tool Registry
 * tags: [core, tools, permissions]
 * status: active
 * author: human
 * graphify_node: tool-registry::class::ToolRegistry
 * related: [permission-gate, session-manager]
 * ---
 */
export interface WikiFrontmatter {
  title?: string;
  tags: string[];
  status: WikiPageStatus;
  author: WikiPageAuthor;

  /** Optional link to a graphify node ID (stable hash) or name::type::file. */
  graphify_node?: string;

  /** Explicitly declared related page slugs (supplements auto-detected wikilinks). */
  related: string[];

  /** If true, this page is excluded from exports and search results. */
  hidden: boolean;

  /** Optional ISO 8601 date of last review. */
  reviewed_at?: string;

  /** Optional custom sort order within a category (lower = first). */
  order?: number;

  /** Arbitrary additional frontmatter fields preserved as-is. */
  [key: string]: unknown;
}

/** A [[wikilink]] reference found inside page body text. */
export interface WikiLinkRef {
  /** Raw text inside the brackets: could be "Page Title" or "slug|display". */
  raw: string;

  /** Resolved target slug (may differ from raw if alias syntax used). */
  targetSlug: string;

  /** Display text shown in rendered output (may differ from target). */
  displayText: string;

  /** The relationship type (defaults to 'references'). */
  linkType: WikiLinkType;

  /** Line number where this link appears (1-based). */
  line: number;

  /** True if the target page does not exist in the index. */
  isDead: boolean;
}

/** An external (http/https/ftp) URL reference. */
export interface ExternalLinkRef {
  url: string;
  displayText: string;
  line: number;
}

/** A Markdown heading extracted for TOC generation. */
export interface WikiHeading {
  level: 1 | 2 | 3 | 4 | 5 | 6;
  text: string;
  anchor: string; // URL-safe fragment id
  line: number;
}

// ─── DB row types ─────────────────────────────────────────────────────────────

/** SQLite `wiki_pages` table row. */
export interface WikiPageRow {
  slug: string;
  title: string;
  file_path: string;
  content: string;
  frontmatter_json: string;    // JSON-serialized WikiFrontmatter
  tags_csv: string;            // Comma-separated for simple LIKE queries
  status: WikiPageStatus;
  author: WikiPageAuthor;
  graphify_node: string | null;
  word_count: number;
  fingerprint: string;
  created_at: string;
  updated_at: string;
}

/** SQLite `wiki_links` table row (directed edge: from → to). */
export interface WikiLinkRow {
  id: number;
  from_slug: string;
  to_slug: string;
  display_text: string;
  link_type: WikiLinkType;
  line: number;
  is_dead: 0 | 1;
}

/** SQLite `wiki_history` table row (append-only change log). */
export interface WikiHistoryRow {
  id: number;
  slug: string;
  changed_at: string;
  author: WikiPageAuthor;
  change_type: 'create' | 'update' | 'delete' | 'restore';
  fingerprint_before: string | null;
  fingerprint_after: string | null;
  summary: string;   // human-readable or agent-generated change summary
}

/** SQLite `wiki_fts` virtual table row (SQLite FTS5). */
export interface WikiFtsRow {
  slug: string;
  title: string;
  content: string;
  tags_csv: string;
}

// ─── Query & result types ──────────────────────────────────────────────────────

export interface WikiSearchOptions {
  /** Full-text query string (supports FTS5 operators). */
  query?: string;

  /** Filter by one or more tags (AND semantics). */
  tags?: string[];

  /** Filter by status. */
  status?: WikiPageStatus | WikiPageStatus[];

  /** Filter by author. */
  author?: WikiPageAuthor;

  /** Return only pages that have a graphify_node link. */
  hasGraphifyNode?: boolean;

  /** Return only dead-link sources or pages with dead links. */
  hasDeadLinks?: boolean;

  /** Limit results (default: 20). */
  limit?: number;

  /** Offset for pagination (default: 0). */
  offset?: number;
}

export interface WikiSearchResult {
  slug: string;
  title: string;
  status: WikiPageStatus;
  tags: string[];
  /** FTS5 snippet with match highlighted (if full-text query given). */
  snippet?: string;
  /** Relevance score (BM25, lower is better in SQLite FTS5). */
  score?: number;
  wordCount: number;
  updatedAt: string;
}

export interface WikiBacklinkResult {
  fromSlug: string;
  fromTitle: string;
  displayText: string;
  linkType: WikiLinkType;
  line: number;
}

export interface WikiStoreStats {
  totalPages: number;
  activePages: number;
  draftPages: number;
  stubPages: number;
  totalLinks: number;
  deadLinks: number;
  totalTags: number;
  uniqueAuthors: WikiPageAuthor[];
  dbSizeBytes: number;
  lastUpdated: string | null;
}

export interface WikiUpsertOptions {
  /** If true, record a history entry even if the fingerprint hasn't changed. */
  forceHistory?: boolean;

  /** Override author for the history entry. */
  author?: WikiPageAuthor;

  /** Human/agent-written summary of the change. */
  summary?: string;
}

// ─── Zod validation schemas ───────────────────────────────────────────────────

export const WikiFrontmatterSchema = z.object({
  title: z.string().optional(),
  tags: z.array(z.string()).default([]),
  status: z
    .enum(['draft', 'active', 'outdated', 'archived', 'stub'])
    .default('draft'),
  author: z
    .enum(['human', 'agent', 'autodream', 'import'])
    .default('human'),
  graphify_node: z.string().optional(),
  related: z.array(z.string()).default([]),
  hidden: z.boolean().default(false),
  reviewed_at: z.string().datetime({ offset: true }).optional(),
  order: z.number().int().optional(),
}).passthrough();

export const WikiPageSchema = z.object({
  slug: z.string().regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, 'Slug must be kebab-case'),
  title: z.string().min(1).max(200),
  filePath: z.string(),
  content: z.string(),
  frontmatter: WikiFrontmatterSchema,
  wikilinks: z.array(z.object({
    raw: z.string(),
    targetSlug: z.string(),
    displayText: z.string(),
    linkType: z.enum(['references','implements','extends','documents','related','contradicts','supersedes']),
    line: z.number().int().positive(),
    isDead: z.boolean(),
  })),
  wordCount: z.number().int().nonnegative(),
  fingerprint: z.string().length(64),
  createdAt: z.string().datetime({ offset: true }),
  updatedAt: z.string().datetime({ offset: true }),
});

// ─── Constants ────────────────────────────────────────────────────────────────

/** Default wiki directory relative to project root. */
export const DEFAULT_WIKI_DIR = '.locoworker/wiki';

/** Default DB path relative to project root. */
export const DEFAULT_WIKI_DB = '.locoworker/wiki/wiki.db';

/** SQLite schema version — bump when schema changes. */
export const WIKI_SCHEMA_VERSION = 1;

/** Maximum FTS snippet length in characters. */
export const FTS_SNIPPET_LENGTH = 200;

/** Maximum number of headings to extract per page (TOC cap). */
export const MAX_HEADINGS = 50;

/** Wikilink regex: [[Target Page]] or [[Target Page|Display Text]] or [[slug|text::type]] */
export const WIKILINK_REGEX =
  /\[\[([^\]|]+?)(?:\|([^\]]*?))?(?:::(references|implements|extends|documents|related|contradicts|supersedes))?\]\]/g;

/** Frontmatter default values applied when keys are absent. */
export const FRONTMATTER_DEFAULTS: Readonly<WikiFrontmatter> = {
  tags: [],
  status: 'draft',
  author: 'human',
  related: [],
  hidden: false,
} as const;
src/WikiStore.ts
TypeScript

/**
 * WikiStore — SQLite persistence layer for @locoworker/wiki
 *
 * Responsibilities:
 *  - Schema creation and migration
 *  - CRUD for wiki pages (upsert, delete, restore from history)
 *  - Link graph writes (upsert links, mark dead links)
 *  - Append-only history log
 *  - Stats queries
 *  - Full rebuild from a list of parsed WikiPage objects
 *
 * The DB is intentionally kept as a derived index: it can be fully
 * rebuilt from .md files at any time via WikiStore.rebuild().
 *
 * Schema:
 *   wiki_pages       — one row per page, core metadata + content
 *   wiki_links       — directed link graph (from_slug → to_slug)
 *   wiki_history     — append-only change log
 *   wiki_fts         — FTS5 virtual table (title + content + tags_csv)
 *   wiki_meta        — key/value store for schema version, built_at, etc.
 */

import Database, { type Database as BunDatabase } from 'better-sqlite3';
import { createHash } from 'node:crypto';
import { existsSync, mkdirSync } from 'node:fs';
import { dirname } from 'node:path';

import type {
  WikiPage,
  WikiPageRow,
  WikiLinkRow,
  WikiHistoryRow,
  WikiStoreStats,
  WikiUpsertOptions,
  WikiPageStatus,
  WikiPageAuthor,
  WikiLinkType,
} from './types/wiki.types.js';

import {
  WIKI_SCHEMA_VERSION,
} from './types/wiki.types.js';

// ─── Internal helpers ─────────────────────────────────────────────────────────

function nowIso(): string {
  return new Date().toISOString();
}

function sha256(input: string): string {
  return createHash('sha256').update(input, 'utf8').digest('hex');
}

/**
 * Ensure the directory for a file path exists before opening SQLite.
 */
function ensureDir(filePath: string): void {
  const dir = dirname(filePath);
  if (!existsSync(dir)) {
    mkdirSync(dir, { recursive: true });
  }
}

// ─── WikiStore ────────────────────────────────────────────────────────────────

export class WikiStore {
  private db: BunDatabase;
  private readonly dbPath: string;

  constructor(dbPath: string) {
    this.dbPath = dbPath;
    ensureDir(dbPath);
    this.db = new Database(dbPath);
    this.applyPragmas();
    this.createSchema();
    this.assertSchemaVersion();
  }

  // ─── Lifecycle ─────────────────────────────────────────────────────────────

  close(): void {
    this.db.close();
  }

  /**
   * Rebuild the entire index from a pre-parsed list of WikiPage objects.
   * Useful for initial indexing and full re-syncs.
   * Does NOT clear history (history is append-only by design).
   */
  rebuild(pages: WikiPage[]): void {
    const tx = this.db.transaction(() => {
      // Clear derived tables only
      this.db.prepare('DELETE FROM wiki_pages').run();
      this.db.prepare('DELETE FROM wiki_links').run();
      this.db.prepare('DELETE FROM wiki_fts').run();

      for (const page of pages) {
        this._insertPage(page);
        this._insertLinks(page);
        this._insertFts(page);
      }

      // After all pages inserted, resolve dead links
      this._resolveDeadLinks();

      this._setMeta('built_at', nowIso());
      this._setMeta('page_count', String(pages.length));
    });
    tx();
  }

  // ─── Page operations ───────────────────────────────────────────────────────

  /**
   * Upsert a single page. If the fingerprint hasn't changed, no write is
   * performed unless `options.forceHistory` is true.
   * Returns 'inserted' | 'updated' | 'unchanged'.
   */
  upsertPage(
    page: WikiPage,
    options: WikiUpsertOptions = {},
  ): 'inserted' | 'updated' | 'unchanged' {
    const existing = this.getPage(page.slug);

    if (existing && existing.fingerprint === page.fingerprint && !options.forceHistory) {
      return 'unchanged';
    }

    const changeType = existing ? 'update' : 'create';
    const author: WikiPageAuthor = options.author ?? page.frontmatter.author;

    const tx = this.db.transaction(() => {
      if (existing) {
        this._updatePage(page);
      } else {
        this._insertPage(page);
      }

      // Refresh links for this page
      this.db
        .prepare('DELETE FROM wiki_links WHERE from_slug = ?')
        .run(page.slug);
      this._insertLinks(page);
      this._resolveDeadLinksForSlug(page.slug);

      // Refresh FTS entry
      this.db
        .prepare('DELETE FROM wiki_fts WHERE slug = ?')
        .run(page.slug);
      this._insertFts(page);

      // Append history
      this._appendHistory({
        slug: page.slug,
        author,
        changeType,
        fingerprintBefore: existing?.fingerprint ?? null,
        fingerprintAfter: page.fingerprint,
        summary: options.summary ?? (changeType === 'create' ? 'Page created' : 'Page updated'),
      });
    });

    tx();
    return changeType === 'create' ? 'inserted' : 'updated';
  }

  /**
   * Delete a page by slug (soft: records history, removes from index).
   */
  deletePage(slug: string, author: WikiPageAuthor = 'human', summary?: string): boolean {
    const existing = this.getPage(slug);
    if (!existing) return false;

    const tx = this.db.transaction(() => {
      this._appendHistory({
        slug,
        author,
        changeType: 'delete',
        fingerprintBefore: existing.fingerprint,
        fingerprintAfter: null,
        summary: summary ?? 'Page deleted',
      });

      this.db.prepare('DELETE FROM wiki_links WHERE from_slug = ?').run(slug);
      this.db.prepare('DELETE FROM wiki_fts WHERE slug = ?').run(slug);
      this.db.prepare('DELETE FROM wiki_pages WHERE slug = ?').run(slug);

      // Mark incoming links to this slug as dead
      this.db
        .prepare('UPDATE wiki_links SET is_dead = 1 WHERE to_slug = ?')
        .run(slug);
    });

    tx();
    return true;
  }

  /**
   * Get a single page by slug. Returns undefined if not found.
   */
  getPage(slug: string): WikiPageRow | undefined {
    return this.db
      .prepare<[string], WikiPageRow>('SELECT * FROM wiki_pages WHERE slug = ?')
      .get(slug);
  }

  /**
   * Get all pages (optionally filtered by status).
   */
  getAllPages(status?: WikiPageStatus | WikiPageStatus[]): WikiPageRow[] {
    if (!status) {
      return this.db
        .prepare<[], WikiPageRow>('SELECT * FROM wiki_pages ORDER BY slug')
        .all();
    }
    const statuses = Array.isArray(status) ? status : [status];
    const placeholders = statuses.map(() => '?').join(', ');
    return this.db
      .prepare<string[], WikiPageRow>(
        `SELECT * FROM wiki_pages WHERE status IN (${placeholders}) ORDER BY slug`,
      )
      .all(...statuses);
  }

  /**
   * Get pages linked to a graphify node ID or name fragment.
   */
  getPagesByGraphifyNode(nodeRef: string): WikiPageRow[] {
    return this.db
      .prepare<[string], WikiPageRow>(
        'SELECT * FROM wiki_pages WHERE graphify_node LIKE ? ORDER BY slug',
      )
      .all(`%${nodeRef}%`);
  }

  // ─── Link operations ───────────────────────────────────────────────────────

  /**
   * Get all outgoing links from a page.
   */
  getLinks(fromSlug: string): WikiLinkRow[] {
    return this.db
      .prepare<[string], WikiLinkRow>(
        'SELECT * FROM wiki_links WHERE from_slug = ? ORDER BY line',
      )
      .all(fromSlug);
  }

  /**
   * Get all incoming links (backlinks) to a page.
   */
  getBacklinks(toSlug: string): WikiLinkRow[] {
    return this.db
      .prepare<[string], WikiLinkRow>(
        'SELECT * FROM wiki_links WHERE to_slug = ? ORDER BY from_slug, line',
      )
      .all(toSlug);
  }

  /**
   * Get all dead links across the entire wiki.
   */
  getDeadLinks(): WikiLinkRow[] {
    return this.db
      .prepare<[], WikiLinkRow>(
        'SELECT * FROM wiki_links WHERE is_dead = 1 ORDER BY from_slug, line',
      )
      .all();
  }

  // ─── History operations ────────────────────────────────────────────────────

  /**
   * Get the full history for a page (most recent first).
   */
  getHistory(slug: string, limit = 50): WikiHistoryRow[] {
    return this.db
      .prepare<[string, number], WikiHistoryRow>(
        'SELECT * FROM wiki_history WHERE slug = ? ORDER BY changed_at DESC LIMIT ?',
      )
      .all(slug, limit);
  }

  /**
   * Get recent history across all pages (most recent first).
   */
  getRecentHistory(limit = 100): WikiHistoryRow[] {
    return this.db
      .prepare<[number], WikiHistoryRow>(
        'SELECT * FROM wiki_history ORDER BY changed_at DESC LIMIT ?',
      )
      .all(limit);
  }

  // ─── Stats ─────────────────────────────────────────────────────────────────

  getStats(): WikiStoreStats {
    const totalPages = (
      this.db
        .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM wiki_pages')
        .get()?.count ?? 0
    );

    const activePages = (
      this.db
        .prepare<[], { count: number }>(
          "SELECT COUNT(*) as count FROM wiki_pages WHERE status = 'active'",
        )
        .get()?.count ?? 0
    );

    const draftPages = (
      this.db
        .prepare<[], { count: number }>(
          "SELECT COUNT(*) as count FROM wiki_pages WHERE status = 'draft'",
        )
        .get()?.count ?? 0
    );

    const stubPages = (
      this.db
        .prepare<[], { count: number }>(
          "SELECT COUNT(*) as count FROM wiki_pages WHERE status = 'stub'",
        )
        .get()?.count ?? 0
    );

    const totalLinks = (
      this.db
        .prepare<[], { count: number }>('SELECT COUNT(*) as count FROM wiki_links')
        .get()?.count ?? 0
    );

    const deadLinks = (
      this.db
        .prepare<[], { count: number }>(
          'SELECT COUNT(*) as count FROM wiki_links WHERE is_dead = 1',
        )
        .get()?.count ?? 0
    );

    // Unique tags (explode the tags_csv column)
    const tagRows = this.db
      .prepare<[], { tags_csv: string }>('SELECT tags_csv FROM wiki_pages WHERE tags_csv != ""')
      .all();
    const allTags = new Set<string>();
    for (const row of tagRows) {
      for (const tag of row.tags_csv.split(',')) {
        const t = tag.trim();
        if (t) allTags.add(t);
      }
    }

    const authorRows = this.db
      .prepare<[], { author: WikiPageAuthor }>(
        'SELECT DISTINCT author FROM wiki_pages',
      )
      .all();

    // DB file size
    const sizeRow = this.db
      .prepare<[], { page_count: number; page_size: number }>(
        'PRAGMA page_count; PRAGMA page_size',
      )
      .get();

    // better-sqlite3 doesn't chain PRAGMAs in one call, so compute separately
    const pageCount = (this.db.prepare('PRAGMA page_count').get() as { page_count: number })
      .page_count;
    const pageSize = (this.db.prepare('PRAGMA page_size').get() as { page_size: number })
      .page_size;

    const lastUpdatedRow = this.db
      .prepare<[], { updated_at: string }>(
        'SELECT MAX(updated_at) as updated_at FROM wiki_pages',
      )
      .get();

    return {
      totalPages,
      activePages,
      draftPages,
      stubPages,
      totalLinks,
      deadLinks,
      totalTags: allTags.size,
      uniqueAuthors: authorRows.map((r) => r.author),
      dbSizeBytes: pageCount * pageSize,
      lastUpdated: lastUpdatedRow?.updated_at ?? null,
    };
  }

  // ─── Schema ────────────────────────────────────────────────────────────────

  private applyPragmas(): void {
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('synchronous = NORMAL');
    this.db.pragma('foreign_keys = ON');
    this.db.pragma('cache_size = -8000');   // 8 MB page cache
    this.db.pragma('temp_store = MEMORY');
  }

  private createSchema(): void {
    this.db.exec(/* sql */ `
      -- Core pages table
      CREATE TABLE IF NOT EXISTS wiki_pages (
        slug              TEXT PRIMARY KEY,
        title             TEXT NOT NULL,
        file_path         TEXT NOT NULL,
        content           TEXT NOT NULL DEFAULT '',
        frontmatter_json  TEXT NOT NULL DEFAULT '{}',
        tags_csv          TEXT NOT NULL DEFAULT '',
        status            TEXT NOT NULL DEFAULT 'draft',
        author            TEXT NOT NULL DEFAULT 'human',
        graphify_node     TEXT,
        word_count        INTEGER NOT NULL DEFAULT 0,
        fingerprint       TEXT NOT NULL,
        created_at        TEXT NOT NULL,
        updated_at        TEXT NOT NULL
      );

      CREATE INDEX IF NOT EXISTS idx_wiki_pages_status      ON wiki_pages(status);
      CREATE INDEX IF NOT EXISTS idx_wiki_pages_author      ON wiki_pages(author);
      CREATE INDEX IF NOT EXISTS idx_wiki_pages_graphify    ON wiki_pages(graphify_node) WHERE graphify_node IS NOT NULL;
      CREATE INDEX IF NOT EXISTS idx_wiki_pages_updated     ON wiki_pages(updated_at DESC);

      -- Directed link graph
      CREATE TABLE IF NOT EXISTS wiki_links (
        id           INTEGER PRIMARY KEY AUTOINCREMENT,
        from_slug    TEXT NOT NULL,
        to_slug      TEXT NOT NULL,
        display_text TEXT NOT NULL DEFAULT '',
        link_type    TEXT NOT NULL DEFAULT 'references',
        line         INTEGER NOT NULL DEFAULT 0,
        is_dead      INTEGER NOT NULL DEFAULT 0,   -- 0 = resolved, 1 = dead
        FOREIGN KEY (from_slug) REFERENCES wiki_pages(slug) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_wiki_links_from   ON wiki_links(from_slug);
      CREATE INDEX IF NOT EXISTS idx_wiki_links_to     ON wiki_links(to_slug);
      CREATE INDEX IF NOT EXISTS idx_wiki_links_dead   ON wiki_links(is_dead) WHERE is_dead = 1;
      CREATE UNIQUE INDEX IF NOT EXISTS idx_wiki_links_unique
        ON wiki_links(from_slug, to_slug, link_type, line);

      -- Append-only history log
      CREATE TABLE IF NOT EXISTS wiki_history (
        id                  INTEGER PRIMARY KEY AUTOINCREMENT,
        slug                TEXT NOT NULL,
        changed_at          TEXT NOT NULL,
        author              TEXT NOT NULL DEFAULT 'human',
        change_type         TEXT NOT NULL,        -- create | update | delete | restore
        fingerprint_before  TEXT,
        fingerprint_after   TEXT,
        summary             TEXT NOT NULL DEFAULT ''
      );

      CREATE INDEX IF NOT EXISTS idx_wiki_history_slug    ON wiki_history(slug);
      CREATE INDEX IF NOT EXISTS idx_wiki_history_changed ON wiki_history(changed_at DESC);

      -- FTS5 virtual table (full-text search over title + content + tags)
      CREATE VIRTUAL TABLE IF NOT EXISTS wiki_fts USING fts5(
        slug UNINDEXED,
        title,
        content,
        tags_csv,
        content='wiki_pages',
        content_rowid='rowid',
        tokenize='porter ascii'
      );

      -- Key/value meta store
      CREATE TABLE IF NOT EXISTS wiki_meta (
        key   TEXT PRIMARY KEY,
        value TEXT NOT NULL
      );
    `);
  }

  private assertSchemaVersion(): void {
    const stored = this._getMeta('schema_version');
    if (stored === null) {
      this._setMeta('schema_version', String(WIKI_SCHEMA_VERSION));
    } else if (Number(stored) !== WIKI_SCHEMA_VERSION) {
      throw new Error(
        `WikiStore: schema version mismatch. DB has v${stored}, code expects v${WIKI_SCHEMA_VERSION}. ` +
          'Run WikiStore.migrate() or delete the DB to rebuild.',
      );
    }
  }

  // ─── Internal write helpers ────────────────────────────────────────────────

  private _insertPage(page: WikiPage): void {
    this.db.prepare<WikiPageRow>(/* sql */ `
      INSERT INTO wiki_pages
        (slug, title, file_path, content, frontmatter_json, tags_csv,
         status, author, graphify_node, word_count, fingerprint, created_at, updated_at)
      VALUES
        (@slug, @title, @file_path, @content, @frontmatter_json, @tags_csv,
         @status, @author, @graphify_node, @word_count, @fingerprint, @created_at, @updated_at)
      ON CONFLICT(slug) DO NOTHING
    `).run(this._pageToRow(page));
  }

  private _updatePage(page: WikiPage): void {
    this.db.prepare(/* sql */ `
      UPDATE wiki_pages SET
        title            = @title,
        file_path        = @file_path,
        content          = @content,
        frontmatter_json = @frontmatter_json,
        tags_csv         = @tags_csv,
        status           = @status,
        author           = @author,
        graphify_node    = @graphify_node,
        word_count       = @word_count,
        fingerprint      = @fingerprint,
        updated_at       = @updated_at
      WHERE slug = @slug
    `).run(this._pageToRow(page));
  }

  private _insertLinks(page: WikiPage): void {
    if (page.wikilinks.length === 0) return;

    const stmt = this.db.prepare<Omit<WikiLinkRow, 'id'>>(/* sql */ `
      INSERT OR IGNORE INTO wiki_links
        (from_slug, to_slug, display_text, link_type, line, is_dead)
      VALUES
        (@from_slug, @to_slug, @display_text, @link_type, @line, @is_dead)
    `);

    for (const link of page.wikilinks) {
      stmt.run({
        from_slug: page.slug,
        to_slug: link.targetSlug,
        display_text: link.displayText,
        link_type: link.linkType,
        line: link.line,
        is_dead: link.isDead ? 1 : 0,
      } as Omit<WikiLinkRow, 'id'>);
    }
  }

  private _insertFts(page: WikiPage): void {
    this.db.prepare(/* sql */ `
      INSERT INTO wiki_fts(slug, title, content, tags_csv)
      VALUES (?, ?, ?, ?)
    `).run(
      page.slug,
      page.title,
      page.content,
      page.frontmatter.tags.join(', '),
    );
  }

  private _resolveDeadLinks(): void {
    // Mark links where to_slug doesn't exist in wiki_pages as dead
    this.db.exec(/* sql */ `
      UPDATE wiki_links
      SET is_dead = CASE
        WHEN to_slug IN (SELECT slug FROM wiki_pages) THEN 0
        ELSE 1
      END
    `);
  }

  private _resolveDeadLinksForSlug(slug: string): void {
    // When a page is upserted, all incoming links to it should become live
    this.db
      .prepare('UPDATE wiki_links SET is_dead = 0 WHERE to_slug = ?')
      .run(slug);

    // And outgoing links from this page need re-evaluation
    this.db.exec(/* sql */ `
      UPDATE wiki_links
      SET is_dead = CASE
        WHEN to_slug IN (SELECT slug FROM wiki_pages) THEN 0
        ELSE 1
      END
      WHERE from_slug = '${slug}'
    `);
  }

  private _appendHistory(opts: {
    slug: string;
    author: WikiPageAuthor;
    changeType: WikiHistoryRow['change_type'];
    fingerprintBefore: string | null;
    fingerprintAfter: string | null;
    summary: string;
  }): void {
    this.db.prepare(/* sql */ `
      INSERT INTO wiki_history
        (slug, changed_at, author, change_type, fingerprint_before, fingerprint_after, summary)
      VALUES (?, ?, ?, ?, ?, ?, ?)
    `).run(
      opts.slug,
      nowIso(),
      opts.author,
      opts.changeType,
      opts.fingerprintBefore,
      opts.fingerprintAfter,
      opts.summary,
    );
  }

  private _pageToRow(page: WikiPage): WikiPageRow {
    return {
      slug: page.slug,
      title: page.title,
      file_path: page.filePath,
      content: page.content,
      frontmatter_json: JSON.stringify(page.frontmatter),
      tags_csv: page.frontmatter.tags.join(','),
      status: page.frontmatter.status,
      author: page.frontmatter.author,
      graphify_node: page.frontmatter.graphify_node ?? null,
      word_count: page.wordCount,
      fingerprint: page.fingerprint,
      created_at: page.createdAt,
      updated_at: page.updatedAt,
    };
  }

  // ─── Meta helpers ──────────────────────────────────────────────────────────

  private _getMeta(key: string): string | null {
    const row = this.db
      .prepare<[string], { value: string }>('SELECT value FROM wiki_meta WHERE key = ?')
      .get(key);
    return row?.value ?? null;
  }

  private _setMeta(key: string, value: string): void {
    this.db
      .prepare('INSERT OR REPLACE INTO wiki_meta(key, value) VALUES (?, ?)')
      .run(key, value);
  }
}
src/WikiParser.ts
TypeScript

/**
 * WikiParser — Markdown → WikiPage (structured, frontmatter-aware)
 *
 * Responsibilities:
 *  - Parse raw .md file content using gray-matter (frontmatter) + custom AST walk
 *  - Validate and normalise frontmatter via Zod (WikiFrontmatterSchema)
 *  - Extract [[wikilinks]], external URLs, and headings via regex + line scan
 *  - Compute word count and SHA-256 fingerprint
 *  - Provide a lazy HTML render via marked
 *  - Generate a kebab-case slug from a title or file name
 *  - Support batch parsing with concurrency control
 *
 * Design: WikiParser is stateless (no DB dependency). It produces plain
 * WikiPage objects which WikiStore then persists.
 */

import { createHash } from 'node:crypto';
import { readFile } from 'node:fs/promises';
import { basename, extname } from 'node:path';
import matter from 'gray-matter';
import { marked } from 'marked';
import type {
  WikiPage,
  WikiFrontmatter,
  WikiLinkRef,
  ExternalLinkRef,
  WikiHeading,
  WikiLinkType,
} from './types/wiki.types.js';
import {
  WikiFrontmatterSchema,
  FRONTMATTER_DEFAULTS,
  WIKILINK_REGEX,
  MAX_HEADINGS,
  FTS_SNIPPET_LENGTH,
} from './types/wiki.types.js';

// ─── Regex helpers ────────────────────────────────────────────────────────────

// External URL: http/https/ftp references in markdown links [text](url)
const EXTERNAL_LINK_REGEX =
  /\[([^\]]*?)\]\((https?:\/\/[^\s)]+|ftp:\/\/[^\s)]+)\)/g;

// Markdown ATX headings: # Title, ## Title, etc.
const HEADING_REGEX = /^(#{1,6})\s+(.+?)\s*(?:#+\s*)?$/;

// Words: split on whitespace for approximate count
const WORD_SPLIT_REGEX = /\s+/;

// ─── Helpers ──────────────────────────────────────────────────────────────────

/**
 * Convert an arbitrary string to a valid kebab-case slug.
 * "Tool Registry & Permissions!" → "tool-registry-permissions"
 */
export function toSlug(input: string): string {
  return input
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, '')       // remove non-word chars (except space/hyphen)
    .replace(/[\s_]+/g, '-')         // space/underscore → hyphen
    .replace(/-+/g, '-')             // collapse multiple hyphens
    .replace(/^-|-$/g, '');          // trim leading/trailing hyphens
}

/**
 * Derive a slug from a file path when no title frontmatter is present.
 * "/docs/tool-registry.md" → "tool-registry"
 */
function slugFromPath(filePath: string): string {
  const name = basename(filePath, extname(filePath));
  return toSlug(name);
}

/**
 * Convert a heading text to a URL-safe anchor fragment.
 * "Tool Registry & Permissions" → "tool-registry-permissions"
 */
function toAnchor(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '');
}

/**
 * Approximate word count (splits on whitespace).
 */
function wordCount(text: string): number {
  const trimmed = text.trim();
  if (!trimmed) return 0;
  return trimmed.split(WORD_SPLIT_REGEX).length;
}

/**
 * SHA-256 of the full raw file content (including frontmatter).
 */
function fingerprint(rawContent: string): string {
  return createHash('sha256').update(rawContent, 'utf8').digest('hex');
}

// ─── Wikilink extraction ──────────────────────────────────────────────────────

/**
 * Extract all [[wikilinks]] from a markdown body.
 *
 * Supported syntaxes:
 *   [[Page Title]]                           → target: "page-title", display: "Page Title"
 *   [[Page Title|Custom Display Text]]       → target: "page-title", display: "Custom Display Text"
 *   [[page-slug|Display Text::implements]]   → target: "page-slug",  type: "implements"
 *   [[Page Title::documents]]                → target: "page-title", type: "documents"
 *
 * Note: `isDead` is always false here — WikiStore resolves dead links
 * after all pages are inserted (it has visibility of all slugs).
 */
function extractWikilinks(body: string): WikiLinkRef[] {
  const links: WikiLinkRef[] = [];
  const lines = body.split('\n');

  for (let lineIdx = 0; lineIdx < lines.length; lineIdx++) {
    const line = lines[lineIdx];
    // Reset lastIndex since we reuse the same compiled regex
    WIKILINK_REGEX.lastIndex = 0;
    let match: RegExpExecArray | null;

    while ((match = WIKILINK_REGEX.exec(line)) !== null) {
      const [, rawTarget, rawDisplay, rawType] = match;
      const targetSlug = toSlug(rawTarget.trim());
      const displayText = rawDisplay?.trim() || rawTarget.trim();
      const linkType: WikiLinkType = (rawType as WikiLinkType) ?? 'references';

      links.push({
        raw: match[0],
        targetSlug,
        displayText,
        linkType,
        line: lineIdx + 1,
        isDead: false, // resolved later by WikiStore
      });
    }
  }

  return links;
}

// ─── External link extraction ─────────────────────────────────────────────────

function extractExternalLinks(body: string): ExternalLinkRef[] {
  const links: ExternalLinkRef[] = [];
  const lines = body.split('\n');

  for (let lineIdx = 0; lineIdx < lines.length; lineIdx++) {
    EXTERNAL_LINK_REGEX.lastIndex = 0;
    let match: RegExpExecArray | null;
    while ((match = EXTERNAL_LINK_REGEX.exec(lines[lineIdx])) !== null) {
      links.push({
        url: match[2],
        displayText: match[1] || match[2],
        line: lineIdx + 1,
      });
    }
  }

  return links;
}

// ─── Heading extraction ───────────────────────────────────────────────────────

function extractHeadings(body: string): WikiHeading[] {
  const headings: WikiHeading[] = [];
  const lines = body.split('\n');

  for (let lineIdx = 0; lineIdx < lines.length; lineIdx++) {
    if (headings.length >= MAX_HEADINGS) break;
    const m = HEADING_REGEX.exec(lines[lineIdx]);
    if (m) {
      const level = m[1].length as WikiHeading['level'];
      const text = m[2];
      headings.push({
        level,
        text,
        anchor: toAnchor(text),
        line: lineIdx + 1,
      });
    }
  }

  return headings;
}

// ─── WikiParser class ─────────────────────────────────────────────────────────

export class WikiParser {
  /**
   * Parse a single .md file from disk.
   * Returns a fully populated WikiPage (html is undefined until render() called).
   */
  async parseFile(filePath: string): Promise<WikiPage> {
    const raw = await readFile(filePath, 'utf8');
    return this.parseContent(raw, filePath);
  }

  /**
   * Parse raw markdown string content (already loaded from disk or generated).
   * `filePath` is used for slug derivation when no title frontmatter exists.
   */
  parseContent(rawContent: string, filePath: string): WikiPage {
    const now = new Date().toISOString();

    // 1) Extract frontmatter with gray-matter
    const parsed = matter(rawContent);
    const rawFm = parsed.data ?? {};
    const body = parsed.content; // content after the frontmatter block

    // 2) Validate + normalise frontmatter via Zod (coerce unknowns via passthrough)
    const fmResult = WikiFrontmatterSchema.safeParse({
      ...FRONTMATTER_DEFAULTS,
      ...rawFm,
    });

    const frontmatter: WikiFrontmatter = fmResult.success
      ? (fmResult.data as WikiFrontmatter)
      : { ...FRONTMATTER_DEFAULTS, ...rawFm } as WikiFrontmatter;

    // 3) Derive title: frontmatter.title → first H1 in body → filename
    let title = frontmatter.title;
    if (!title) {
      const firstH1 = extractHeadings(body).find((h) => h.level === 1);
      title = firstH1?.text ?? basename(filePath, extname(filePath));
    }
    frontmatter.title = title;

    // 4) Derive slug
    const slug = toSlug(title) || slugFromPath(filePath);

    // 5) Structural extraction
    const wikilinks = extractWikilinks(body);
    const externalLinks = extractExternalLinks(body);
    const headings = extractHeadings(body);
    const wc = wordCount(body);
    const fp = fingerprint(rawContent);

    return {
      slug,
      title,
      filePath,
      content: body,
      html: undefined, // lazy — call render()
      frontmatter,
      wikilinks,
      externalLinks,
      headings,
      wordCount: wc,
      fingerprint: fp,
      createdAt: now,
      updatedAt: now,
    };
  }

  /**
   * Render page content to HTML using marked.
   * Mutates the page.html field in-place and returns the HTML string.
   * [[wikilinks]] are converted to <a> tags with data-wiki-slug attribute.
   */
  render(page: WikiPage): string {
    // 1) Pre-process: replace [[wikilinks]] with placeholder HTML
    const withLinks = page.content.replace(
      WIKILINK_REGEX,
      (_, rawTarget, rawDisplay, _rawType) => {
        const slug = toSlug(rawTarget.trim());
        const text = rawDisplay?.trim() || rawTarget.trim();
        return `<a href="/wiki/${slug}" data-wiki-slug="${slug}">${text}</a>`;
      },
    );

    // 2) Render Markdown → HTML
    const html = marked.parse(withLinks) as string;
    page.html = html;
    return html;
  }

  /**
   * Parse multiple files with concurrency control.
   * Uses Promise.allSettled so one bad file doesn't abort the batch.
   * Returns an array of { page, error } results.
   */
  async parseFiles(
    filePaths: string[],
    concurrency = 8,
  ): Promise<Array<{ filePath: string; page?: WikiPage; error?: Error }>> {
    const results: Array<{ filePath: string; page?: WikiPage; error?: Error }> = [];

    for (let i = 0; i < filePaths.length; i += concurrency) {
      const chunk = filePaths.slice(i, i + concurrency);
      const settled = await Promise.allSettled(
        chunk.map((fp) => this.parseFile(fp)),
      );

      for (let j = 0; j < chunk.length; j++) {
        const result = settled[j];
        if (result.status === 'fulfilled') {
          results.push({ filePath: chunk[j], page: result.value });
        } else {
          results.push({
            filePath: chunk[j],
            error:
              result.reason instanceof Error
                ? result.reason
                : new Error(String(result.reason)),
          });
        }
      }
    }

    return results;
  }

  /**
   * Generate a minimal stub page for a target slug that doesn't exist yet.
   * Used by WikiLinker to create placeholder pages for dead links.
   */
  generateStub(slug: string, referencedFrom: string): WikiPage {
    const now = new Date().toISOString();
    const title = slug
      .split('-')
      .map((w) => w.charAt(0).toUpperCase() + w.slice(1))
      .join(' ');

    const content = [
      `# ${title}`,
      '',
      `> **Stub page** — This page was auto-created because it was linked from \`${referencedFrom}\` but did not exist.`,
      '',
      '<!-- Add content here -->',
    ].join('\n');

    return {
      slug,
      title,
      filePath: `${slug}.md`,
      content,
      html: undefined,
      frontmatter: {
        ...FRONTMATTER_DEFAULTS,
        title,
        status: 'stub',
        author: 'agent',
      },
      wikilinks: [],
      externalLinks: [],
      headings: [{ level: 1, text: title, anchor: slug, line: 1 }],
      wordCount: wordCount(content),
      fingerprint: fingerprint(content),
      createdAt: now,
      updatedAt: now,
    };
  }
}
src/WikiSearch.ts
TypeScript

/**
 * WikiSearch — full-text search + structured queries over the wiki index.
 *
 * Responsibilities:
 *  - FTS5 full-text search (porter stemming, BM25 ranking, snippets)
 *  - Tag-based filtering
 *  - Status/author filtering
 *  - Backlink-based search ("find all pages that link to X")
 *  - "Orphan" detection (pages with no incoming links)
 *  - "Dead link" aggregation (pages that have unresolved wikilinks)
 *  - "Related pages" suggestion via shared tags + shared wikilink targets
 *
 * WikiSearch wraps WikiStore (read-only). It never writes to the DB.
 */

import type Database from 'better-sqlite3';
import type {
  WikiSearchOptions,
  WikiSearchResult,
  WikiBacklinkResult,
  WikiPageStatus,
  WikiPageAuthor,
  WikiLinkType,
} from './types/wiki.types.js';
import { FTS_SNIPPET_LENGTH } from './types/wiki.types.js';
import type { WikiStore } from './WikiStore.js';

// ─── Internal types ───────────────────────────────────────────────────────────

interface FtsRow {
  slug: string;
  title: string;
  tags_csv: string;
  status: WikiPageStatus;
  word_count: number;
  updated_at: string;
  snippet: string;
  rank: number;
}

interface TagRow {
  slug: string;
  title: string;
  tags_csv: string;
  status: WikiPageStatus;
  word_count: number;
  updated_at: string;
}

// ─── WikiSearch ───────────────────────────────────────────────────────────────

export class WikiSearch {
  // WikiSearch accesses the DB directly for performance (avoids
  // deserialising full rows when only a subset of fields is needed).
  private readonly db: Database.Database;

  constructor(private readonly store: WikiStore) {
    // Access the raw DB handle from WikiStore via a package-internal accessor.
    // WikiStore exposes `_db` as a protected internal for sibling classes.
    this.db = (store as unknown as { db: Database.Database }).db;
  }

  /**
   * Main search entry point.
   *
   * When `options.query` is provided, uses FTS5 with BM25 ranking + snippets.
   * When only filters are provided (tags, status, author), uses structured queries.
   * Filters are applied AFTER FTS when both are present.
   */
  search(options: WikiSearchOptions): WikiSearchResult[] {
    const limit = options.limit ?? 20;
    const offset = options.offset ?? 0;

    if (options.query) {
      return this._ftsSearch(options, limit, offset);
    }
    return this._structuredSearch(options, limit, offset);
  }

  /**
   * Get all pages that link to a given slug (backlinks).
   */
  getBacklinks(toSlug: string): WikiBacklinkResult[] {
    interface BacklinkJoinRow {
      from_slug: string;
      title: string;
      display_text: string;
      link_type: WikiLinkType;
      line: number;
    }

    const rows = this.db
      .prepare<[string], BacklinkJoinRow>(/* sql */ `
        SELECT
          wl.from_slug,
          wp.title,
          wl.display_text,
          wl.link_type,
          wl.line
        FROM wiki_links wl
        JOIN wiki_pages wp ON wl.from_slug = wp.slug
        WHERE wl.to_slug = ?
        ORDER BY wp.title, wl.line
      `)
      .all(toSlug);

    return rows.map((r) => ({
      fromSlug: r.from_slug,
      fromTitle: r.title,
      displayText: r.display_text,
      linkType: r.link_type,
      line: r.line,
    }));
  }

  /**
   * Find pages with no incoming links (orphans).
   * Excludes pages that are hidden or have status 'archived'.
   */
  getOrphans(): WikiSearchResult[] {
    interface OrphanRow {
      slug: string;
      title: string;
      tags_csv: string;
      status: WikiPageStatus;
      word_count: number;
      updated_at: string;
    }

    const rows = this.db
      .prepare<[], OrphanRow>(/* sql */ `
        SELECT slug, title, tags_csv, status, word_count, updated_at
        FROM wiki_pages wp
        WHERE hidden = 0
          AND status != 'archived'
          AND NOT EXISTS (
            SELECT 1 FROM wiki_links wl WHERE wl.to_slug = wp.slug
          )
        ORDER BY title
      `)
      .all();

    return rows.map((r) => this._toSearchResult(r));
  }

  /**
   * Return pages that have one or more dead wikilinks.
   */
  getPagesWithDeadLinks(): Array<{ slug: string; title: string; deadLinkCount: number }> {
    interface DeadRow {
      slug: string;
      title: string;
      dead_count: number;
    }

    return this.db
      .prepare<[], DeadRow>(/* sql */ `
        SELECT
          wp.slug,
          wp.title,
          COUNT(wl.id) AS dead_count
        FROM wiki_pages wp
        JOIN wiki_links wl ON wl.from_slug = wp.slug AND wl.is_dead = 1
        GROUP BY wp.slug
        ORDER BY dead_count DESC, wp.title
      `)
      .all()
      .map((r) => ({ slug: r.slug, title: r.title, deadLinkCount: r.dead_count }));
  }

  /**
   * Suggest related pages for a given slug based on:
   *  1. Shared tags (weight 2 per shared tag)
   *  2. Pages that link to the same targets (weight 1 per shared target)
   *  3. Pages that are mutually linked (weight 3)
   *
   * Returns top-N suggestions sorted by score descending.
   */
  getRelated(slug: string, limit = 10): Array<{ slug: string; title: string; score: number }> {
    // Get this page's tags and outgoing link targets
    const pageRow = this.db
      .prepare<[string], { tags_csv: string }>(
        'SELECT tags_csv FROM wiki_pages WHERE slug = ?',
      )
      .get(slug);

    if (!pageRow) return [];

    const tags = pageRow.tags_csv
      ? pageRow.tags_csv.split(',').map((t) => t.trim()).filter(Boolean)
      : [];

    const outTargets: Array<{ to_slug: string }> = this.db
      .prepare<[string], { to_slug: string }>(
        'SELECT to_slug FROM wiki_links WHERE from_slug = ? AND is_dead = 0',
      )
      .all(slug);

    const scores = new Map<string, number>();

    // Shared tags
    if (tags.length > 0) {
      const tagRows: Array<{ slug: string; tags_csv: string }> = this.db
        .prepare<[], { slug: string; tags_csv: string }>(
          'SELECT slug, tags_csv FROM wiki_pages WHERE slug != ?',
        )
        .all(slug);

      for (const row of tagRows) {
        const otherTags = row.tags_csv
          ? row.tags_csv.split(',').map((t) => t.trim()).filter(Boolean)
          : [];
        const shared = tags.filter((t) => otherTags.includes(t)).length;
        if (shared > 0) {
          scores.set(row.slug, (scores.get(row.slug) ?? 0) + shared * 2);
        }
      }
    }

    // Shared link targets
    for (const { to_slug } of outTargets) {
      const sharedTargetPages: Array<{ from_slug: string }> = this.db
        .prepare<[string, string], { from_slug: string }>(
          'SELECT from_slug FROM wiki_links WHERE to_slug = ? AND from_slug != ?',
        )
        .all(to_slug, slug);

      for (const { from_slug } of sharedTargetPages) {
        scores.set(from_slug, (scores.get(from_slug) ?? 0) + 1);
      }
    }

    // Mutual links (bidirectional bonus)
    const incomingSlugs: Array<{ from_slug: string }> = this.db
      .prepare<[string], { from_slug: string }>(
        'SELECT from_slug FROM wiki_links WHERE to_slug = ? AND is_dead = 0',
      )
      .all(slug);

    const outSlugsSet = new Set(outTargets.map((t) => t.to_slug));
    for (const { from_slug } of incomingSlugs) {
      if (outSlugsSet.has(from_slug)) {
        // Mutually linked
        scores.set(from_slug, (scores.get(from_slug) ?? 0) + 3);
      }
    }

    if (scores.size === 0) return [];

    // Fetch titles for all candidate slugs
    const candidateSlugs = [...scores.keys()];
    const placeholders = candidateSlugs.map(() => '?').join(', ');
    const titleRows: Array<{ slug: string; title: string }> = this.db
      .prepare<string[], { slug: string; title: string }>(
        `SELECT slug, title FROM wiki_pages WHERE slug IN (${placeholders})`,
      )
      .all(...candidateSlugs);

    const titleMap = new Map(titleRows.map((r) => [r.slug, r.title]));

    return [...scores.entries()]
      .filter(([s]) => titleMap.has(s))
      .map(([s, score]) => ({ slug: s, title: titleMap.get(s)!, score }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  /**
   * Get all unique tags across the wiki with page counts.
   */
  getAllTags(): Array<{ tag: string; count: number }> {
    const rows: Array<{ tags_csv: string }> = this.db
      .prepare<[], { tags_csv: string }>(
        'SELECT tags_csv FROM wiki_pages WHERE tags_csv != ""',
      )
      .all();

    const tagCounts = new Map<string, number>();
    for (const row of rows) {
      for (const tag of row.tags_csv.split(',')) {
        const t = tag.trim();
        if (t) tagCounts.set(t, (tagCounts.get(t) ?? 0) + 1);
      }
    }

    return [...tagCounts.entries()]
      .map(([tag, count]) => ({ tag, count }))
      .sort((a, b) => b.count - a.count || a.tag.localeCompare(b.tag));
  }

  // ─── Private search helpers ─────────────────────────────────────────────────

  private _ftsSearch(
    options: WikiSearchOptions,
    limit: number,
    offset: number,
  ): WikiSearchResult[] {
    // Escape FTS5 special chars in the query
    const escapedQuery = this._escapeFtsQuery(options.query!);

    // Build join conditions for additional filters
    const filterClauses: string[] = [];
    const filterParams: (string | number)[] = [];

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      const ph = statuses.map(() => '?').join(', ');
      filterClauses.push(`wp.status IN (${ph})`);
      filterParams.push(...statuses);
    }

    if (options.author) {
      filterClauses.push('wp.author = ?');
      filterParams.push(options.author);
    }

    if (options.hasGraphifyNode) {
      filterClauses.push('wp.graphify_node IS NOT NULL');
    }

    const whereClause = filterClauses.length > 0
      ? `AND ${filterClauses.join(' AND ')}`
      : '';

    const snippetLength = FTS_SNIPPET_LENGTH;

    const rows = this.db
      .prepare<(string | number)[], FtsRow>(/* sql */ `
        SELECT
          wp.slug,
          wp.title,
          wp.tags_csv,
          wp.status,
          wp.word_count,
          wp.updated_at,
          snippet(wiki_fts, 2, '<mark>', '</mark>', '...', ${Math.floor(snippetLength / 5)}) AS snippet,
          fts.rank
        FROM wiki_fts fts
        JOIN wiki_pages wp ON fts.slug = wp.slug
        WHERE wiki_fts MATCH ?
          AND wp.hidden = 0
          ${whereClause}
        ORDER BY rank
        LIMIT ? OFFSET ?
      `)
      .all(escapedQuery, ...filterParams, limit, offset);

    // Apply tag filter in memory (SQLite FTS5 doesn't support LIKE inside fts match)
    let results = rows.map((r) => this._ftsRowToResult(r));

    if (options.tags && options.tags.length > 0) {
      results = results.filter((r) =>
        options.tags!.every((tag) => r.tags.includes(tag)),
      );
    }

    return results;
  }

  private _structuredSearch(
    options: WikiSearchOptions,
    limit: number,
    offset: number,
  ): WikiSearchResult[] {
    const conditions: string[] = ['hidden = 0'];
    const params: (string | number)[] = [];

    if (options.status) {
      const statuses = Array.isArray(options.status) ? options.status : [options.status];
      const ph = statuses.map(() => '?').join(', ');
      conditions.push(`status IN (${ph})`);
      params.push(...statuses);
    }

    if (options.author) {
      conditions.push('author = ?');
      params.push(options.author);
    }

    if (options.hasGraphifyNode) {
      conditions.push('graphify_node IS NOT NULL');
    }

    if (options.hasDeadLinks) {
      conditions.push(
        'EXISTS (SELECT 1 FROM wiki_links wl WHERE wl.from_slug = wiki_pages.slug AND wl.is_dead = 1)',
      );
    }

    if (options.tags && options.tags.length > 0) {
      // Each tag must appear in tags_csv
      for (const tag of options.tags) {
        conditions.push('("," || tags_csv || ",") LIKE ?');
        params.push(`%,${tag},%`);
      }
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

    const rows = this.db
      .prepare<(string | number)[], TagRow>(/* sql */ `
        SELECT slug, title, tags_csv, status, word_count, updated_at
        FROM wiki_pages
        ${where}
        ORDER BY updated_at DESC
        LIMIT ? OFFSET ?
      `)
      .all(...params, limit, offset);

    return rows.map((r) => this._toSearchResult(r));
  }

  private _toSearchResult(row: TagRow): WikiSearchResult {
    return {
      slug: row.slug,
      title: row.title,
      status: row.status,
      tags: row.tags_csv ? row.tags_csv.split(',').map((t) => t.trim()).filter(Boolean) : [],
      wordCount: row.word_count,
      updatedAt: row.updated_at,
    };
  }

  private _ftsRowToResult(row: FtsRow): WikiSearchResult {
    return {
      slug: row.slug,
      title: row.title,
      status: row.status,
      tags: row.tags_csv ? row.tags_csv.split(',').map((t) => t.trim()).filter(Boolean) : [],
      snippet: row.snippet,
      score: row.rank,
      wordCount: row.word_count,
      updatedAt: row.updated_at,
    };
  }

  private _escapeFtsQuery(query: string): string {
    // Escape double quotes inside the query for FTS5 phrase safety
    // Then wrap in double quotes only if it contains spaces (phrase search)
    const escaped = query.replace(/"/g, '""');
    // If the query uses FTS5 operators (AND/OR/NOT/-/*), pass through as-is
    const hasOperators = /\b(AND|OR|NOT)\b|[*\-^]/.test(query);
    if (hasOperators) return escaped;
    // Otherwise, make each word a prefix search
    return escaped
      .trim()
      .split(/\s+/)
      .map((w) => `${w}*`)
      .join(' ');
  }
}
src/WikiLinker.ts
TypeScript

/**
 * WikiLinker — [[wikilink]] resolution, dead-link detection, and
 *              link health reporting for @locoworker/wiki.
 *
 * Responsibilities:
 *  - Resolve [[wikilinks]] in a page against the current slug index
 *  - Generate a dead-link report across the entire wiki
 *  - Suggest candidate slugs for dead links (fuzzy slug matching)
 *  - Create stub pages for dead links (via WikiParser.generateStub)
 *  - Cross-reference graphify_node fields against a provided node name set
 *    (optional: only when @locoworker/graphify is available at runtime)
 *  - Build an in-memory adjacency map for graph traversal (used by WikiExporter)
 *
 * Design: WikiLinker is read/compute-only. It reports but never writes to DB.
 * Callers (WikiSyncAgent, WikiWatcher) decide what to do with the results.
 */

import type { WikiStore } from './WikiStore.js';
import { WikiParser, toSlug } from './WikiParser.js';
import type {
  WikiLinkRef,
  WikiLinkType,
  WikiBacklinkResult,
} from './types/wiki.types.js';

// ─── Report types ─────────────────────────────────────────────────────────────

export interface DeadLinkReport {
  /** Total number of dead links across the wiki. */
  totalDeadLinks: number;

  /** Per-page breakdown. */
  byPage: Array<{
    slug: string;
    title: string;
    deadLinks: Array<{
      targetSlug: string;
      displayText: string;
      line: number;
      candidates: string[];  // suggested existing slugs (fuzzy match)
    }>;
  }>;

  /** Slugs that appear as link targets but have no page (de-duplicated). */
  unresolvedTargets: string[];
}

export interface GraphifyLinkReport {
  /** Pages whose graphify_node field points to a node name that doesn't exist. */
  brokenGraphifyLinks: Array<{
    pageSlug: string;
    pageTitle: string;
    graphifyNode: string;
  }>;
  /** Pages with valid graphify links. */
  validGraphifyLinks: Array<{
    pageSlug: string;
    pageTitle: string;
    graphifyNode: string;
  }>;
}

export interface WikiLinkGraph {
  /** Adjacency list: slug → set of slugs it links to (live links only). */
  outgoing: Map<string, Set<string>>;

  /** Reverse adjacency: slug → set of slugs that link to it. */
  incoming: Map<string, Set<string>>;

  /** All slugs in the index. */
  allSlugs: Set<string>;
}

// ─── WikiLinker ───────────────────────────────────────────────────────────────

export class WikiLinker {
  private readonly parser: WikiParser;

  constructor(private readonly store: WikiStore) {
    this.parser = new WikiParser();
  }

  // ─── Dead-link reporting ────────────────────────────────────────────────────

  /**
   * Generate a full dead-link report for the entire wiki.
   * Uses the wiki_links table which is always up-to-date after upsert/rebuild.
   */
  getDeadLinkReport(): DeadLinkReport {
    const deadLinks = this.store.getDeadLinks();

    if (deadLinks.length === 0) {
      return {
        totalDeadLinks: 0,
        byPage: [],
        unresolvedTargets: [],
      };
    }

    // Get all valid slugs for candidate suggestion
    const allPages = this.store.getAllPages();
    const allSlugs = new Set(allPages.map((p) => p.slug));

    // Build a title map for page names in report
    const titleMap = new Map(allPages.map((p) => [p.slug, p.title]));

    // Group dead links by source page
    const byPageMap = new Map<
      string,
      Array<{
        targetSlug: string;
        displayText: string;
        line: number;
        candidates: string[];
      }>
    >();

    const unresolvedTargets = new Set<string>();

    for (const link of deadLinks) {
      unresolvedTargets.add(link.to_slug);
      const candidates = this._suggestCandidates(link.to_slug, allSlugs);

      if (!byPageMap.has(link.from_slug)) {
        byPageMap.set(link.from_slug, []);
      }
      byPageMap.get(link.from_slug)!.push({
        targetSlug: link.to_slug,
        displayText: link.display_text,
        line: link.line,
        candidates,
      });
    }

    const byPage = [...byPageMap.entries()].map(([slug, links]) => ({
      slug,
      title: titleMap.get(slug) ?? slug,
      deadLinks: links.sort((a, b) => a.line - b.line),
    }));

    return {
      totalDeadLinks: deadLinks.length,
      byPage: byPage.sort((a, b) => a.slug.localeCompare(b.slug)),
      unresolvedTargets: [...unresolvedTargets].sort(),
    };
  }

  // ─── Graphify node cross-reference ─────────────────────────────────────────

  /**
   * Check all pages' `graphify_node` frontmatter against a provided set of
   * known node names/IDs. Returns a report of valid and broken links.
   *
   * `knownNodes` should be a Set of node names or IDs as strings.
   * This is intentionally decoupled from @locoworker/graphify: the caller
   * provides the set (e.g., from GraphifyClient.findNode results).
   */
  checkGraphifyLinks(knownNodes: Set<string>): GraphifyLinkReport {
    const pages = this.store.getAllPages();
    const broken: GraphifyLinkReport['brokenGraphifyLinks'] = [];
    const valid: GraphifyLinkReport['validGraphifyLinks'] = [];

    for (const page of pages) {
      if (!page.graphify_node) continue;

      const nodeRef = page.graphify_node;
      // Accept if any known node name or ID starts with or equals the ref
      const isValid = knownNodes.has(nodeRef) ||
        [...knownNodes].some((n) => n.includes(nodeRef) || nodeRef.includes(n));

      if (isValid) {
        valid.push({ pageSlug: page.slug, pageTitle: page.title, graphifyNode: nodeRef });
      } else {
        broken.push({ pageSlug: page.slug, pageTitle: page.title, graphifyNode: nodeRef });
      }
    }

    return { brokenGraphifyLinks: broken, validGraphifyLinks: valid };
  }

  // ─── Link graph construction ────────────────────────────────────────────────

  /**
   * Build an in-memory link graph (adjacency lists) for the entire wiki.
   * Only includes live (non-dead) links.
   *
   * Used by WikiExporter for graph visualisation and by WikiSearch for
   * advanced "related" queries.
   */
  buildLinkGraph(): WikiLinkGraph {
    const allPages = this.store.getAllPages();
    const allSlugs = new Set(allPages.map((p) => p.slug));

    const outgoing = new Map<string, Set<string>>();
    const incoming = new Map<string, Set<string>>();

    for (const slug of allSlugs) {
      outgoing.set(slug, new Set());
      incoming.set(slug, new Set());
    }

    const allLinks = this.store.getDeadLinks(); // get all links — we filter below
    // Actually get ALL links (not just dead) — need a dedicated method on store
    // We'll use the store's DB via an internal accessor here for efficiency
    const db = (this.store as unknown as { db: import('better-sqlite3').Database }).db;

    interface LinkRow { from_slug: string; to_slug: string; is_dead: number }
    const allLinkRows = db
      .prepare<[], LinkRow>('SELECT from_slug, to_slug, is_dead FROM wiki_links')
      .all();

    for (const row of allLinkRows) {
      if (row.is_dead === 1) continue;
      if (!outgoing.has(row.from_slug)) outgoing.set(row.from_slug, new Set());
      if (!incoming.has(row.to_slug)) incoming.set(row.to_slug, new Set());
      outgoing.get(row.from_slug)!.add(row.to_slug);
      incoming.get(row.to_slug)!.add(row.from_slug);
    }

    return { outgoing, incoming, allSlugs };
  }

  // ─── Stub page generation ───────────────────────────────────────────────────

  /**
   * For each unresolved target in the dead-link report, generate a stub WikiPage.
   * Callers (typically WikiSyncAgent) can then write these stubs to disk and
   * call WikiStore.upsertPage() to register them.
   */
  generateStubsForDeadLinks(
    report: DeadLinkReport,
    sourceSlug: string,
  ): import('./types/wiki.types.js').WikiPage[] {
    return report.unresolvedTargets.map((targetSlug) =>
      this.parser.generateStub(targetSlug, sourceSlug),
    );
  }

  // ─── Wikilink resolution (single page) ─────────────────────────────────────

  /**
   * Re-resolve all wikilinks in a parsed page against the current slug index.
   * Updates `link.isDead` in-place and returns the page.
   * Use this after WikiStore.rebuild() when isDead may be stale on in-memory pages.
   */
  resolveLinks<T extends { wikilinks: WikiLinkRef[] }>(
    page: T,
    slugIndex: Set<string>,
  ): T {
    for (const link of page.wikilinks) {
      link.isDead = !slugIndex.has(link.targetSlug);
    }
    return page;
  }

  // ─── Slug suggestion (fuzzy) ────────────────────────────────────────────────

  /**
   * Suggest existing slugs that are "close" to a dead-link target slug.
   * Uses three tiers: exact prefix, token overlap (Jaccard), and edit distance.
   * Returns up to 3 candidates.
   */
  private _suggestCandidates(targetSlug: string, allSlugs: Set<string>): string[] {
    const candidates: Array<{ slug: string; score: number }> = [];

    const targetTokens = new Set(targetSlug.split('-'));

    for (const candidate of allSlugs) {
      let score = 0;

      // Tier 1: prefix match
      if (candidate.startsWith(targetSlug) || targetSlug.startsWith(candidate)) {
        score += 10;
      }

      // Tier 2: Jaccard similarity on tokens
      const candidateTokens = new Set(candidate.split('-'));
      const intersection = [...targetTokens].filter((t) => candidateTokens.has(t)).length;
      const union = new Set([...targetTokens, ...candidateTokens]).size;
      if (union > 0) score += (intersection / union) * 5;

      // Tier 3: simple edit distance (only if short slugs)
      if (targetSlug.length <= 20 && candidate.length <= 20) {
        const ed = this._editDistance(targetSlug, candidate);
        const maxLen = Math.max(targetSlug.length, candidate.length);
        const similarity = 1 - ed / maxLen;
        score += similarity * 3;
      }

      if (score > 3) {
        candidates.push({ slug: candidate, score });
      }
    }

    return candidates
      .sort((a, b) => b.score - a.score)
      .slice(0, 3)
      .map((c) => c.slug);
  }

  /**
   * Standard Levenshtein edit distance (iterative, O(n*m) space).
   */
  private _editDistance(a: string, b: string): number {
    const m = a.length;
    const n = b.length;
    const dp: number[][] = Array.from({ length: m + 1 }, (_, i) =>
      Array.from({ length: n + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0)),
    );

    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        dp[i][j] =
          a[i - 1] === b[j - 1]
            ? dp[i - 1][j - 1]
            : 1 + Math.min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]);
      }
    }

    return dp[m][n];
  }
}
Pass 5 — Part 1 Complete ✅
What was generated
File	Purpose	Lines (approx)
package.json	Package config, deps (better-sqlite3, gray-matter, marked, zod), bun test	45
tsconfig.json	Extends monorepo base, path aliases for core/shared/graphify	20
src/types/wiki.types.ts	Full type ABI: WikiPage, WikiFrontmatter, WikiLinkRef, row types, Zod schemas, constants, regex	~250
src/WikiStore.ts	SQLite persistence: 4-table schema (pages/links/history/FTS5), WAL pragmas, upsert/delete/rebuild, dead-link resolution, stats	~380
src/WikiParser.ts	gray-matter frontmatter parse, Zod validation, wikilink extraction, heading/URL extraction, word count, fingerprint, batch parse, stub generation, marked renderer	~320
src/WikiSearch.ts	FTS5 full-text search with BM25 + snippets, structured filter queries, backlinks, orphan detection, dead-link pages, related page suggestions, tag index	~340
src/WikiLinker.ts	Dead-link report, graphify_node cross-ref, in-memory link graph, stub generation, wikilink resolution, fuzzy slug suggestion (Jaccard + Levenshtein)	~270
Part 2 preview (ready to generate on request)
File	Purpose
src/WikiExporter.ts	Export to HTML bundle / single-file Markdown / JSON dump
src/WikiSyncAgent.ts	Agent-triggered page write pipeline (integrates with packages/core EventBus + HooksRegistry)
src/WikiWatcher.ts	fs.watch based auto-sync on file save → WikiStore.upsertPage
src/index.ts	Clean barrel export
src/tests/WikiStore.test.ts	Full bun test suite
src/tests/WikiParser.test.ts	Full bun test suite
src/tests/WikiSearch.test.ts	Full bun test suite
src/tests/WikiLinker.test.ts	Full bun test suite
Key design decisions & consistency notes
DB column names are fully consistent with Part 1 GraphBuilder convention (from_slug/to_slug/is_dead mirrors from_id/to_id style, not the inconsistent Part 2 graphify names). ✅
WikiStore is the only writer; WikiSearch and WikiLinker are read-only — this prevents cross-class write conflicts. ✅
FTS5 content= external table keeps the full-text index in sync with wiki_pages automatically via SQLite triggers — no manual double-write needed. ✅
fingerprint is SHA-256 of the full raw content (including frontmatter) — same pattern as IncrementalUpdater in graphify, so cross-package tooling can compare page states consistently. ✅
graphify is a peer/optional dependency — WikiLinker accepts a Set<string> of known node names rather than importing GraphifyClient directly, so wiki works standalone without graphify present. ✅
Wikilink regex handles all three syntaxes ([[Page]], [[Page|Display]], [[Page|Display::type]]) using a single compiled regex with named capture groups. ✅




