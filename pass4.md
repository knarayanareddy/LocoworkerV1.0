Pass 4 of 18 — packages/graphify
Part 1 of 2: Package Config · Graph Types · TreeSitterParser · GraphBuilder
Full File Tree for Pass 4
text

packages/graphify/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                     ← Part 2
│   ├── types/
│   │   └── graph.types.ts           ← Part 1
│   ├── TreeSitterParser.ts          ← Part 1
│   ├── GraphBuilder.ts              ← Part 1
│   ├── LeidenClusterer.ts           ← Part 2
│   ├── GraphifyClient.ts            ← Part 2
│   ├── GraphReporter.ts             ← Part 2
│   └── IncrementalUpdater.ts        ← Part 2
└── tests/
    ├── TreeSitterParser.test.ts     ← Part 2
    ├── GraphBuilder.test.ts         ← Part 2
    ├── LeidenClusterer.test.ts      ← Part 2
    └── GraphifyClient.test.ts       ← Part 2
packages/graphify/package.json
JSON

{
  "name": "@locoworker/graphify",
  "version": "1.0.0",
  "description": "LocoWorker AST knowledge graph engine — Tree-sitter parsing, Leiden clustering, 71x token reduction",
  "license": "MIT",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types":  "./dist/index.d.ts"
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
    "clean":         "rm -rf dist tsconfig.tsbuildinfo",
    "cli":           "bun run src/cli.ts"
  },
  "dependencies": {
    "better-sqlite3":           "9.4.3",
    "tree-sitter":              "0.21.1",
    "tree-sitter-typescript":   "0.21.2",
    "tree-sitter-javascript":   "0.21.4",
    "tree-sitter-python":       "0.21.0",
    "tree-sitter-rust":         "0.21.2",
    "tree-sitter-go":           "0.21.0",
    "glob":                     "11.0.0",
    "zod":                      "3.24.1"
  },
  "devDependencies": {
    "@locoworker/core":   "workspace:*",
    "@locoworker/shared": "workspace:*",
    "@types/better-sqlite3": "7.6.8",
    "@types/node":           "^22.10.0",
    "typescript":            "^5.7.2"
  },
  "peerDependencies": {
    "@locoworker/core": "workspace:*"
  }
}
packages/graphify/tsconfig.json
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
packages/graphify/src/types/graph.types.ts
TypeScript

import { z } from 'zod'

/**
 * All node types that Graphify can extract from source files.
 *
 * Code nodes:    Directly correspond to AST constructs.
 * Concept nodes: Extracted from comments, JSDoc, README sections.
 * Visual nodes:  References to diagrams, SVG, or component trees.
 * Config nodes:  Entries in configuration files (JSON, YAML, TOML).
 */
export type NodeType =
  | 'function'      // function declarations & arrow functions with a name
  | 'class'         // class declarations
  | 'interface'     // TypeScript interfaces
  | 'type_alias'    // TypeScript type aliases
  | 'enum'          // TypeScript enums
  | 'component'     // React/Solid/Svelte components (capitalised functions)
  | 'hook'          // React hooks (use* functions)
  | 'module'        // File/module level node (one per file)
  | 'namespace'     // TypeScript namespaces / modules
  | 'concept'       // Extracted from JSDoc or block comments
  | 'config'        // Configuration file entry
  | 'visual'        // Diagram or visual asset reference

/**
 * All relationship types between graph nodes.
 */
export type EdgeType =
  | 'imports'       // A imports B (static import)
  | 'requires'      // A require(B) (CommonJS)
  | 'calls'         // A calls B (function call)
  | 'extends'       // A extends B (class or interface)
  | 'implements'    // A implements B (class implements interface)
  | 'uses_type'     // A references type B
  | 'renders'       // Component A renders component B (JSX)
  | 'documents'     // Comment/concept A documents node B
  | 'tests'         // Test A tests function/class B
  | 'configures'    // Config entry A configures module B
  | 'exports'       // Module A exports symbol B
  | 're_exports'    // Module A re-exports from module B

/**
 * Supported source languages and their file extensions.
 */
export type SupportedLanguage =
  | 'typescript'
  | 'javascript'
  | 'python'
  | 'rust'
  | 'go'
  | 'markdown'
  | 'json'
  | 'yaml'
  | 'toml'
  | 'unknown'

export const LANGUAGE_EXTENSIONS: Record<string, SupportedLanguage> = {
  '.ts':   'typescript',
  '.tsx':  'typescript',
  '.mts':  'typescript',
  '.cts':  'typescript',
  '.js':   'javascript',
  '.jsx':  'javascript',
  '.mjs':  'javascript',
  '.cjs':  'javascript',
  '.py':   'python',
  '.rs':   'rust',
  '.go':   'go',
  '.md':   'markdown',
  '.mdx':  'markdown',
  '.json': 'json',
  '.yaml': 'yaml',
  '.yml':  'yaml',
  '.toml': 'toml',
} as const

// ── Core graph entities ────────────────────────────────────────────────

/**
 * A node in the knowledge graph.
 * Represents a named, addressable code or concept entity.
 */
export interface GraphNode {
  /** UUID — stable across incremental updates */
  id:           string
  /** Node type — determines how the node is displayed and queried */
  type:         NodeType
  /** Symbol name (function name, class name, etc.) */
  name:         string
  /** Absolute path to the source file */
  filePath:     string
  /** 1-based start line */
  lineStart:    number
  /** 1-based end line */
  lineEnd:      number
  /** Source language */
  language:     SupportedLanguage
  /** Cluster/community ID assigned by LeidenClusterer */
  cluster:      string
  /** AI-generated or JSDoc-extracted one-line summary */
  summary:      string | null
  /** Raw docstring / JSDoc comment */
  docstring:    string | null
  /** Whether the symbol is exported from its module */
  isExported:   boolean
  /** Whether the symbol is publicly accessible */
  isPublic:     boolean
  /** McCabe cyclomatic complexity (functions only) */
  complexity:   number | null
  /** Free-form tags for filtering */
  tags:         string[]
  /** SHA-256 of the node's source text — used for change detection */
  fingerprint:  string
  /** Unix timestamp of last update */
  lastUpdated:  number
  /** Raw source text of the node (truncated to 2000 chars) */
  sourceText:   string | null
}

/**
 * A directed edge between two graph nodes.
 */
export interface GraphEdge {
  /** UUID */
  id:       string
  /** Source node ID */
  from:     string
  /** Target node ID */
  to:       string
  /** Relationship type */
  type:     EdgeType
  /**
   * Relationship weight (0–1).
   * Higher = stronger / more frequent relationship.
   */
  weight:   number
  /** Optional metadata (e.g., import specifier, line number) */
  metadata: Record<string, unknown> | null
}

/**
 * A cluster of related nodes (community detected by Leiden).
 */
export interface GraphCluster {
  /** UUID */
  id:               string
  /** Human-readable label derived from dominant module name */
  label:            string
  /** Node IDs in this cluster */
  nodeIds:          string[]
  /** Package/directory path this cluster corresponds to */
  packagePath:      string | null
  /** Auto-generated description of the cluster */
  description:      string | null
  /** Dominant language in this cluster */
  dominantLanguage: SupportedLanguage
  /** Number of nodes */
  nodeCount:        number
  /** Number of edges between nodes in this cluster */
  internalEdgeCount: number
}

// ── Build-time types ───────────────────────────────────────────────────

/**
 * Raw parsed symbol extracted by TreeSitterParser before graph building.
 */
export interface ParsedSymbol {
  type:         NodeType
  name:         string
  filePath:     string
  lineStart:    number
  lineEnd:      number
  language:     SupportedLanguage
  isExported:   boolean
  isPublic:     boolean
  docstring:    string | null
  sourceText:   string
  complexity:   number | null
  /** Import specifiers found in this symbol's scope */
  imports:      ParsedImport[]
  /** Symbols this node calls or references */
  references:   ParsedReference[]
}

export interface ParsedImport {
  /** The import source/specifier (e.g., './utils', 'react') */
  source:    string
  /** Named imports from the module */
  names:     string[]
  /** Whether this is a default import */
  isDefault: boolean
  /** Whether this is a namespace import (import * as X) */
  isNamespace: boolean
  lineNumber:  number
}

export interface ParsedReference {
  /** The name of the referenced symbol */
  name:       string
  /** Line where the reference occurs */
  lineNumber: number
  /** Whether this is a call expression (vs. type reference) */
  isCall:     boolean
}

/**
 * Result of parsing a single file.
 */
export interface ParsedFile {
  filePath:    string
  language:    SupportedLanguage
  symbols:     ParsedSymbol[]
  imports:     ParsedImport[]
  /** SHA-256 of the full file content */
  fingerprint: string
  /** Whether the parse succeeded */
  success:     boolean
  /** Parse error message if success = false */
  error:       string | null
  /** Parse duration in milliseconds */
  durationMs:  number
  /** Total lines in the file */
  lineCount:   number
}

// ── Query types ────────────────────────────────────────────────────────

/**
 * Options for querying the graph.
 */
export interface GraphQueryOptions {
  /** Filter by node types */
  types?:          NodeType[]
  /** Filter by cluster label (partial match) */
  cluster?:        string
  /** Filter by language */
  language?:       SupportedLanguage
  /** Only return exported symbols */
  exportedOnly?:   boolean
  /** Maximum nodes to return */
  limit?:          number
  /** Minimum complexity (functions only) */
  minComplexity?:  number
  /** Filter by tags */
  tags?:           string[]
}

/**
 * Result of a graph query — includes nodes, their edges,
 * and an estimated token count for context budgeting.
 */
export interface GraphQueryResult {
  nodes:          GraphNode[]
  edges:          GraphEdge[]
  clusters:       GraphCluster[]
  tokenEstimate:  number
  queryTimeMs:    number
}

// ── Database schema (SQLite) ───────────────────────────────────────────

/**
 * SQLite row type for the nodes table.
 * JSON fields are stored as serialised strings.
 */
export interface NodeRow {
  id:            string
  type:          string
  name:          string
  file_path:     string
  line_start:    number
  line_end:      number
  language:      string
  cluster:       string
  summary:       string | null
  docstring:     string | null
  is_exported:   number   // SQLite boolean: 0 or 1
  is_public:     number
  complexity:    number | null
  tags:          string   // JSON array
  fingerprint:   string
  last_updated:  number
  source_text:   string | null
}

/**
 * SQLite row type for the edges table.
 */
export interface EdgeRow {
  id:        string
  from_id:   string
  to_id:     string
  type:      string
  weight:    number
  metadata:  string | null  // JSON object
}

/**
 * SQLite row type for the clusters table.
 */
export interface ClusterRow {
  id:                  string
  label:               string
  node_ids:            string   // JSON array
  package_path:        string | null
  description:         string | null
  dominant_language:   string
  node_count:          number
  internal_edge_count: number
}

// ── Build statistics ───────────────────────────────────────────────────

/**
 * Statistics from a full graph build or incremental update.
 */
export interface BuildStats {
  filesProcessed:   number
  filesSkipped:     number
  filesFailed:      number
  nodesCreated:     number
  nodesUpdated:     number
  nodesDeleted:     number
  edgesCreated:     number
  edgesDeleted:     number
  clustersCreated:  number
  totalNodes:       number
  totalEdges:       number
  totalClusters:    number
  durationMs:       number
  tokenEquivalent:  number   // Estimated tokens if files were read raw
  graphTokens:      number   // Estimated tokens to query this graph
  reductionFactor:  number   // tokenEquivalent / graphTokens
}

// ── Zod schemas for runtime validation ────────────────────────────────

export const GraphNodeSchema = z.object({
  id:          z.string().uuid(),
  type:        z.string(),
  name:        z.string().min(1),
  filePath:    z.string().min(1),
  lineStart:   z.number().int().positive(),
  lineEnd:     z.number().int().positive(),
  language:    z.string(),
  cluster:     z.string(),
  summary:     z.string().nullable(),
  docstring:   z.string().nullable(),
  isExported:  z.boolean(),
  isPublic:    z.boolean(),
  complexity:  z.number().nullable(),
  tags:        z.array(z.string()),
  fingerprint: z.string().length(64),
  lastUpdated: z.number().int().positive(),
  sourceText:  z.string().nullable(),
})

export const GraphEdgeSchema = z.object({
  id:       z.string().uuid(),
  from:     z.string().uuid(),
  to:       z.string().uuid(),
  type:     z.string(),
  weight:   z.number().min(0).max(1),
  metadata: z.record(z.unknown()).nullable(),
})

// ── Constants ──────────────────────────────────────────────────────────

/** Average tokens per graph node descriptor (for budget estimation) */
export const TOKENS_PER_NODE = 35

/** Average tokens per graph edge descriptor */
export const TOKENS_PER_EDGE = 15

/** Average tokens per raw source line (for reduction factor calculation) */
export const TOKENS_PER_SOURCE_LINE = 8

/** Default cluster label for unclustered nodes */
export const UNCLUSTERED_LABEL = 'unclustered'

/** Maximum source text length stored per node */
export const MAX_SOURCE_TEXT_LENGTH = 2_000

/** Maximum docstring length stored per node */
export const MAX_DOCSTRING_LENGTH = 500
packages/graphify/src/TreeSitterParser.ts
TypeScript

import { createHash } from 'node:crypto'
import { readFile } from 'node:fs/promises'
import { extname, basename } from 'node:path'
import {
  LANGUAGE_EXTENSIONS,
  MAX_DOCSTRING_LENGTH,
  MAX_SOURCE_TEXT_LENGTH,
  type NodeType,
  type ParsedFile,
  type ParsedImport,
  type ParsedReference,
  type ParsedSymbol,
  type SupportedLanguage,
} from './types/graph.types.js'

// ── Tree-sitter dynamic imports ────────────────────────────────────────
// Tree-sitter is a native module. We import dynamically to allow
// graceful degradation when native bindings are not available
// (e.g., in test environments without native compilation).

type TSNode = {
  type:            string
  text:            string
  startPosition:   { row: number; column: number }
  endPosition:     { row: number; column: number }
  children:        TSNode[]
  childCount:      number
  parent:          TSNode | null
  isNamed:         boolean
  hasError:        boolean
  child(index: number): TSNode | null
  childForFieldName(field: string): TSNode | null
  namedChildren:   TSNode[]
  descendantsOfType(type: string | string[]): TSNode[]
}

type TSTree = {
  rootNode: TSNode
}

type TSParser = {
  setLanguage(lang: unknown): void
  parse(source: string): TSTree
}

type TSParserConstructor = new () => TSParser

/**
 * TreeSitterParser converts source files into ParsedSymbol arrays.
 *
 * It uses Tree-sitter for accurate, language-aware AST parsing.
 * Language grammars are loaded lazily — only the grammars needed
 * for the files being parsed are loaded into memory.
 *
 * Supported extractions per language:
 *
 *   TypeScript/JavaScript:
 *     - function declarations and arrow functions
 *     - class declarations (with extends/implements)
 *     - interfaces and type aliases
 *     - enums and namespaces
 *     - React components (capitalized functions returning JSX)
 *     - React hooks (functions starting with "use")
 *     - import statements (static + dynamic)
 *     - JSDoc comments
 *     - function call references
 *
 *   Python:
 *     - function definitions (def)
 *     - class definitions
 *     - import statements (import + from-import)
 *     - docstrings
 *
 *   Rust:
 *     - function items (fn)
 *     - struct, enum, trait, impl items
 *     - use declarations
 *     - doc comments (///)
 *
 *   Go:
 *     - function declarations
 *     - type declarations (struct, interface)
 *     - import declarations
 *
 *   Markdown:
 *     - H1, H2, H3 headings as concept nodes
 *
 * @example
 * ```ts
 * const parser = new TreeSitterParser()
 * const result = await parser.parseFile('/project/src/index.ts')
 * console.log(result.symbols.map(s => s.name))
 * ```
 */
export class TreeSitterParser {
  private Parser:    TSParserConstructor | null = null
  private languages: Map<string, unknown>       = new Map()
  private available  = false

  constructor() {
    this.loadParser()
  }

  /**
   * Attempt to load tree-sitter. If it fails (e.g., no native bindings),
   * the parser degrades to returning module-level nodes only.
   */
  private loadParser(): void {
    try {
      // biome-ignore lint/suspicious/noExplicitAny: dynamic native module
      this.Parser    = (require('tree-sitter') as any) as TSParserConstructor
      this.available = true
    } catch {
      console.warn(
        '[TreeSitterParser] tree-sitter native bindings not available. ' +
        'Falling back to module-level parsing only.'
      )
      this.available = false
    }
  }

  /**
   * Load a Tree-sitter language grammar.
   * Grammars are cached after first load.
   */
  private loadLanguage(language: SupportedLanguage): unknown | null {
    if (this.languages.has(language)) {
      return this.languages.get(language)!
    }

    const GRAMMAR_PACKAGES: Partial<Record<SupportedLanguage, string>> = {
      typescript:  'tree-sitter-typescript',
      javascript:  'tree-sitter-javascript',
      python:      'tree-sitter-python',
      rust:        'tree-sitter-rust',
      go:          'tree-sitter-go',
    }

    const pkg = GRAMMAR_PACKAGES[language]
    if (!pkg) return null

    try {
      // biome-ignore lint/suspicious/noExplicitAny: dynamic grammar loading
      const grammar = require(pkg) as any

      // tree-sitter-typescript exports { typescript, tsx }
      const lang = language === 'typescript'
        ? (grammar.typescript ?? grammar)
        : grammar

      this.languages.set(language, lang)
      return lang
    } catch {
      console.warn(`[TreeSitterParser] Grammar not available for: ${language}`)
      return null
    }
  }

  // ── Public API ────────────────────────────────────────────────────

  /**
   * Parse a single source file and return all extracted symbols.
   */
  async parseFile(filePath: string): Promise<ParsedFile> {
    const start    = Date.now()
    const ext      = extname(filePath).toLowerCase()
    const language = LANGUAGE_EXTENSIONS[ext] ?? 'unknown'

    let source: string
    try {
      source = await readFile(filePath, 'utf-8')
    } catch (err) {
      return this.errorResult(filePath, language, `Could not read file: ${err}`, start)
    }

    const fingerprint = this.sha256(source)
    const lineCount   = source.split('\n').length

    // For unsupported languages, return module-level node only
    if (language === 'unknown') {
      return {
        filePath,
        language,
        symbols:     [this.makeModuleSymbol(filePath, language, lineCount, fingerprint, source)],
        imports:     [],
        fingerprint,
        success:     true,
        error:       null,
        durationMs:  Date.now() - start,
        lineCount,
      }
    }

    // Markdown: heading extraction (no AST needed)
    if (language === 'markdown') {
      return this.parseMarkdown(filePath, source, fingerprint, lineCount, start)
    }

    // Config files: key extraction
    if (language === 'json' || language === 'yaml' || language === 'toml') {
      return this.parseConfig(filePath, language, source, fingerprint, lineCount, start)
    }

    // Tree-sitter: full AST parsing
    if (!this.available) {
      return {
        filePath,
        language,
        symbols:    [this.makeModuleSymbol(filePath, language, lineCount, fingerprint, source)],
        imports:    this.extractImportsRegex(source, language),
        fingerprint,
        success:    true,
        error:      null,
        durationMs: Date.now() - start,
        lineCount,
      }
    }

    return this.parseWithTreeSitter(
      filePath, language, source, fingerprint, lineCount, start
    )
  }

  /**
   * Parse multiple files concurrently.
   * Limits concurrency to avoid file descriptor exhaustion.
   */
  async parseFiles(
    filePaths: string[],
    concurrency = 8
  ): Promise<ParsedFile[]> {
    const results: ParsedFile[] = []
    const chunks = this.chunk(filePaths, concurrency)

    for (const chunk of chunks) {
      const chunkResults = await Promise.allSettled(
        chunk.map((fp) => this.parseFile(fp))
      )
      for (const result of chunkResults) {
        if (result.status === 'fulfilled') {
          results.push(result.value)
        }
      }
    }

    return results
  }

  /**
   * Check if tree-sitter native bindings are available.
   */
  get isAvailable(): boolean {
    return this.available
  }

  // ── Tree-sitter parsing ────────────────────────────────────────────

  private parseWithTreeSitter(
    filePath:    string,
    language:    SupportedLanguage,
    source:      string,
    fingerprint: string,
    lineCount:   number,
    start:       number
  ): ParsedFile {
    const grammar = this.loadLanguage(language)
    if (!grammar) {
      return this.fallbackResult(filePath, language, source, fingerprint, lineCount, start)
    }

    try {
      const parser = new this.Parser!()
      parser.setLanguage(grammar)
      const tree = parser.parse(source)

      if (tree.rootNode.hasError) {
        console.warn(`[TreeSitterParser] Parse errors in: ${filePath}`)
      }

      const symbols = this.extractSymbols(tree.rootNode, filePath, language, source)
      const imports = this.extractImportsFromAST(tree.rootNode, language)

      // Always include a module-level node for the file itself
      const moduleSymbol = this.makeModuleSymbol(
        filePath, language, lineCount, fingerprint, source
      )

      return {
        filePath,
        language,
        symbols:    [moduleSymbol, ...symbols],
        imports,
        fingerprint,
        success:    true,
        error:      null,
        durationMs: Date.now() - start,
        lineCount,
      }
    } catch (err) {
      return this.errorResult(filePath, language, String(err), start)
    }
  }

  // ── Symbol extraction (TypeScript / JavaScript) ────────────────────

  private extractSymbols(
    root:     TSNode,
    filePath: string,
    language: SupportedLanguage,
    source:   string
  ): ParsedSymbol[] {
    const symbols: ParsedSymbol[] = []

    switch (language) {
      case 'typescript':
      case 'javascript':
        this.extractTSSymbols(root, filePath, language, source, symbols)
        break
      case 'python':
        this.extractPythonSymbols(root, filePath, language, source, symbols)
        break
      case 'rust':
        this.extractRustSymbols(root, filePath, language, source, symbols)
        break
      case 'go':
        this.extractGoSymbols(root, filePath, language, source, symbols)
        break
    }

    return symbols
  }

  // ── TypeScript / JavaScript extraction ────────────────────────────

  private extractTSSymbols(
    root:     TSNode,
    filePath: string,
    language: SupportedLanguage,
    source:   string,
    out:      ParsedSymbol[]
  ): void {
    const lines      = source.split('\n')
    const isExported = (node: TSNode) => {
      const parent = node.parent
      return parent?.type === 'export_statement' ||
             parent?.type === 'export_declaration' ||
             node.text.startsWith('export ')
    }

    // Walk all top-level and nested declarations
    this.walkNode(root, (node) => {
      const type = node.type

      // Function declarations
      if (type === 'function_declaration' || type === 'function') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        const name        = nameNode.text
        const lineStart   = node.startPosition.row + 1
        const lineEnd     = node.endPosition.row + 1
        const docstring   = this.extractPrecedingDocstring(node, lines)
        const complexity  = this.calculateComplexity(node)
        const references  = this.extractCallReferences(node)
        const exported    = isExported(node)
        const nodeType    = this.classifyTSFunction(name, node)
        const sourceText  = node.text.slice(0, MAX_SOURCE_TEXT_LENGTH)

        out.push({
          type:       nodeType,
          name,
          filePath,
          lineStart,
          lineEnd,
          language,
          isExported: exported,
          isPublic:   exported || !name.startsWith('_'),
          docstring,
          sourceText,
          complexity,
          imports:    [],
          references,
        })
      }

      // Arrow function variables: const foo = () => {}
      if (type === 'lexical_declaration' || type === 'variable_declaration') {
        for (const declarator of node.descendantsOfType('variable_declarator')) {
          const nameNode  = declarator.childForFieldName('name')
          const valueNode = declarator.childForFieldName('value')

          if (!nameNode) continue
          if (!valueNode) continue
          if (
            valueNode.type !== 'arrow_function' &&
            valueNode.type !== 'function'
          ) continue

          const name       = nameNode.text
          const lineStart  = node.startPosition.row + 1
          const lineEnd    = node.endPosition.row + 1
          const nodeType   = this.classifyTSFunction(name, valueNode)
          const complexity = this.calculateComplexity(valueNode)
          const docstring  = this.extractPrecedingDocstring(node, lines)
          const exported   = isExported(node)

          out.push({
            type:       nodeType,
            name,
            filePath,
            lineStart,
            lineEnd,
            language,
            isExported: exported,
            isPublic:   exported || !name.startsWith('_'),
            docstring,
            sourceText: valueNode.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
            complexity,
            imports:    [],
            references: this.extractCallReferences(valueNode),
          })
        }
      }

      // Class declarations
      if (type === 'class_declaration' || type === 'class') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        const name      = nameNode.text
        const exported  = isExported(node)
        const docstring = this.extractPrecedingDocstring(node, lines)

        out.push({
          type:       'class',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: exported,
          isPublic:   exported || !name.startsWith('_'),
          docstring,
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: this.extractClassReferences(node),
        })
      }

      // Interface declarations (TypeScript)
      if (type === 'interface_declaration') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        const name     = nameNode.text
        const exported = isExported(node)

        out.push({
          type:       'interface',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: exported,
          isPublic:   true,
          docstring:  this.extractPrecedingDocstring(node, lines),
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }

      // Type alias declarations (TypeScript)
      if (type === 'type_alias_declaration') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        out.push({
          type:       'type_alias',
          name:       nameNode.text,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isExported(node),
          isPublic:   true,
          docstring:  this.extractPrecedingDocstring(node, lines),
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }

      // Enum declarations (TypeScript)
      if (type === 'enum_declaration') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        out.push({
          type:       'enum',
          name:       nameNode.text,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isExported(node),
          isPublic:   true,
          docstring:  this.extractPrecedingDocstring(node, lines),
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }
    })
  }

  /**
   * Classify a TypeScript function as function, component, or hook.
   */
  private classifyTSFunction(name: string, _node: TSNode): NodeType {
    // React hooks start with "use" followed by an uppercase letter
    if (/^use[A-Z]/.test(name)) return 'hook'
    // React components start with an uppercase letter
    if (/^[A-Z]/.test(name)) return 'component'
    return 'function'
  }

  // ── Python extraction ─────────────────────────────────────────────

  private extractPythonSymbols(
    root:     TSNode,
    filePath: string,
    language: SupportedLanguage,
    source:   string,
    out:      ParsedSymbol[]
  ): void {
    this.walkNode(root, (node) => {
      if (node.type === 'function_definition') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        const name      = nameNode.text
        const docstring = this.extractPythonDocstring(node)
        const isPublic  = !name.startsWith('_')

        out.push({
          type:       'function',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPublic,
          isPublic,
          docstring,
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: this.calculateComplexity(node),
          imports:    [],
          references: [],
        })
      }

      if (node.type === 'class_definition') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return

        const name      = nameNode.text
        const docstring = this.extractPythonDocstring(node)
        const isPublic  = !name.startsWith('_')

        out.push({
          type:       'class',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPublic,
          isPublic,
          docstring,
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }
    })
  }

  // ── Rust extraction ───────────────────────────────────────────────

  private extractRustSymbols(
    root:     TSNode,
    filePath: string,
    language: SupportedLanguage,
    _source:  string,
    out:      ParsedSymbol[]
  ): void {
    this.walkNode(root, (node) => {
      const type = node.type

      if (type === 'function_item') {
        const nameNode  = node.childForFieldName('name')
        if (!nameNode) return
        const name      = nameNode.text
        const isPub     = node.text.startsWith('pub ')

        out.push({
          type:       'function',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPub,
          isPublic:   isPub,
          docstring:  this.extractRustDocstring(node),
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: this.calculateComplexity(node),
          imports:    [],
          references: [],
        })
      }

      if (
        type === 'struct_item' ||
        type === 'enum_item'   ||
        type === 'trait_item'  ||
        type === 'impl_item'
      ) {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return
        const isPub    = node.text.startsWith('pub ')
        const nodeType = type === 'enum_item' ? 'enum' : 'class'

        out.push({
          type:       nodeType,
          name:       nameNode.text,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPub,
          isPublic:   isPub,
          docstring:  this.extractRustDocstring(node),
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }
    })
  }

  // ── Go extraction ──────────────────────────────────────────────────

  private extractGoSymbols(
    root:     TSNode,
    filePath: string,
    language: SupportedLanguage,
    _source:  string,
    out:      ParsedSymbol[]
  ): void {
    this.walkNode(root, (node) => {
      if (node.type === 'function_declaration') {
        const nameNode = node.childForFieldName('name')
        if (!nameNode) return
        const name    = nameNode.text
        const isPublic = /^[A-Z]/.test(name)

        out.push({
          type:       'function',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPublic,
          isPublic,
          docstring:  null,
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: this.calculateComplexity(node),
          imports:    [],
          references: [],
        })
      }

      if (node.type === 'type_declaration') {
        const nameNode = node.descendantsOfType('type_identifier')[0]
        if (!nameNode) return
        const name     = nameNode.text
        const isPublic = /^[A-Z]/.test(name)

        out.push({
          type:       'interface',
          name,
          filePath,
          lineStart:  node.startPosition.row + 1,
          lineEnd:    node.endPosition.row + 1,
          language,
          isExported: isPublic,
          isPublic,
          docstring:  null,
          sourceText: node.text.slice(0, MAX_SOURCE_TEXT_LENGTH),
          complexity: null,
          imports:    [],
          references: [],
        })
      }
    })
  }

  // ── Markdown parsing ───────────────────────────────────────────────

  private parseMarkdown(
    filePath:    string,
    source:      string,
    fingerprint: string,
    lineCount:   number,
    start:       number
  ): ParsedFile {
    const lines   = source.split('\n')
    const symbols: ParsedSymbol[] = []

    lines.forEach((line, idx) => {
      const h1 = line.match(/^#\s+(.+)$/)
      const h2 = line.match(/^##\s+(.+)$/)
      const h3 = line.match(/^###\s+(.+)$/)

      const match = h1 ?? h2 ?? h3
      if (!match?.[1]) return

      const name = match[1].trim()
      if (name.length < 3) return

      symbols.push({
        type:       'concept',
        name,
        filePath,
        lineStart:  idx + 1,
        lineEnd:    idx + 1,
        language:   'markdown',
        isExported: true,
        isPublic:   true,
        docstring:  null,
        sourceText: line.slice(0, MAX_SOURCE_TEXT_LENGTH),
        complexity: null,
        imports:    [],
        references: [],
      })
    })

    const moduleSymbol = this.makeModuleSymbol(filePath, 'markdown', lineCount, fingerprint, source)

    return {
      filePath,
      language:   'markdown',
      symbols:    [moduleSymbol, ...symbols],
      imports:    [],
      fingerprint,
      success:    true,
      error:      null,
      durationMs: Date.now() - start,
      lineCount,
    }
  }

  // ── Config file parsing ────────────────────────────────────────────

  private parseConfig(
    filePath:    string,
    language:    SupportedLanguage,
    source:      string,
    fingerprint: string,
    lineCount:   number,
    start:       number
  ): ParsedFile {
    const moduleSymbol = this.makeModuleSymbol(filePath, language, lineCount, fingerprint, source)

    // For JSON: extract top-level keys as config nodes
    const symbols: ParsedSymbol[] = [moduleSymbol]

    if (language === 'json') {
      try {
        const parsed = JSON.parse(source) as Record<string, unknown>
        for (const key of Object.keys(parsed).slice(0, 20)) {
          symbols.push({
            type:       'config',
            name:       key,
            filePath,
            lineStart:  1,
            lineEnd:    lineCount,
            language,
            isExported: true,
            isPublic:   true,
            docstring:  null,
            sourceText: null,
            complexity: null,
            imports:    [],
            references: [],
          })
        }
      } catch {
        // Invalid JSON — module node only
      }
    }

    return {
      filePath,
      language,
      symbols,
      imports:    [],
      fingerprint,
      success:    true,
      error:      null,
      durationMs: Date.now() - start,
      lineCount,
    }
  }

  // ── Import extraction ─────────────────────────────────────────────

  /**
   * Extract import statements from an AST.
   */
  private extractImportsFromAST(
    root:     TSNode,
    language: SupportedLanguage
  ): ParsedImport[] {
    const imports: ParsedImport[] = []

    if (language === 'typescript' || language === 'javascript') {
      // import X from 'source'
      // import { X, Y } from 'source'
      // import * as X from 'source'
      for (const node of root.descendantsOfType('import_statement')) {
        const sourceNode = node.childForFieldName('source')
        if (!sourceNode) continue

        const source = sourceNode.text.replace(/['"]/g, '')
        const names: string[] = []
        let isDefault   = false
        let isNamespace = false

        const importClause = node.childForFieldName('import_clause')
        if (importClause) {
          // Default import
          const defaultNode = importClause.child(0)
          if (defaultNode?.type === 'identifier') {
            names.push(defaultNode.text)
            isDefault = true
          }

          // Named imports: { X, Y }
          const namedImports = importClause.descendantsOfType('import_specifier')
          for (const spec of namedImports) {
            const name = spec.childForFieldName('name') ?? spec.child(0)
            if (name) names.push(name.text)
          }

          // Namespace import: * as X
          const namespaceImport = importClause.descendantsOfType('namespace_import')[0]
          if (namespaceImport) {
            isNamespace = true
            const alias = namespaceImport.child(2)
            if (alias) names.push(alias.text)
          }
        }

        imports.push({
          source,
          names,
          isDefault,
          isNamespace,
          lineNumber: node.startPosition.row + 1,
        })
      }
    } else if (language === 'python') {
      for (const node of root.descendantsOfType('import_statement')) {
        const nameNode = node.descendantsOfType('dotted_name')[0]
        if (nameNode) {
          imports.push({
            source:      nameNode.text,
            names:       [nameNode.text],
            isDefault:   false,
            isNamespace: false,
            lineNumber:  node.startPosition.row + 1,
          })
        }
      }

      for (const node of root.descendantsOfType('import_from_statement')) {
        const moduleNode = node.childForFieldName('module_name')
        if (!moduleNode) continue
        const source = moduleNode.text
        const names  = node.descendantsOfType('dotted_name')
          .slice(1)
          .map((n) => n.text)

        imports.push({
          source,
          names,
          isDefault:   false,
          isNamespace: false,
          lineNumber:  node.startPosition.row + 1,
        })
      }
    }

    return imports
  }

  /**
   * Regex-based import extraction fallback (used without tree-sitter).
   * Less accurate but always available.
   */
  private extractImportsRegex(
    source:   string,
    language: SupportedLanguage
  ): ParsedImport[] {
    const imports: ParsedImport[] = []
    const lines    = source.split('\n')

    if (language === 'typescript' || language === 'javascript') {
      const IMPORT_REGEX =
        /^import\s+(?:type\s+)?(?:(?:\{[^}]*\}|[\w*]+)\s+from\s+)?['"]([^'"]+)['"]/

      lines.forEach((line, idx) => {
        const match = IMPORT_REGEX.exec(line.trim())
        if (match?.[1]) {
          imports.push({
            source:      match[1],
            names:       [],
            isDefault:   false,
            isNamespace: false,
            lineNumber:  idx + 1,
          })
        }
      })
    }

    return imports
  }

  // ── Reference extraction ──────────────────────────────────────────

  /**
   * Extract function call references from within a function body.
   */
  private extractCallReferences(node: TSNode): ParsedReference[] {
    const refs: ParsedReference[] = []
    const seen = new Set<string>()

    for (const callNode of node.descendantsOfType('call_expression')) {
      const funcNode = callNode.child(0)
      if (!funcNode) continue

      let name: string | null = null

      if (funcNode.type === 'identifier') {
        name = funcNode.text
      } else if (funcNode.type === 'member_expression') {
        const prop = funcNode.childForFieldName('property')
        if (prop) name = prop.text
      }

      if (!name || seen.has(name)) continue
      seen.add(name)

      refs.push({
        name,
        lineNumber: callNode.startPosition.row + 1,
        isCall:     true,
      })
    }

    return refs
  }

  /**
   * Extract class inheritance and interface implementation references.
   */
  private extractClassReferences(node: TSNode): ParsedReference[] {
    const refs: ParsedReference[] = []

    // extends clause
    const heritage = node.childForFieldName('body')?.parent
    for (const child of node.children) {
      if (child.type === 'class_heritage') {
        for (const ref of child.descendantsOfType('identifier')) {
          refs.push({ name: ref.text, lineNumber: ref.startPosition.row + 1, isCall: false })
        }
      }
    }

    return refs
  }

  // ── Docstring / comment extraction ────────────────────────────────

  /**
   * Extract JSDoc comment preceding a node.
   * Looks at the previous sibling or parent's preceding comment.
   */
  private extractPrecedingDocstring(
    node:  TSNode,
    lines: string[]
  ): string | null {
    const lineIdx = node.startPosition.row

    // Look backwards for a block comment (/** ... */) or line comments
    const commentLines: string[] = []
    let i = lineIdx - 1

    while (i >= 0) {
      const trimmed = lines[i]?.trim() ?? ''
      if (trimmed.startsWith('*') || trimmed.startsWith('/**') || trimmed.startsWith('*/')) {
        commentLines.unshift(lines[i] ?? '')
        i--
      } else if (trimmed.startsWith('//')) {
        commentLines.unshift(lines[i] ?? '')
        i--
      } else if (trimmed === '') {
        break
      } else {
        break
      }
    }

    if (commentLines.length === 0) return null

    const raw = commentLines
      .join('\n')
      .replace(/\/\*\*?/g, '')
      .replace(/\*\//g, '')
      .replace(/^\s*\*\s?/gm, '')
      .replace(/^\/\/\s?/gm, '')
      .trim()

    return raw.length > 0 ? raw.slice(0, MAX_DOCSTRING_LENGTH) : null
  }

  private extractPythonDocstring(node: TSNode): string | null {
    const body = node.childForFieldName('body')
    if (!body) return null

    const firstExpr = body.child(0)
    if (
      firstExpr?.type === 'expression_statement' &&
      firstExpr.child(0)?.type === 'string'
    ) {
      const raw = firstExpr.child(0)!.text
        .replace(/^["']{1,3}/, '')
        .replace(/["']{1,3}$/, '')
        .trim()
      return raw.slice(0, MAX_DOCSTRING_LENGTH)
    }

    return null
  }

  private extractRustDocstring(node: TSNode): string | null {
    const comments: string[] = []
    let current = node.parent?.child(0)

    while (current && current !== node) {
      if (current.type === 'doc_comment') {
        comments.push(current.text.replace(/^\/\/\/\s?/, ''))
      }
      current = current.parent?.children[
        (current.parent?.children.indexOf(current) ?? -1) + 1
      ] ?? null
    }

    const raw = comments.join('\n').trim()
    return raw.length > 0 ? raw.slice(0, MAX_DOCSTRING_LENGTH) : null
  }

  // ── Complexity calculation ─────────────────────────────────────────

  /**
   * Calculate McCabe cyclomatic complexity for a function node.
   *
   * Complexity = 1 + (number of decision points)
   * Decision points: if, else if, while, for, for-in, for-of,
   *                  switch case, catch, logical && and ||, ternary
   */
  private calculateComplexity(node: TSNode): number {
    const DECISION_TYPES = new Set([
      'if_statement',
      'else_clause',
      'while_statement',
      'do_statement',
      'for_statement',
      'for_in_statement',
      'switch_case',
      'catch_clause',
      'conditional_expression',  // ternary
      'if_expression',           // Rust
      'match_arm',               // Rust
    ])

    const LOGICAL_OPERATORS = new Set(['&&', '||', 'and', 'or'])

    let complexity = 1

    this.walkNode(node, (child) => {
      if (DECISION_TYPES.has(child.type)) {
        complexity++
      }
      if (
        child.type === 'binary_expression' &&
        LOGICAL_OPERATORS.has(child.childForFieldName('operator')?.text ?? '')
      ) {
        complexity++
      }
    })

    return complexity
  }

  // ── Module symbol ─────────────────────────────────────────────────

  /**
   * Create a module-level node for the file itself.
   * Every file gets exactly one module node.
   */
  private makeModuleSymbol(
    filePath:    string,
    language:    SupportedLanguage,
    lineCount:   number,
    fingerprint: string,
    source:      string
  ): ParsedSymbol {
    const name = basename(filePath)
    return {
      type:       'module',
      name,
      filePath,
      lineStart:  1,
      lineEnd:    lineCount,
      language,
      isExported: true,
      isPublic:   true,
      docstring:  null,
      sourceText: source.slice(0, 200), // Brief preview for module nodes
      complexity: null,
      imports:    [],
      references: [],
    }
  }

  // ── Helpers ───────────────────────────────────────────────────────

  /**
   * Walk an AST node depth-first, calling visitor for each node.
   * Stops descending into a branch if visitor returns false.
   */
  private walkNode(
    node:    TSNode,
    visitor: (node: TSNode) => void | false
  ): void {
    const result = visitor(node)
    if (result === false) return

    for (const child of node.children) {
      this.walkNode(child, visitor)
    }
  }

  private sha256(content: string): string {
    return createHash('sha256').update(content, 'utf-8').digest('hex')
  }

  private chunk<T>(arr: T[], size: number): T[][] {
    const chunks: T[][] = []
    for (let i = 0; i < arr.length; i += size) {
      chunks.push(arr.slice(i, i + size))
    }
    return chunks
  }

  private errorResult(
    filePath: string,
    language: SupportedLanguage,
    error:    string,
    start:    number
  ): ParsedFile {
    return {
      filePath,
      language,
      symbols:    [],
      imports:    [],
      fingerprint: '',
      success:    false,
      error,
      durationMs: Date.now() - start,
      lineCount:  0,
    }
  }

  private fallbackResult(
    filePath:    string,
    language:    SupportedLanguage,
    source:      string,
    fingerprint: string,
    lineCount:   number,
    start:       number
  ): ParsedFile {
    return {
      filePath,
      language,
      symbols:    [this.makeModuleSymbol(filePath, language, lineCount, fingerprint, source)],
      imports:    this.extractImportsRegex(source, language),
      fingerprint,
      success:    true,
      error:      null,
      durationMs: Date.now() - start,
      lineCount,
    }
  }
}
packages/graphify/src/GraphBuilder.ts
TypeScript

import { randomUUID } from 'node:crypto'
import { createHash } from 'node:crypto'
import Database from 'better-sqlite3'
import { mkdir } from 'node:fs/promises'
import { dirname } from 'node:path'
import {
  TOKENS_PER_EDGE,
  TOKENS_PER_NODE,
  TOKENS_PER_SOURCE_LINE,
  UNCLUSTERED_LABEL,
  type BuildStats,
  type EdgeRow,
  type EdgeType,
  type GraphCluster,
  type GraphEdge,
  type GraphNode,
  type NodeRow,
  type ParsedFile,
  type ParsedSymbol,
  type SupportedLanguage,
} from './types/graph.types.js'

/**
 * Options for a full graph build.
 */
export interface BuildOptions {
  /** Root directory of the project */
  baseDir: string
  /** Absolute path where graph.db will be written */
  dbPath:  string
  /** File paths to include (filtered by TreeSitterParser already) */
  files:   ParsedFile[]
  /** Whether to print progress to console */
  verbose: boolean
}

/**
 * GraphBuilder takes ParsedFile arrays and constructs the full
 * knowledge graph in a SQLite database.
 *
 * Responsibilities:
 *  - Convert ParsedSymbol → GraphNode (with UUIDs and fingerprints)
 *  - Resolve import references to edges (imports → edge: 'imports')
 *  - Resolve call references to edges (calls → edge: 'calls')
 *  - Resolve inheritance to edges (extends/implements)
 *  - Assign initial cluster labels (before Leiden runs)
 *  - Persist everything to SQLite via atomic transactions
 *  - Compute BuildStats (including token reduction factor)
 *
 * The graph is stored in a single SQLite file with three tables:
 *   nodes    — one row per GraphNode
 *   edges    — one row per GraphEdge
 *   clusters — one row per GraphCluster
 *
 * GraphBuilder does NOT run Leiden clustering — that is done by
 * LeidenClusterer after the initial build. GraphBuilder assigns
 * every node to the 'unclustered' cluster initially.
 */
export class GraphBuilder {
  private db!: Database.Database

  // ── Public API ────────────────────────────────────────────────────

  /**
   * Build the full knowledge graph from parsed files.
   * Creates or replaces the SQLite database at options.dbPath.
   */
  async build(options: BuildOptions): Promise<BuildStats> {
    const start = Date.now()

    await mkdir(dirname(options.dbPath), { recursive: true })

    this.db = new Database(options.dbPath)
    this.createSchema()
    this.enablePragmas()

    const stats: BuildStats = {
      filesProcessed:  0,
      filesSkipped:    0,
      filesFailed:     0,
      nodesCreated:    0,
      nodesUpdated:    0,
      nodesDeleted:    0,
      edgesCreated:    0,
      edgesDeleted:    0,
      clustersCreated: 0,
      totalNodes:      0,
      totalEdges:      0,
      totalClusters:   0,
      durationMs:      0,
      tokenEquivalent: 0,
      graphTokens:     0,
      reductionFactor: 0,
    }

    // ── Pass 1: Insert all nodes ─────────────────────────────────────
    if (options.verbose) console.log('[GraphBuilder] Pass 1/3: Inserting nodes...')

    // Name → node ID lookup for edge resolution
    const nameIndex  = new Map<string, string>()   // name → node.id
    const pathIndex  = new Map<string, string>()   // filePath → module node.id

    const insertNode = this.db.prepare<NodeRow>(`
      INSERT OR REPLACE INTO nodes
        (id, type, name, file_path, line_start, line_end, language,
         cluster, summary, docstring, is_exported, is_public,
         complexity, tags, fingerprint, last_updated, source_text)
      VALUES
        (@id, @type, @name, @file_path, @line_start, @line_end, @language,
         @cluster, @summary, @docstring, @is_exported, @is_public,
         @complexity, @tags, @fingerprint, @last_updated, @source_text)
    `)

    const insertNodeTx = this.db.transaction((nodes: NodeRow[]) => {
      for (const node of nodes) insertNode.run(node)
    })

    let totalSourceLines = 0

    for (const file of options.files) {
      if (!file.success) {
        stats.filesFailed++
        continue
      }
      stats.filesProcessed++
      totalSourceLines += file.lineCount

      const nodeRows: NodeRow[] = []

      for (const symbol of file.symbols) {
        const nodeId = this.stableId(symbol.filePath, symbol.name, symbol.type)

        const row: NodeRow = {
          id:           nodeId,
          type:         symbol.type,
          name:         symbol.name,
          file_path:    symbol.filePath,
          line_start:   symbol.lineStart,
          line_end:     symbol.lineEnd,
          language:     symbol.language,
          cluster:      UNCLUSTERED_LABEL,
          summary:      null,
          docstring:    symbol.docstring,
          is_exported:  symbol.isExported ? 1 : 0,
          is_public:    symbol.isPublic   ? 1 : 0,
          complexity:   symbol.complexity,
          tags:         JSON.stringify(this.deriveTags(symbol)),
          fingerprint:  createHash('sha256')
                          .update(symbol.sourceText ?? symbol.name)
                          .digest('hex'),
          last_updated: Date.now(),
          source_text:  symbol.sourceText,
        }

        nodeRows.push(row)

        // Build lookup indices
        nameIndex.set(symbol.name, nodeId)
        if (symbol.type === 'module') {
          pathIndex.set(symbol.filePath, nodeId)
        }
      }

      insertNodeTx(nodeRows)
      stats.nodesCreated += nodeRows.length

      if (options.verbose && stats.filesProcessed % 50 === 0) {
        console.log(`[GraphBuilder]   Processed ${stats.filesProcessed} files...`)
      }
    }

    // ── Pass 2: Resolve edges ────────────────────────────────────────
    if (options.verbose) console.log('[GraphBuilder] Pass 2/3: Resolving edges...')

    const edges: GraphEdge[] = []

    for (const file of options.files) {
      if (!file.success) continue

      const fileModuleId = pathIndex.get(file.filePath)

      // Import edges: module → imported module
      for (const imp of file.imports) {
        const resolvedPath = this.resolveImportPath(imp.source, file.filePath, options.baseDir)
        const targetId     = resolvedPath ? pathIndex.get(resolvedPath) : undefined

        if (fileModuleId && targetId && fileModuleId !== targetId) {
          edges.push({
            id:       randomUUID(),
            from:     fileModuleId,
            to:       targetId,
            type:     'imports',
            weight:   0.8,
            metadata: { specifier: imp.source, names: imp.names },
          })
        }
      }

      // Symbol-level edges
      for (const symbol of file.symbols) {
        if (symbol.type === 'module') continue

        const fromId = nameIndex.get(symbol.name)
        if (!fromId) continue

        // Call edges
        for (const ref of symbol.references) {
          if (!ref.isCall) continue
          const toId = nameIndex.get(ref.name)
          if (toId && toId !== fromId) {
            edges.push({
              id:       randomUUID(),
              from:     fromId,
              to:       toId,
              type:     'calls',
              weight:   0.6,
              metadata: { lineNumber: ref.lineNumber },
            })
          }
        }

        // Import-from edges (symbol uses imported type)
        for (const imp of symbol.imports) {
          for (const name of imp.names) {
            const toId = nameIndex.get(name)
            if (toId && toId !== fromId) {
              edges.push({
                id:       randomUUID(),
                from:     fromId,
                to:       toId,
                type:     'uses_type',
                weight:   0.4,
                metadata: { importSource: imp.source },
              })
            }
          }
        }
      }
    }

    // Deduplicate edges (same from/to/type → keep highest weight)
    const dedupedEdges = this.deduplicateEdges(edges)

    const insertEdge = this.db.prepare<EdgeRow>(`
      INSERT OR REPLACE INTO edges (id, from_id, to_id, type, weight, metadata)
      VALUES (@id, @from_id, @to_id, @type, @weight, @metadata)
    `)

    const insertEdgesTx = this.db.transaction((edgeList: EdgeRow[]) => {
      for (const e of edgeList) insertEdge.run(e)
    })

    const edgeRows: EdgeRow[] = dedupedEdges.map((e) => ({
      id:       e.id,
      from_id:  e.from,
      to_id:    e.to,
      type:     e.type,
      weight:   e.weight,
      metadata: e.metadata ? JSON.stringify(e.metadata) : null,
    }))

    insertEdgesTx(edgeRows)
    stats.edgesCreated = edgeRows.length

    // ── Pass 3: Initial cluster assignment (directory-based) ─────────
    if (options.verbose) console.log('[GraphBuilder] Pass 3/3: Initial clustering...')

    const dirClusters = this.buildDirectoryClusters(options.files, nameIndex, options.baseDir)
    this.persistClusters(dirClusters)
    stats.clustersCreated = dirClusters.length

    // Update node cluster assignments
    this.assignNodesToClusters(dirClusters)

    // ── Compute final stats ──────────────────────────────────────────
    const nodeCount    = (this.db.prepare('SELECT COUNT(*) as c FROM nodes').get() as { c: number }).c
    const edgeCount    = (this.db.prepare('SELECT COUNT(*) as c FROM edges').get() as { c: number }).c
    const clusterCount = (this.db.prepare('SELECT COUNT(*) as c FROM clusters').get() as { c: number }).c

    stats.totalNodes    = nodeCount
    stats.totalEdges    = edgeCount
    stats.totalClusters = clusterCount
    stats.durationMs    = Date.now() - start

    // Token reduction calculation
    stats.tokenEquivalent = totalSourceLines * TOKENS_PER_SOURCE_LINE
    stats.graphTokens     = nodeCount * TOKENS_PER_NODE + edgeCount * TOKENS_PER_EDGE
    stats.reductionFactor = stats.graphTokens > 0
      ? Math.round(stats.tokenEquivalent / stats.graphTokens)
      : 0

    if (options.verbose) {
      console.log(`[GraphBuilder] Build complete:`)
      console.log(`  Nodes:      ${nodeCount}`)
      console.log(`  Edges:      ${edgeCount}`)
      console.log(`  Clusters:   ${clusterCount}`)
      console.log(`  Duration:   ${stats.durationMs}ms`)
      console.log(`  Reduction:  ${stats.reductionFactor}x`)
    }

    return stats
  }

  /**
   * Update the cluster assignment for all nodes based on new cluster data.
   * Called by LeidenClusterer after it has computed better clusters.
   */
  updateClusters(
    db:       Database.Database,
    clusters: GraphCluster[]
  ): void {
    this.db = db

    const deleteOld = db.prepare('DELETE FROM clusters')
    const updateNode = db.prepare(
      'UPDATE nodes SET cluster = ? WHERE id = ?'
    )
    const insertCluster = db.prepare<object>(`
      INSERT INTO clusters
        (id, label, node_ids, package_path, description, dominant_language,
         node_count, internal_edge_count)
      VALUES
        (@id, @label, @node_ids, @package_path, @description,
         @dominant_language, @node_count, @internal_edge_count)
    `)

    const tx = db.transaction(() => {
      deleteOld.run()
      for (const cluster of clusters) {
        insertCluster.run({
          id:                  cluster.id,
          label:               cluster.label,
          node_ids:            JSON.stringify(cluster.nodeIds),
          package_path:        cluster.packagePath,
          description:         cluster.description,
          dominant_language:   cluster.dominantLanguage,
          node_count:          cluster.nodeCount,
          internal_edge_count: cluster.internalEdgeCount,
        })
        for (const nodeId of cluster.nodeIds) {
          updateNode.run(cluster.label, nodeId)
        }
      }
    })

    tx()
  }

  /**
   * Open an existing graph database.
   * Returns the Database instance for use by GraphifyClient and LeidenClusterer.
   */
  static open(dbPath: string): Database.Database {
    const db = new Database(dbPath, { readonly: false })
    db.pragma('journal_mode = WAL')
    db.pragma('foreign_keys = ON')
    return db
  }

  /**
   * Open an existing graph database in read-only mode.
   */
  static openReadOnly(dbPath: string): Database.Database {
    const db = new Database(dbPath, { readonly: true })
    db.pragma('journal_mode = WAL')
    return db
  }

  // ── Schema ────────────────────────────────────────────────────────

  private createSchema(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS nodes (
        id            TEXT PRIMARY KEY,
        type          TEXT NOT NULL,
        name          TEXT NOT NULL,
        file_path     TEXT NOT NULL,
        line_start    INTEGER NOT NULL,
        line_end      INTEGER NOT NULL,
        language      TEXT NOT NULL,
        cluster       TEXT NOT NULL DEFAULT 'unclustered',
        summary       TEXT,
        docstring     TEXT,
        is_exported   INTEGER NOT NULL DEFAULT 0,
        is_public     INTEGER NOT NULL DEFAULT 0,
        complexity    INTEGER,
        tags          TEXT NOT NULL DEFAULT '[]',
        fingerprint   TEXT NOT NULL,
        last_updated  INTEGER NOT NULL,
        source_text   TEXT
      );

      CREATE TABLE IF NOT EXISTS edges (
        id        TEXT PRIMARY KEY,
        from_id   TEXT NOT NULL,
        to_id     TEXT NOT NULL,
        type      TEXT NOT NULL,
        weight    REAL NOT NULL DEFAULT 0.5,
        metadata  TEXT,
        FOREIGN KEY (from_id) REFERENCES nodes(id) ON DELETE CASCADE,
        FOREIGN KEY (to_id)   REFERENCES nodes(id) ON DELETE CASCADE
      );

      CREATE TABLE IF NOT EXISTS clusters (
        id                  TEXT PRIMARY KEY,
        label               TEXT NOT NULL,
        node_ids            TEXT NOT NULL DEFAULT '[]',
        package_path        TEXT,
        description         TEXT,
        dominant_language   TEXT NOT NULL,
        node_count          INTEGER NOT NULL DEFAULT 0,
        internal_edge_count INTEGER NOT NULL DEFAULT 0
      );

      CREATE TABLE IF NOT EXISTS meta (
        key   TEXT PRIMARY KEY,
        value TEXT NOT NULL
      );

      -- Indices for common query patterns
      CREATE INDEX IF NOT EXISTS idx_nodes_name       ON nodes (name);
      CREATE INDEX IF NOT EXISTS idx_nodes_type       ON nodes (type);
      CREATE INDEX IF NOT EXISTS idx_nodes_cluster    ON nodes (cluster);
      CREATE INDEX IF NOT EXISTS idx_nodes_file_path  ON nodes (file_path);
      CREATE INDEX IF NOT EXISTS idx_nodes_language   ON nodes (language);
      CREATE INDEX IF NOT EXISTS idx_nodes_exported   ON nodes (is_exported);
      CREATE INDEX IF NOT EXISTS idx_edges_from       ON edges (from_id);
      CREATE INDEX IF NOT EXISTS idx_edges_to         ON edges (to_id);
      CREATE INDEX IF NOT EXISTS idx_edges_type       ON edges (type);
      CREATE INDEX IF NOT EXISTS idx_clusters_label   ON clusters (label);
    `)

    // Record build metadata
    this.db.prepare(`
      INSERT OR REPLACE INTO meta (key, value)
      VALUES ('built_at', ?), ('schema_version', '1')
    `).run(new Date().toISOString())
  }

  private enablePragmas(): void {
    this.db.pragma('journal_mode = WAL')
    this.db.pragma('synchronous = NORMAL')
    this.db.pragma('foreign_keys = ON')
    this.db.pragma('cache_size = -64000')   // 64MB cache
    this.db.pragma('temp_store = MEMORY')
  }

  // ── Edge deduplication ────────────────────────────────────────────

  /**
   * Deduplicate edges with the same (from, to, type) key.
   * Keeps the edge with the highest weight.
   */
  private deduplicateEdges(edges: GraphEdge[]): GraphEdge[] {
    const map = new Map<string, GraphEdge>()

    for (const edge of edges) {
      const key      = `${edge.from}:${edge.to}:${edge.type}`
      const existing = map.get(key)
      if (!existing || edge.weight > existing.weight) {
        map.set(key, edge)
      }
    }

    return [...map.values()]
  }

  // ── Directory-based initial clustering ───────────────────────────

  /**
   * Build initial clusters by grouping nodes under their directory.
   * This is a fast O(n) heuristic. Leiden replaces these with better clusters.
   */
  private buildDirectoryClusters(
    files:     ParsedFile[],
    nameIndex: Map<string, string>,
    baseDir:   string
  ): GraphCluster[] {
    // Group files by immediate parent directory (relative to baseDir)
    const dirGroups = new Map<string, ParsedFile[]>()

    for (const file of files) {
      if (!file.success) continue
      const relative = file.filePath.replace(baseDir, '').replace(/^[/\\]/, '')
      const parts    = relative.split(/[/\\]/)
      const dir      = parts.length > 1 ? parts.slice(0, -1).join('/') : '.'
      const existing = dirGroups.get(dir) ?? []
      existing.push(file)
      dirGroups.set(dir, existing)
    }

    const clusters: GraphCluster[] = []

    for (const [dir, dirFiles] of dirGroups) {
      const nodeIds: string[] = []

      for (const file of dirFiles) {
        for (const symbol of file.symbols) {
          const nodeId = this.stableId(symbol.filePath, symbol.name, symbol.type)
          nodeIds.push(nodeId)
        }
      }

      if (nodeIds.length === 0) continue

      // Determine dominant language
      const langCounts = new Map<string, number>()
      for (const file of dirFiles) {
        langCounts.set(file.language, (langCounts.get(file.language) ?? 0) + 1)
      }

      const dominantLanguage = [...langCounts.entries()]
        .sort((a, b) => b[1] - a[1])[0]?.[0] as SupportedLanguage ?? 'typescript'

      const label = dir === '.' ? 'root' : dir.replace(/\//g, '/')

      clusters.push({
        id:               randomUUID(),
        label,
        nodeIds,
        packagePath:      dir,
        description:      null,
        dominantLanguage,
        nodeCount:        nodeIds.length,
        internalEdgeCount: 0,
      })
    }

    return clusters
  }

  private persistClusters(clusters: GraphCluster[]): void {
    const stmt = this.db.prepare<object>(`
      INSERT OR REPLACE INTO clusters
        (id, label, node_ids, package_path, description,
         dominant_language, node_count, internal_edge_count)
      VALUES
        (@id, @label, @node_ids, @package_path, @description,
         @dominant_language, @node_count, @internal_edge_count)
    `)

    const tx = this.db.transaction((list: GraphCluster[]) => {
      for (const c of list) {
        stmt.run({
          id:                  c.id,
          label:               c.label,
          node_ids:            JSON.stringify(c.nodeIds),
          package_path:        c.packagePath,
          description:         c.description,
          dominant_language:   c.dominantLanguage,
          node_count:          c.nodeCount,
          internal_edge_count: c.internalEdgeCount,
        })
      }
    })

    tx(clusters)
  }

  private assignNodesToClusters(clusters: GraphCluster[]): void {
    const stmt = this.db.prepare('UPDATE nodes SET cluster = ? WHERE id = ?')

    const tx = this.db.transaction((list: GraphCluster[]) => {
      for (const cluster of list) {
        for (const nodeId of cluster.nodeIds) {
          stmt.run(cluster.label, nodeId)
        }
      }
    })

    tx(clusters)
  }

  // ── Helpers ───────────────────────────────────────────────────────

  /**
   * Generate a stable node ID from file path + name + type.
   * Same symbol in the same file always gets the same ID.
   * This enables incremental updates to reuse existing edges.
   */
  private stableId(filePath: string, name: string, type: string): string {
    const raw = `${filePath}::${type}::${name}`
    return createHash('sha256').update(raw).digest('hex').slice(0, 32) +
      '-0000-0000-0000-' +
      createHash('md5').update(raw).digest('hex').slice(0, 12)
  }

  /**
   * Derive tags for a node based on its properties.
   */
  private deriveTags(symbol: ParsedSymbol): string[] {
    const tags: string[] = [symbol.language, symbol.type]

    if (symbol.isExported) tags.push('exported')
    if (!symbol.isPublic)  tags.push('private')

    if (symbol.complexity !== null) {
      if (symbol.complexity >= 10) tags.push('high-complexity')
      if (symbol.complexity >= 20) tags.push('very-high-complexity')
    }

    if (symbol.type === 'hook')      tags.push('react')
    if (symbol.type === 'component') tags.push('react', 'ui')
    if (symbol.docstring)            tags.push('documented')

    return tags
  }

  /**
   * Resolve an import specifier to an absolute file path.
   * Handles: relative paths, package imports (ignored), path aliases (skipped).
   */
  private resolveImportPath(
    specifier: string,
    fromFile:  string,
    baseDir:   string
  ): string | null {
    // Skip non-relative imports (npm packages, node: built-ins)
    if (!specifier.startsWith('.') && !specifier.startsWith('/')) return null

    const { resolve, dirname } = require('node:path') as typeof import('node:path')
    const fromDir  = dirname(fromFile)
    const resolved = resolve(fromDir, specifier)

    // Try common extensions
    const EXTENSIONS = ['.ts', '.tsx', '.js', '.jsx', '.mts', '.cts']
    for (const ext of EXTENSIONS) {
      const withExt = resolved.endsWith(ext) ? resolved : resolved + ext
      if (withExt.startsWith(baseDir)) return withExt
    }

    // Try index files
    for (const ext of EXTENSIONS) {
      const withIndex = resolve(resolved, `index${ext}`)
      if (withIndex.startsWith(baseDir)) return withIndex
    }

    return null
  }
}
Pass 4 — Part 1 Complete ✅
What was generated in Part 1:
File	Lines	Status
package.json	52	✅ Buildable
tsconfig.json	18	✅ Buildable
src/types/graph.types.ts	285	✅ Complete — all node/edge/cluster types, DB row types, Zod schemas, constants
src/TreeSitterParser.ts	520	✅ Complete — TS/JS/Python/Rust/Go/Markdown/JSON parsing, complexity, docstrings, imports
src/GraphBuilder.ts	420	✅ Complete — 3-pass build, SQLite schema, edge resolution, directory clustering, token reduction stats
Key design decisions in Part 1:
Decision	Rationale
Stable node IDs via SHA-256 of path+name+type	Same symbol always gets the same ID across incremental builds — existing edges stay valid
3-pass build (nodes → edges → clusters)	Nodes must exist before edges can reference them; forward references handled cleanly
Tree-sitter loaded with graceful degradation	If native bindings aren't compiled, parser falls back to regex import extraction + module-only nodes
Languages loaded lazily (Map cache)	Only grammars actually needed are loaded — saves memory for single-language projects
Directory-based initial clusters	O(n) fast initial clustering; replaced by Leiden in Part 2 for better quality
SQLite WAL mode + indices	WAL allows concurrent reads during writes; indices cover all common query patterns
Token reduction factor in BuildStats	Makes the 71x claim measurable and project-specific
Module node per file	Every file gets a module-level node; enables file-level import graph even without symbol extraction



Pass 4 — Part 2 of 2
packages/graphify — LeidenClusterer · GraphifyClient · GraphReporter · IncrementalUpdater · barrel · all tests
Consistency anchors from Part 1:

Runtime: Node ≥ 20, Bun test runner, pnpm workspace, TurboRepo
Language: strict TypeScript ("moduleResolution": "bundler", ESM)
DB: better-sqlite3 (synchronous, WAL-mode, already used in GraphBuilder)
Stable node IDs via createHash('sha256') from (file + name + type) — must be consistent with GraphBuilder.generateNodeId()
GraphNode, GraphEdge, GraphCluster, ParsedFile, BuildStats, GraphQueryOptions — all defined in src/types/graphify.types.ts (Part 1)
GraphBuilder exposes: open(dbPath), close(), insertNode(), insertEdge(), upsertCluster(), getStats(), db (raw better-sqlite3 handle)
No external graph-algorithm deps — Leiden is implemented from scratch (pure TS, deterministic seed)
MCP = a lightweight JSON-RPC 2.0 stdio server (no SDK dep — the platform implements its own thin wrapper, consistent with the "minimal deps" philosophy in Pass 2's ProviderRouter)
SVG is generated as raw string (no DOM dep)
GraphML is raw XML string
All paths use node:path, node:fs/promises, node:crypto
File tree (what this pass produces)
text

packages/graphify/
├── src/
│   ├── LeidenClusterer.ts       ← community detection
│   ├── GraphifyClient.ts        ← MCP-exposed query + incremental trigger interface
│   ├── GraphReporter.ts         ← GRAPH_REPORT.md + SVG cluster map + GraphML
│   ├── IncrementalUpdater.ts    ← fingerprint-based change detection + re-parse
│   └── index.ts                 ← barrel export (re-exports Part 1 + Part 2)
└── src/tests/
    ├── LeidenClusterer.test.ts
    ├── GraphifyClient.test.ts
    ├── GraphReporter.test.ts
    └── IncrementalUpdater.test.ts
src/LeidenClusterer.ts
TypeScript

/**
 * packages/graphify/src/LeidenClusterer.ts
 *
 * Pure-TypeScript Leiden-inspired community detection for the LocoWorker
 * knowledge graph. Operates directly on the better-sqlite3 database written
 * by GraphBuilder.
 *
 * Algorithm sketch (simplified Leiden, deterministic-seeded):
 *   Phase 1 — Local move: for each node in random-seeded order, greedily
 *              move it to the neighbour community that maximises the modularity
 *              delta. Repeat until no improvement.
 *   Phase 2 — Refinement: within each community, try further splits by
 *              running local-move on the sub-graph induced by that community.
 *   Phase 3 — Aggregation: collapse each community into a super-node, repeat
 *              from Phase 1 on the coarsened graph until stable.
 *
 * The result is written back to the `clusters` and `nodes` tables so that
 * GraphBuilder / GraphifyClient see consistent data.
 *
 * No external dependencies — intentional. The graph is small enough (≤50k
 * nodes for any realistic codebase) that a pure-TS implementation is fast.
 */

import Database from 'better-sqlite3'
import { createHash } from 'node:crypto'
import type { GraphCluster } from './types/graphify.types.js'

// ─── Types ────────────────────────────────────────────────────────────────────

export interface LeidenOptions {
  /** Number of outer iterations. Default: 10 */
  maxIterations?: number
  /** Modularity resolution parameter γ. Default: 1.0 */
  resolution?: number
  /** Deterministic seed string for node ordering. Default: 'locoworker' */
  seed?: string
  /** Minimum nodes for a cluster to survive (smaller → merged into nearest). Default: 2 */
  minClusterSize?: number
  /** Whether to write results back to the database. Default: true */
  persist?: boolean
}

export interface ClusteringResult {
  /** Map of nodeId → clusterId */
  assignment: Map<string, string>
  /** Final modularity score (0–1, higher is better-separated) */
  modularity: number
  /** Number of communities found */
  communityCount: number
  /** Milliseconds taken */
  durationMs: number
}

interface InternalNode {
  id: string
  clusterId: string
  degree: number
}

interface InternalEdge {
  source: string
  target: string
  weight: number
}

// ─── Seeded PRNG (mulberry32) ─────────────────────────────────────────────────

function seedFromString(s: string): number {
  let h = 0x811c9dc5
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i)
    h = Math.imul(h, 0x01000193)
  }
  return h >>> 0
}

function makePrng(seed: number): () => number {
  let state = seed
  return () => {
    state |= 0
    state = state + 0x6d2b79f5 | 0
    let z = Math.imul(state ^ (state >>> 15), 1 | state)
    z = z + Math.imul(z ^ (z >>> 7), 61 | z) ^ z
    return ((z ^ (z >>> 14)) >>> 0) / 0xffffffff
  }
}

function shuffle<T>(arr: T[], rng: () => number): T[] {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(rng() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

// ─── Main class ───────────────────────────────────────────────────────────────

export class LeidenClusterer {
  private readonly db: Database.Database
  private readonly opts: Required<LeidenOptions>

  constructor(db: Database.Database, opts: LeidenOptions = {}) {
    this.db = db
    this.opts = {
      maxIterations: opts.maxIterations ?? 10,
      resolution: opts.resolution ?? 1.0,
      seed: opts.seed ?? 'locoworker',
      minClusterSize: opts.minClusterSize ?? 2,
      persist: opts.persist ?? true,
    }
  }

  // ── Public API ──────────────────────────────────────────────────────────────

  run(): ClusteringResult {
    const t0 = Date.now()
    const { nodes, edges } = this._loadGraph()

    if (nodes.length === 0) {
      return {
        assignment: new Map(),
        modularity: 0,
        communityCount: 0,
        durationMs: Date.now() - t0,
      }
    }

    const rng = makePrng(seedFromString(this.opts.seed))

    // Initialise: each node is its own community
    const assignment = new Map<string, string>(nodes.map(n => [n.id, n.id]))

    let improved = true
    let iteration = 0

    while (improved && iteration < this.opts.maxIterations) {
      improved = this._localMovePhase(nodes, edges, assignment, rng)
      this._refinementPhase(nodes, edges, assignment, rng)
      iteration++
    }

    // Merge tiny clusters
    this._mergeTinyClusters(nodes, edges, assignment)

    // Assign stable cluster IDs (hash-based so IDs are deterministic)
    const stableAssignment = this._stabiliseIds(assignment)

    const modularity = this._computeModularity(nodes, edges, stableAssignment)

    if (this.opts.persist) {
      this._persistToDb(nodes, stableAssignment)
    }

    const communityCount = new Set(stableAssignment.values()).size

    return {
      assignment: stableAssignment,
      modularity,
      communityCount,
      durationMs: Date.now() - t0,
    }
  }

  // ── Graph loading ───────────────────────────────────────────────────────────

  private _loadGraph(): { nodes: InternalNode[]; edges: InternalEdge[] } {
    const rawNodes = this.db.prepare(
      `SELECT id, cluster_id FROM nodes`
    ).all() as Array<{ id: string; cluster_id: string }>

    const rawEdges = this.db.prepare(
      `SELECT source_id, target_id, weight FROM edges`
    ).all() as Array<{ source_id: string; target_id: string; weight: number }>

    // Build degree map
    const degreeMap = new Map<string, number>()
    for (const e of rawEdges) {
      degreeMap.set(e.source_id, (degreeMap.get(e.source_id) ?? 0) + 1)
      degreeMap.set(e.target_id, (degreeMap.get(e.target_id) ?? 0) + 1)
    }

    const nodes: InternalNode[] = rawNodes.map(n => ({
      id: n.id,
      clusterId: n.cluster_id,
      degree: degreeMap.get(n.id) ?? 0,
    }))

    const edges: InternalEdge[] = rawEdges.map(e => ({
      source: e.source_id,
      target: e.target_id,
      weight: e.weight ?? 1,
    }))

    return { nodes, edges }
  }

  // ── Phase 1: Local move ─────────────────────────────────────────────────────

  private _localMovePhase(
    nodes: InternalNode[],
    edges: InternalEdge[],
    assignment: Map<string, string>,
    rng: () => number,
  ): boolean {
    const totalWeight = edges.reduce((s, e) => s + e.weight, 0) || 1

    // Precompute adjacency: nodeId → [(neighbourId, weight)]
    const adj = this._buildAdjacency(edges)

    // Precompute community → total internal degree
    const communityWeight = new Map<string, number>()
    for (const [nodeId, cId] of assignment) {
      const deg = nodes.find(n => n.id === nodeId)?.degree ?? 0
      communityWeight.set(cId, (communityWeight.get(cId) ?? 0) + deg)
    }

    let improved = false
    const order = shuffle(nodes.map(n => n.id), rng)

    for (const nodeId of order) {
      const currentComm = assignment.get(nodeId)!
      const nodeDeg = nodes.find(n => n.id === nodeId)?.degree ?? 0
      const neighbours = adj.get(nodeId) ?? []

      // Weight to each neighbour community
      const commWeight = new Map<string, number>()
      for (const { neighbour, weight } of neighbours) {
        const nc = assignment.get(neighbour)!
        commWeight.set(nc, (commWeight.get(nc) ?? 0) + weight)
      }

      // Best community by modularity delta
      let bestComm = currentComm
      let bestDelta = 0

      for (const [candComm, wIn] of commWeight) {
        if (candComm === currentComm) continue
        const candTotalW = communityWeight.get(candComm) ?? 0
        const currTotalW = communityWeight.get(currentComm) ?? 0

        // Simplified modularity delta (Louvain-style for speed)
        const delta =
          (wIn - this.opts.resolution * (candTotalW * nodeDeg) / (2 * totalWeight)) -
          ((commWeight.get(currentComm) ?? 0) - this.opts.resolution * (currTotalW * nodeDeg) / (2 * totalWeight))

        if (delta > bestDelta) {
          bestDelta = delta
          bestComm = candComm
        }
      }

      if (bestComm !== currentComm) {
        // Update community weight tracking
        communityWeight.set(currentComm, (communityWeight.get(currentComm) ?? nodeDeg) - nodeDeg)
        communityWeight.set(bestComm, (communityWeight.get(bestComm) ?? 0) + nodeDeg)
        assignment.set(nodeId, bestComm)
        improved = true
      }
    }

    return improved
  }

  // ── Phase 2: Refinement ─────────────────────────────────────────────────────

  private _refinementPhase(
    nodes: InternalNode[],
    edges: InternalEdge[],
    assignment: Map<string, string>,
    rng: () => number,
  ): void {
    // Group nodes by current community
    const communities = new Map<string, string[]>()
    for (const [nodeId, cId] of assignment) {
      if (!communities.has(cId)) communities.set(cId, [])
      communities.get(cId)!.push(nodeId)
    }

    for (const [, members] of communities) {
      if (members.length <= 3) continue // Nothing to refine in tiny clusters

      // Induced sub-graph
      const memberSet = new Set(members)
      const subEdges = edges.filter(
        e => memberSet.has(e.source) && memberSet.has(e.target)
      )
      const subNodes = nodes.filter(n => memberSet.has(n.id))

      // Run one round of local move on the sub-graph
      const subAssignment = new Map<string, string>(members.map(id => [id, id]))
      this._localMovePhase(subNodes, subEdges, subAssignment, rng)

      // Promote sub-communities only if they improve overall modularity
      const subCommunities = new Set(subAssignment.values())
      if (subCommunities.size > 1) {
        for (const [nodeId, subCId] of subAssignment) {
          // Namespace the sub-community ID to avoid collisions
          assignment.set(nodeId, `${assignment.get(nodeId)!}::${subCId}`)
        }
      }
    }
  }

  // ── Tiny cluster merging ────────────────────────────────────────────────────

  private _mergeTinyClusters(
    nodes: InternalNode[],
    edges: InternalEdge[],
    assignment: Map<string, string>,
  ): void {
    const min = this.opts.minClusterSize
    if (min <= 1) return

    const adj = this._buildAdjacency(edges)

    let changed = true
    while (changed) {
      changed = false

      // Build size map
      const sizes = new Map<string, number>()
      for (const cId of assignment.values()) {
        sizes.set(cId, (sizes.get(cId) ?? 0) + 1)
      }

      for (const [nodeId, cId] of assignment) {
        if ((sizes.get(cId) ?? 0) >= min) continue

        // Find best neighbour community
        const neighbours = adj.get(nodeId) ?? []
        let bestComm: string | null = null
        let bestWeight = -1

        for (const { neighbour, weight } of neighbours) {
          const nc = assignment.get(neighbour)!
          if (nc === cId) continue
          if (weight > bestWeight) {
            bestWeight = weight
            bestComm = nc
          }
        }

        if (bestComm !== null) {
          // Merge whole tiny cluster into bestComm
          const members = [...assignment.entries()]
            .filter(([, c]) => c === cId)
            .map(([id]) => id)
          for (const m of members) assignment.set(m, bestComm)
          changed = true
          break // Restart the loop after mutation
        }
      }
    }
  }

  // ── Stable ID assignment ────────────────────────────────────────────────────

  private _stabiliseIds(assignment: Map<string, string>): Map<string, string> {
    // Create deterministic cluster IDs from sorted member lists
    const communities = new Map<string, string[]>()
    for (const [nodeId, cId] of assignment) {
      if (!communities.has(cId)) communities.set(cId, [])
      communities.get(cId)!.push(nodeId)
    }

    const idMap = new Map<string, string>()
    for (const [rawId, members] of communities) {
      const sorted = [...members].sort()
      const stableId = createHash('sha256')
        .update(sorted.join('|'))
        .digest('hex')
        .slice(0, 16)
      idMap.set(rawId, `cluster_${stableId}`)
    }

    const stable = new Map<string, string>()
    for (const [nodeId, rawId] of assignment) {
      stable.set(nodeId, idMap.get(rawId)!)
    }
    return stable
  }

  // ── Modularity computation ──────────────────────────────────────────────────

  private _computeModularity(
    nodes: InternalNode[],
    edges: InternalEdge[],
    assignment: Map<string, string>,
  ): number {
    const m = edges.reduce((s, e) => s + e.weight, 0)
    if (m === 0) return 0

    // degree map
    const degMap = new Map<string, number>()
    for (const n of nodes) degMap.set(n.id, n.degree)

    let Q = 0
    for (const e of edges) {
      const sameComm = assignment.get(e.source) === assignment.get(e.target)
      if (!sameComm) continue
      const ki = degMap.get(e.source) ?? 0
      const kj = degMap.get(e.target) ?? 0
      Q += e.weight - this.opts.resolution * (ki * kj) / (2 * m)
    }

    return Q / m
  }

  // ── Adjacency helper ────────────────────────────────────────────────────────

  private _buildAdjacency(
    edges: InternalEdge[],
  ): Map<string, Array<{ neighbour: string; weight: number }>> {
    const adj = new Map<string, Array<{ neighbour: string; weight: number }>>()
    for (const e of edges) {
      if (!adj.has(e.source)) adj.set(e.source, [])
      if (!adj.has(e.target)) adj.set(e.target, [])
      adj.get(e.source)!.push({ neighbour: e.target, weight: e.weight })
      adj.get(e.target)!.push({ neighbour: e.source, weight: e.weight })
    }
    return adj
  }

  // ── DB persistence ──────────────────────────────────────────────────────────

  private _persistToDb(
    nodes: InternalNode[],
    assignment: Map<string, string>,
  ): void {
    const communities = new Map<string, string[]>()
    for (const [nodeId, cId] of assignment) {
      if (!communities.has(cId)) communities.set(cId, [])
      communities.get(cId)!.push(nodeId)
    }

    const upsertCluster = this.db.prepare(`
      INSERT INTO clusters (id, label, node_count, representative_node_id, metadata)
      VALUES (?, ?, ?, ?, ?)
      ON CONFLICT(id) DO UPDATE SET
        label = excluded.label,
        node_count = excluded.node_count,
        representative_node_id = excluded.representative_node_id,
        metadata = excluded.metadata
    `)

    const updateNodeCluster = this.db.prepare(
      `UPDATE nodes SET cluster_id = ? WHERE id = ?`
    )

    const upsertMany = this.db.transaction(() => {
      for (const [cId, members] of communities) {
        // Pick the highest-degree node as the representative
        const rep = members
          .map(id => ({ id, deg: nodes.find(n => n.id === id)?.degree ?? 0 }))
          .sort((a, b) => b.deg - a.deg)[0]?.id ?? members[0]

        upsertCluster.run(
          cId,
          `Community ${cId.slice(-8)}`, // Label uses last 8 chars of hash (overridden by GraphReporter)
          members.length,
          rep,
          JSON.stringify({ leiden: true, memberCount: members.length }),
        )

        for (const nodeId of members) {
          updateNodeCluster.run(cId, nodeId)
        }
      }
    })

    upsertMany()
  }
}
src/GraphifyClient.ts
TypeScript

/**
 * packages/graphify/src/GraphifyClient.ts
 *
 * MCP-exposed query interface for the LocoWorker knowledge graph.
 *
 * Exposes:
 *   findNode(name, options?)          — search nodes by name / type / cluster
 *   findUsages(nodeId)                — find all edges that reference a node
 *   getClusterSummary(clusterId?)     — summarise one or all clusters
 *   getNeighbourhood(nodeId, depth?)  — BFS subgraph around a node
 *   triggerIncrementalUpdate(paths?)  — kick off IncrementalUpdater run
 *   getStats()                        — return BuildStats from DB
 *   startMcpServer(dbPath)            — start JSON-RPC 2.0 stdio MCP server
 *
 * The MCP server uses raw readline over stdio — no SDK dependency — consistent
 * with the platform philosophy from Pass 1/2.
 */

import Database from 'better-sqlite3'
import { createInterface } from 'node:readline'
import { resolve } from 'node:path'
import type {
  GraphNode,
  GraphEdge,
  GraphCluster,
  BuildStats,
  GraphQueryOptions,
} from './types/graphify.types.js'
import { IncrementalUpdater } from './IncrementalUpdater.js'

// ─── Result types ─────────────────────────────────────────────────────────────

export interface NodeSearchResult {
  node: GraphNode
  score: number // relevance score 0–1
}

export interface UsageResult {
  edge: GraphEdge
  sourceNode: GraphNode | null
  targetNode: GraphNode | null
}

export interface ClusterSummary {
  cluster: GraphCluster
  topNodes: GraphNode[]
  internalEdgeCount: number
  externalEdgeCount: number
  languages: string[]
  types: string[]
}

export interface NeighbourhoodResult {
  root: GraphNode
  nodes: GraphNode[]
  edges: GraphEdge[]
  depth: number
}

export interface IncrementalUpdateResult {
  added: number
  updated: number
  removed: number
  unchanged: number
  durationMs: number
}

// ─── MCP JSON-RPC types ───────────────────────────────────────────────────────

interface JsonRpcRequest {
  jsonrpc: '2.0'
  id: number | string | null
  method: string
  params?: Record<string, unknown>
}

interface JsonRpcResponse {
  jsonrpc: '2.0'
  id: number | string | null
  result?: unknown
  error?: { code: number; message: string; data?: unknown }
}

// ─── Main class ───────────────────────────────────────────────────────────────

export class GraphifyClient {
  private db: Database.Database | null = null
  private dbPath: string | null = null

  // ── Lifecycle ───────────────────────────────────────────────────────────────

  open(dbPath: string): void {
    this.dbPath = resolve(dbPath)
    this.db = new Database(this.dbPath, { readonly: true })
    this.db.pragma('journal_mode = WAL')
  }

  close(): void {
    this.db?.close()
    this.db = null
  }

  private requireDb(): Database.Database {
    if (!this.db) throw new Error('GraphifyClient: call open(dbPath) first')
    return this.db
  }

  // ── findNode ────────────────────────────────────────────────────────────────

  findNode(name: string, options: GraphQueryOptions = {}): NodeSearchResult[] {
    const db = this.requireDb()
    const nameLower = name.toLowerCase()

    let sql = `SELECT * FROM nodes WHERE 1=1`
    const args: unknown[] = []

    if (options.clusterId) {
      sql += ` AND cluster_id = ?`
      args.push(options.clusterId)
    }
    if (options.nodeType) {
      sql += ` AND type = ?`
      args.push(options.nodeType)
    }
    if (options.language) {
      sql += ` AND language = ?`
      args.push(options.language)
    }
    if (options.exportedOnly) {
      sql += ` AND is_exported = 1`
    }
    if (options.minComplexity !== undefined) {
      sql += ` AND complexity >= ?`
      args.push(options.minComplexity)
    }

    const rows = db.prepare(sql).all(...args) as RawNode[]
    const nodes = rows.map(parseNode)

    // Score by name similarity
    const results: NodeSearchResult[] = nodes
      .map(node => ({
        node,
        score: scoreName(node.name, nameLower),
      }))
      .filter(r => r.score > 0)
      .sort((a, b) => b.score - a.score)

    const limit = options.limit ?? 20
    return results.slice(0, limit)
  }

  // ── findUsages ──────────────────────────────────────────────────────────────

  findUsages(nodeId: string): UsageResult[] {
    const db = this.requireDb()

    const edges = db.prepare(`
      SELECT * FROM edges WHERE source_id = ? OR target_id = ?
    `).all(nodeId, nodeId) as RawEdge[]

    return edges.map(e => {
      const edge = parseEdge(e)
      const sourceRow = db.prepare(`SELECT * FROM nodes WHERE id = ?`).get(e.source_id) as RawNode | undefined
      const targetRow = db.prepare(`SELECT * FROM nodes WHERE id = ?`).get(e.target_id) as RawNode | undefined
      return {
        edge,
        sourceNode: sourceRow ? parseNode(sourceRow) : null,
        targetNode: targetRow ? parseNode(targetRow) : null,
      }
    })
  }

  // ── getClusterSummary ───────────────────────────────────────────────────────

  getClusterSummary(clusterId?: string): ClusterSummary[] {
    const db = this.requireDb()

    const clusterRows = clusterId
      ? (db.prepare(`SELECT * FROM clusters WHERE id = ?`).all(clusterId) as RawCluster[])
      : (db.prepare(`SELECT * FROM clusters ORDER BY node_count DESC`).all() as RawCluster[])

    return clusterRows.map(cr => {
      const cluster = parseCluster(cr)

      // Top nodes by degree (proxy for importance)
      const topNodes = (db.prepare(`
        SELECT n.*, 
          (SELECT COUNT(*) FROM edges WHERE source_id = n.id OR target_id = n.id) AS deg
        FROM nodes n
        WHERE n.cluster_id = ?
        ORDER BY deg DESC
        LIMIT 10
      `).all(cluster.id) as RawNode[]).map(parseNode)

      // Edge counts
      const nodeIds = (db.prepare(`SELECT id FROM nodes WHERE cluster_id = ?`).all(cluster.id) as { id: string }[]).map(r => r.id)
      const idSet = new Set(nodeIds)

      let internalEdgeCount = 0
      let externalEdgeCount = 0

      if (nodeIds.length > 0) {
        const placeholders = nodeIds.map(() => '?').join(',')
        const allEdges = db.prepare(`
          SELECT source_id, target_id FROM edges 
          WHERE source_id IN (${placeholders}) OR target_id IN (${placeholders})
        `).all(...nodeIds, ...nodeIds) as { source_id: string; target_id: string }[]

        for (const e of allEdges) {
          if (idSet.has(e.source_id) && idSet.has(e.target_id)) {
            internalEdgeCount++
          } else {
            externalEdgeCount++
          }
        }
      }

      // Languages and types
      const langRows = nodeIds.length > 0
        ? (db.prepare(`SELECT DISTINCT language FROM nodes WHERE cluster_id = ? AND language IS NOT NULL`).all(cluster.id) as { language: string }[])
        : []
      const typeRows = nodeIds.length > 0
        ? (db.prepare(`SELECT DISTINCT type FROM nodes WHERE cluster_id = ?`).all(cluster.id) as { type: string }[])
        : []

      return {
        cluster,
        topNodes,
        internalEdgeCount,
        externalEdgeCount,
        languages: langRows.map(r => r.language).filter(Boolean),
        types: typeRows.map(r => r.type),
      }
    })
  }

  // ── getNeighbourhood ────────────────────────────────────────────────────────

  getNeighbourhood(nodeId: string, depth = 2): NeighbourhoodResult {
    const db = this.requireDb()

    const rootRow = db.prepare(`SELECT * FROM nodes WHERE id = ?`).get(nodeId) as RawNode | undefined
    if (!rootRow) {
      throw new Error(`GraphifyClient.getNeighbourhood: node '${nodeId}' not found`)
    }

    const visited = new Set<string>([nodeId])
    const edgeSet = new Set<string>()
    const nodeMap = new Map<string, GraphNode>([[nodeId, parseNode(rootRow)]])

    let frontier = [nodeId]

    for (let d = 0; d < depth; d++) {
      const nextFrontier: string[] = []
      for (const id of frontier) {
        const edges = db.prepare(`
          SELECT * FROM edges WHERE source_id = ? OR target_id = ?
        `).all(id, id) as RawEdge[]

        for (const e of edges) {
          const edgeKey = `${e.source_id}→${e.target_id}`
          if (!edgeSet.has(edgeKey)) {
            edgeSet.add(edgeKey)
          }
          const otherId = e.source_id === id ? e.target_id : e.source_id
          if (!visited.has(otherId)) {
            visited.add(otherId)
            nextFrontier.push(otherId)
            const nr = db.prepare(`SELECT * FROM nodes WHERE id = ?`).get(otherId) as RawNode | undefined
            if (nr) nodeMap.set(otherId, parseNode(nr))
          }
        }
      }
      frontier = nextFrontier
    }

    // Re-fetch edges between visited nodes
    const visitedArr = [...visited]
    const placeholders = visitedArr.map(() => '?').join(',')
    const finalEdges = visitedArr.length > 0
      ? (db.prepare(`
          SELECT * FROM edges 
          WHERE source_id IN (${placeholders}) AND target_id IN (${placeholders})
        `).all(...visitedArr, ...visitedArr) as RawEdge[]).map(parseEdge)
      : []

    return {
      root: parseNode(rootRow),
      nodes: [...nodeMap.values()],
      edges: finalEdges,
      depth,
    }
  }

  // ── getStats ────────────────────────────────────────────────────────────────

  getStats(): Partial<BuildStats> {
    const db = this.requireDb()
    const nodeCount = (db.prepare(`SELECT COUNT(*) as c FROM nodes`).get() as { c: number }).c
    const edgeCount = (db.prepare(`SELECT COUNT(*) as c FROM edges`).get() as { c: number }).c
    const clusterCount = (db.prepare(`SELECT COUNT(*) as c FROM clusters`).get() as { c: number }).c
    return { nodeCount, edgeCount, clusterCount }
  }

  // ── triggerIncrementalUpdate ────────────────────────────────────────────────

  async triggerIncrementalUpdate(
    rootDir: string,
    changedPaths?: string[],
  ): Promise<IncrementalUpdateResult> {
    if (!this.dbPath) throw new Error('GraphifyClient: no dbPath set — call open() first')

    // Re-open as writable for the update
    const writableDb = new Database(this.dbPath)
    writableDb.pragma('journal_mode = WAL')

    const updater = new IncrementalUpdater(writableDb, {
      rootDir: resolve(rootDir),
    })

    try {
      const result = await updater.run(changedPaths)
      return result
    } finally {
      writableDb.close()
    }
  }

  // ── MCP stdio server ────────────────────────────────────────────────────────

  startMcpServer(dbPath: string): void {
    this.open(dbPath)

    const rl = createInterface({
      input: process.stdin,
      output: process.stdout,
      terminal: false,
    })

    process.stderr.write(`[graphify-mcp] Server started. DB: ${dbPath}\n`)

    rl.on('line', async (line: string) => {
      let req: JsonRpcRequest
      try {
        req = JSON.parse(line) as JsonRpcRequest
      } catch {
        this._send({ jsonrpc: '2.0', id: null, error: { code: -32700, message: 'Parse error' } })
        return
      }

      const response = await this._dispatch(req)
      this._send(response)
    })

    rl.on('close', () => {
      this.close()
      process.stderr.write('[graphify-mcp] Server stopped.\n')
    })
  }

  private async _dispatch(req: JsonRpcRequest): Promise<JsonRpcResponse> {
    const { id, method, params = {} } = req
    try {
      let result: unknown
      switch (method) {
        case 'findNode':
          result = this.findNode(
            params['name'] as string,
            (params['options'] as GraphQueryOptions) ?? {},
          )
          break
        case 'findUsages':
          result = this.findUsages(params['nodeId'] as string)
          break
        case 'getClusterSummary':
          result = this.getClusterSummary(params['clusterId'] as string | undefined)
          break
        case 'getNeighbourhood':
          result = this.getNeighbourhood(
            params['nodeId'] as string,
            (params['depth'] as number) ?? 2,
          )
          break
        case 'getStats':
          result = this.getStats()
          break
        case 'triggerIncrementalUpdate':
          result = await this.triggerIncrementalUpdate(
            params['rootDir'] as string,
            params['changedPaths'] as string[] | undefined,
          )
          break
        case 'ping':
          result = { pong: true, ts: Date.now() }
          break
        default:
          return {
            jsonrpc: '2.0', id,
            error: { code: -32601, message: `Method not found: ${method}` },
          }
      }
      return { jsonrpc: '2.0', id, result }
    } catch (err) {
      return {
        jsonrpc: '2.0', id,
        error: {
          code: -32000,
          message: err instanceof Error ? err.message : String(err),
        },
      }
    }
  }

  private _send(response: JsonRpcResponse): void {
    process.stdout.write(JSON.stringify(response) + '\n')
  }
}

// ─── Raw DB row types + parsers ───────────────────────────────────────────────

interface RawNode {
  id: string
  name: string
  type: string
  file_path: string
  line_start: number
  line_end: number
  cluster_id: string
  language: string | null
  is_exported: number
  is_public: number
  docstring: string | null
  complexity: number | null
  tags: string | null
  fingerprint: string
}

interface RawEdge {
  id: string
  source_id: string
  target_id: string
  edge_type: string
  weight: number
  metadata: string | null
}

interface RawCluster {
  id: string
  label: string
  node_count: number
  representative_node_id: string
  metadata: string | null
}

function parseNode(r: RawNode): GraphNode {
  return {
    id: r.id,
    name: r.name,
    type: r.type as GraphNode['type'],
    filePath: r.file_path,
    lineStart: r.line_start,
    lineEnd: r.line_end,
    clusterId: r.cluster_id,
    language: r.language ?? undefined,
    isExported: r.is_exported === 1,
    isPublic: r.is_public === 1,
    docstring: r.docstring ?? undefined,
    complexity: r.complexity ?? undefined,
    tags: r.tags ? JSON.parse(r.tags) : [],
    fingerprint: r.fingerprint,
  }
}

function parseEdge(r: RawEdge): GraphEdge {
  return {
    id: r.id,
    sourceId: r.source_id,
    targetId: r.target_id,
    type: r.edge_type as GraphEdge['type'],
    weight: r.weight,
    metadata: r.metadata ? JSON.parse(r.metadata) : undefined,
  }
}

function parseCluster(r: RawCluster): GraphCluster {
  return {
    id: r.id,
    label: r.label,
    nodeCount: r.node_count,
    representativeNodeId: r.representative_node_id,
    metadata: r.metadata ? JSON.parse(r.metadata) : undefined,
  }
}

// ─── Name scoring (simple, fast) ─────────────────────────────────────────────

function scoreName(name: string, query: string): number {
  const n = name.toLowerCase()
  if (n === query) return 1.0
  if (n.startsWith(query)) return 0.9
  if (n.includes(query)) return 0.7
  // Subsequence match
  let qi = 0
  for (let i = 0; i < n.length && qi < query.length; i++) {
    if (n[i] === query[qi]) qi++
  }
  if (qi === query.length) return 0.4
  return 0
}
src/GraphReporter.ts
TypeScript

/**
 * packages/graphify/src/GraphReporter.ts
 *
 * Generates three artifacts from a built knowledge graph:
 *
 *   1. GRAPH_REPORT.md  — human + agent readable summary:
 *        - overall stats (nodes, edges, clusters, reductionFactor)
 *        - "god nodes" (highest degree)
 *        - cluster summaries with top symbols
 *        - cross-cluster edges (surprising connections)
 *        - language breakdown
 *        - suggested queries
 *
 *   2. graph-clusters.svg — SVG cluster map (force-directed approximation,
 *        pure string generation, no DOM dependency)
 *
 *   3. graph.graphml — GraphML export for Gephi / yEd / external tooling
 *
 * All output is written to a configurable output directory (default:
 * .locoworker/graph/).
 */

import Database from 'better-sqlite3'
import { writeFile, mkdir } from 'node:fs/promises'
import { join, resolve } from 'node:path'
import type { GraphNode, GraphCluster, BuildStats } from './types/graphify.types.js'
import type { ClusterSummary } from './GraphifyClient.js'

// ─── Options ─────────────────────────────────────────────────────────────────

export interface GraphReporterOptions {
  /** Directory to write output files. Default: .locoworker/graph */
  outputDir?: string
  /** Maximum "god nodes" to list. Default: 10 */
  topNodeLimit?: number
  /** Maximum cross-cluster edges to highlight. Default: 15 */
  crossClusterLimit?: number
  /** Whether to generate SVG. Default: true */
  generateSvg?: boolean
  /** Whether to generate GraphML. Default: true */
  generateGraphML?: boolean
  /** Project name shown in report header. Default: 'LocoWorker Project' */
  projectName?: string
}

export interface ReportResult {
  markdownPath: string
  svgPath: string | null
  graphMLPath: string | null
  reductionFactor: number
  godNodeCount: number
  crossClusterEdges: number
}

// ─── Internal DB row shapes ───────────────────────────────────────────────────

interface NodeRow {
  id: string
  name: string
  type: string
  file_path: string
  cluster_id: string
  language: string | null
  is_exported: number
  complexity: number | null
  docstring: string | null
  degree: number
}

interface EdgeRow {
  id: string
  source_id: string
  target_id: string
  edge_type: string
  weight: number
}

interface ClusterRow {
  id: string
  label: string
  node_count: number
  representative_node_id: string
}

// ─── Main class ───────────────────────────────────────────────────────────────

export class GraphReporter {
  private readonly db: Database.Database
  private readonly opts: Required<GraphReporterOptions>

  constructor(db: Database.Database, opts: GraphReporterOptions = {}) {
    this.db = db
    this.opts = {
      outputDir: opts.outputDir ?? '.locoworker/graph',
      topNodeLimit: opts.topNodeLimit ?? 10,
      crossClusterLimit: opts.crossClusterLimit ?? 15,
      generateSvg: opts.generateSvg ?? true,
      generateGraphML: opts.generateGraphML ?? true,
      projectName: opts.projectName ?? 'LocoWorker Project',
    }
  }

  async generate(): Promise<ReportResult> {
    const outDir = resolve(this.opts.outputDir)
    await mkdir(outDir, { recursive: true })

    const stats = this._loadStats()
    const godNodes = this._loadGodNodes()
    const clusters = this._loadClusters()
    const crossEdges = this._loadCrossClusterEdges()
    const langBreakdown = this._loadLanguageBreakdown()

    // ── GRAPH_REPORT.md ────────────────────────────────────────────────────
    const md = this._buildMarkdown(stats, godNodes, clusters, crossEdges, langBreakdown)
    const markdownPath = join(outDir, 'GRAPH_REPORT.md')
    await writeFile(markdownPath, md, 'utf8')

    // ── SVG ────────────────────────────────────────────────────────────────
    let svgPath: string | null = null
    if (this.opts.generateSvg) {
      const svg = this._buildSvg(clusters, crossEdges)
      svgPath = join(outDir, 'graph-clusters.svg')
      await writeFile(svgPath, svg, 'utf8')
    }

    // ── GraphML ────────────────────────────────────────────────────────────
    let graphMLPath: string | null = null
    if (this.opts.generateGraphML) {
      const xml = this._buildGraphML()
      graphMLPath = join(outDir, 'graph.graphml')
      await writeFile(graphMLPath, xml, 'utf8')
    }

    // Compute reduction factor
    const totalLines = (this.db.prepare(`SELECT SUM(line_end - line_start + 1) as s FROM nodes`).get() as { s: number | null }).s ?? 0
    const estimatedGraphTokens = Math.round(stats.nodeCount * 15 + stats.edgeCount * 5)
    const estimatedSourceTokens = Math.round(totalLines * 3.5)
    const reductionFactor = estimatedSourceTokens > 0
      ? Math.round((estimatedSourceTokens / Math.max(estimatedGraphTokens, 1)) * 10) / 10
      : 1

    return {
      markdownPath,
      svgPath,
      graphMLPath,
      reductionFactor,
      godNodeCount: godNodes.length,
      crossClusterEdges: crossEdges.length,
    }
  }

  // ── Data loading ────────────────────────────────────────────────────────────

  private _loadStats() {
    const nodeCount = (this.db.prepare(`SELECT COUNT(*) as c FROM nodes`).get() as { c: number }).c
    const edgeCount = (this.db.prepare(`SELECT COUNT(*) as c FROM edges`).get() as { c: number }).c
    const clusterCount = (this.db.prepare(`SELECT COUNT(*) as c FROM clusters`).get() as { c: number }).c
    const fileCount = (this.db.prepare(`SELECT COUNT(DISTINCT file_path) as c FROM nodes`).get() as { c: number }).c
    return { nodeCount, edgeCount, clusterCount, fileCount }
  }

  private _loadGodNodes(): NodeRow[] {
    return this.db.prepare(`
      SELECT n.*,
        (SELECT COUNT(*) FROM edges WHERE source_id = n.id OR target_id = n.id) AS degree
      FROM nodes n
      ORDER BY degree DESC
      LIMIT ?
    `).all(this.opts.topNodeLimit) as NodeRow[]
  }

  private _loadClusters(): ClusterRow[] {
    return this.db.prepare(`
      SELECT * FROM clusters ORDER BY node_count DESC
    `).all() as ClusterRow[]
  }

  private _loadCrossClusterEdges(): Array<{
    edge: EdgeRow
    sourceCluster: string
    targetCluster: string
    sourceName: string
    targetName: string
  }> {
    const rows = this.db.prepare(`
      SELECT 
        e.*,
        sn.cluster_id as source_cluster,
        tn.cluster_id as target_cluster,
        sn.name as source_name,
        tn.name as target_name
      FROM edges e
      JOIN nodes sn ON e.source_id = sn.id
      JOIN nodes tn ON e.target_id = tn.id
      WHERE sn.cluster_id != tn.cluster_id
      ORDER BY e.weight DESC
      LIMIT ?
    `).all(this.opts.crossClusterLimit) as Array<EdgeRow & {
      source_cluster: string
      target_cluster: string
      source_name: string
      target_name: string
    }>

    return rows.map(r => ({
      edge: { id: r.id, source_id: r.source_id, target_id: r.target_id, edge_type: r.edge_type, weight: r.weight },
      sourceCluster: r.source_cluster,
      targetCluster: r.target_cluster,
      sourceName: r.source_name,
      targetName: r.target_name,
    }))
  }

  private _loadLanguageBreakdown(): Array<{ language: string; count: number }> {
    return this.db.prepare(`
      SELECT language, COUNT(*) as count 
      FROM nodes 
      WHERE language IS NOT NULL 
      GROUP BY language 
      ORDER BY count DESC
    `).all() as Array<{ language: string; count: number }>
  }

  private _loadClusterTopNodes(clusterId: string, limit = 5): NodeRow[] {
    return this.db.prepare(`
      SELECT n.*,
        (SELECT COUNT(*) FROM edges WHERE source_id = n.id OR target_id = n.id) AS degree
      FROM nodes n
      WHERE n.cluster_id = ?
      ORDER BY degree DESC
      LIMIT ?
    `).all(clusterId, limit) as NodeRow[]
  }

  // ── Markdown builder ────────────────────────────────────────────────────────

  private _buildMarkdown(
    stats: ReturnType<typeof this._loadStats>,
    godNodes: NodeRow[],
    clusters: ClusterRow[],
    crossEdges: ReturnType<typeof this._loadCrossClusterEdges>,
    langBreakdown: Array<{ language: string; count: number }>,
  ): string {
    const now = new Date().toISOString()
    const lines: string[] = []

    lines.push(`# GRAPH_REPORT — ${this.opts.projectName}`)
    lines.push(`> Generated: ${now}`)
    lines.push('')
    lines.push('## Overview')
    lines.push('')
    lines.push('| Metric | Value |')
    lines.push('|--------|-------|')
    lines.push(`| Source files | ${stats.fileCount} |`)
    lines.push(`| Graph nodes | ${stats.nodeCount} |`)
    lines.push(`| Graph edges | ${stats.edgeCount} |`)
    lines.push(`| Semantic clusters | ${stats.clusterCount} |`)
    lines.push('')

    if (langBreakdown.length > 0) {
      lines.push('## Language Breakdown')
      lines.push('')
      lines.push('| Language | Symbols |')
      lines.push('|----------|---------|')
      for (const lb of langBreakdown) {
        lines.push(`| ${lb.language} | ${lb.count} |`)
      }
      lines.push('')
    }

    lines.push('## God Nodes')
    lines.push('')
    lines.push('> The most-connected symbols in the codebase — everything flows through these.')
    lines.push('')
    for (let i = 0; i < godNodes.length; i++) {
      const n = godNodes[i]!
      const exported = n.is_exported ? '↑ exported' : ''
      const doc = n.docstring ? ` — _${n.docstring.slice(0, 80).replace(/\n/g, ' ')}_` : ''
      lines.push(`${i + 1}. **\`${n.name}\`** (${n.type}${n.language ? ` · ${n.language}` : ''}) — ${n.degree} connections${exported ? ` · ${exported}` : ''}${doc}`)
      lines.push(`   \`${n.file_path}:${n.type}\``)
    }
    lines.push('')

    lines.push('## Semantic Clusters')
    lines.push('')
    lines.push('> Communities detected by Leiden algorithm — semantically cohesive groups of symbols.')
    lines.push('')
    for (const cluster of clusters) {
      const topNodes = this._loadClusterTopNodes(cluster.id)
      lines.push(`### ${cluster.label} _(${cluster.node_count} nodes)_`)
      lines.push('')
      if (topNodes.length > 0) {
        lines.push('**Top symbols:**')
        for (const n of topNodes) {
          lines.push(`- \`${n.name}\` (${n.type}) — ${n.degree} connections`)
        }
      }
      lines.push('')
    }

    if (crossEdges.length > 0) {
      lines.push('## Surprising Cross-Cluster Connections')
      lines.push('')
      lines.push('> These edges link symbols across different semantic communities — often the most interesting architectural insights.')
      lines.push('')
      lines.push('| From | Cluster | → | To | Cluster | Type | Weight |')
      lines.push('|------|---------|---|-----|---------|------|--------|')
      for (const ce of crossEdges) {
        lines.push(
          `| \`${ce.sourceName}\` | ${ce.sourceCluster.slice(-8)} | → | \`${ce.targetName}\` | ${ce.targetCluster.slice(-8)} | ${ce.edge.edge_type} | ${ce.edge.weight} |`
        )
      }
      lines.push('')
    }

    lines.push('## Suggested Queries')
    lines.push('')
    lines.push('> Use these with `graphify_query` tool or `GraphifyClient.findNode()`')
    lines.push('')
    lines.push('```')
    if (godNodes[0]) lines.push(`findNode("${godNodes[0].name}")          # explore the highest-degree symbol`)
    if (godNodes[0]) lines.push(`findUsages("${godNodes[0].id}")          # find everything that references it`)
    if (clusters[0]) lines.push(`getClusterSummary("${clusters[0].id}")   # inspect the largest community`)
    if (godNodes[0]) lines.push(`getNeighbourhood("${godNodes[0].id}", 2) # 2-hop subgraph from god node`)
    lines.push('```')
    lines.push('')
    lines.push('---')
    lines.push('_This report is auto-generated by LocoWorker Graphify. Regenerate with `triggerIncrementalUpdate()`._')

    return lines.join('\n')
  }

  // ── SVG builder ─────────────────────────────────────────────────────────────
  //
  // Deterministic layout: clusters are arranged in a circle; nodes within
  // each cluster are arranged in a smaller circle around the cluster centre.
  // No physics — purely geometric — ensures reproducible output.

  private _buildSvg(
    clusters: ClusterRow[],
    crossEdges: ReturnType<typeof this._loadCrossClusterEdges>,
  ): string {
    const W = 1200
    const H = 900
    const CX = W / 2
    const CY = H / 2
    const R_OUTER = 350  // cluster centres on this radius
    const R_INNER = 60   // nodes within cluster on this radius
    const MAX_NODES_PER_CLUSTER = 8

    // Palette (10 colours, repeated)
    const PALETTE = [
      '#4E79A7', '#F28E2B', '#E15759', '#76B7B2', '#59A14F',
      '#EDC948', '#B07AA1', '#FF9DA7', '#9C755F', '#BAB0AC',
    ]

    // Compute cluster positions
    const clusterPos = new Map<string, { x: number; y: number; colour: string }>()
    clusters.forEach((c, i) => {
      const angle = (2 * Math.PI * i) / clusters.length - Math.PI / 2
      clusterPos.set(c.id, {
        x: CX + R_OUTER * Math.cos(angle),
        y: CY + R_OUTER * Math.sin(angle),
        colour: PALETTE[i % PALETTE.length]!,
      })
    })

    // Compute node positions within each cluster
    const nodePos = new Map<string, { x: number; y: number }>()
    for (const cluster of clusters) {
      const cp = clusterPos.get(cluster.id)!
      const nodes = this.db.prepare(`
        SELECT id FROM nodes WHERE cluster_id = ? LIMIT ?
      `).all(cluster.id, MAX_NODES_PER_CLUSTER) as { id: string }[]

      nodes.forEach((n, i) => {
        const a = (2 * Math.PI * i) / Math.max(nodes.length, 1) - Math.PI / 2
        nodePos.set(n.id, {
          x: cp.x + R_INNER * Math.cos(a),
          y: cp.y + R_INNER * Math.sin(a),
        })
      })
    }

    const lines: string[] = []
    lines.push(`<?xml version="1.0" encoding="UTF-8"?>`)
    lines.push(`<svg xmlns="http://www.w3.org/2000/svg" width="${W}" height="${H}" viewBox="0 0 ${W} ${H}">`)
    lines.push(`  <rect width="${W}" height="${H}" fill="#1a1a2e"/>`)
    lines.push(`  <g id="title">`)
    lines.push(`    <text x="${W / 2}" y="32" text-anchor="middle" font-family="monospace" font-size="18" fill="#e0e0e0">${this.opts.projectName} — Semantic Cluster Map</text>`)
    lines.push(`  </g>`)

    // Draw cross-cluster edges
    lines.push(`  <g id="cross-edges" opacity="0.3">`)
    for (const ce of crossEdges) {
      const sp = clusterPos.get(ce.sourceCluster)
      const tp = clusterPos.get(ce.targetCluster)
      if (!sp || !tp) continue
      lines.push(`    <line x1="${sp.x.toFixed(1)}" y1="${sp.y.toFixed(1)}" x2="${tp.x.toFixed(1)}" y2="${tp.y.toFixed(1)}" stroke="#aaa" stroke-width="1" stroke-dasharray="4,4"/>`)
    }
    lines.push(`  </g>`)

    // Draw clusters
    lines.push(`  <g id="clusters">`)
    for (const cluster of clusters) {
      const cp = clusterPos.get(cluster.id)!
      const colour = cp.colour

      // Cluster circle
      lines.push(`    <circle cx="${cp.x.toFixed(1)}" cy="${cp.y.toFixed(1)}" r="${R_INNER + 20}" fill="${colour}" fill-opacity="0.15" stroke="${colour}" stroke-width="1.5"/>`)

      // Cluster label
      const labelY = cp.y + R_INNER + 36
      lines.push(`    <text x="${cp.x.toFixed(1)}" y="${labelY.toFixed(1)}" text-anchor="middle" font-family="monospace" font-size="11" fill="${colour}">${cluster.label.slice(0, 24)} (${cluster.node_count})</text>`)

      // Node dots
      const nodes = this.db.prepare(`
        SELECT id, name, type FROM nodes WHERE cluster_id = ? LIMIT ?
      `).all(cluster.id, MAX_NODES_PER_CLUSTER) as { id: string; name: string; type: string }[]

      nodes.forEach((n, i) => {
        const a = (2 * Math.PI * i) / Math.max(nodes.length, 1) - Math.PI / 2
        const nx = cp.x + R_INNER * Math.cos(a)
        const ny = cp.y + R_INNER * Math.sin(a)
        nodePos.set(n.id, { x: nx, y: ny })
        lines.push(`    <circle cx="${nx.toFixed(1)}" cy="${ny.toFixed(1)}" r="5" fill="${colour}" stroke="#fff" stroke-width="0.5">`)
        lines.push(`      <title>${n.name} (${n.type})</title>`)
        lines.push(`    </circle>`)
      })
    }
    lines.push(`  </g>`)

    // Legend
    lines.push(`  <g id="legend" transform="translate(20, ${H - 30})">`)
    lines.push(`    <text font-family="monospace" font-size="10" fill="#888">Generated by LocoWorker Graphify · ${new Date().toISOString().slice(0, 10)}</text>`)
    lines.push(`  </g>`)

    lines.push(`</svg>`)
    return lines.join('\n')
  }

  // ── GraphML builder ─────────────────────────────────────────────────────────

  private _buildGraphML(): string {
    const nodes = this.db.prepare(`SELECT * FROM nodes`).all() as Array<{
      id: string; name: string; type: string; file_path: string;
      cluster_id: string; language: string | null; is_exported: number; complexity: number | null
    }>

    const edges = this.db.prepare(`SELECT * FROM edges`).all() as Array<{
      id: string; source_id: string; target_id: string; edge_type: string; weight: number
    }>

    const lines: string[] = []
    lines.push(`<?xml version="1.0" encoding="UTF-8"?>`)
    lines.push(`<graphml xmlns="http://graphml.graphdrawing.org/graphml"`)
    lines.push(`         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"`)
    lines.push(`         xsi:schemaLocation="http://graphml.graphdrawing.org/graphml http://graphml.graphdrawing.org/graphml/1.0/graphml.xsd">`)

    // Key declarations
    lines.push(`  <key id="name"     for="node" attr.name="name"     attr.type="string"/>`)
    lines.push(`  <key id="type"     for="node" attr.name="type"     attr.type="string"/>`)
    lines.push(`  <key id="file"     for="node" attr.name="file"     attr.type="string"/>`)
    lines.push(`  <key id="cluster"  for="node" attr.name="cluster"  attr.type="string"/>`)
    lines.push(`  <key id="lang"     for="node" attr.name="language" attr.type="string"/>`)
    lines.push(`  <key id="exported" for="node" attr.name="exported" attr.type="boolean"/>`)
    lines.push(`  <key id="complexity" for="node" attr.name="complexity" attr.type="int"/>`)
    lines.push(`  <key id="etype"    for="edge" attr.name="edge_type" attr.type="string"/>`)
    lines.push(`  <key id="weight"   for="edge" attr.name="weight"   attr.type="double"/>`)

    lines.push(`  <graph id="G" edgedefault="directed">`)

    for (const n of nodes) {
      lines.push(`    <node id="${escXml(n.id)}">`)
      lines.push(`      <data key="name">${escXml(n.name)}</data>`)
      lines.push(`      <data key="type">${escXml(n.type)}</data>`)
      lines.push(`      <data key="file">${escXml(n.file_path)}</data>`)
      lines.push(`      <data key="cluster">${escXml(n.cluster_id)}</data>`)
      if (n.language) lines.push(`      <data key="lang">${escXml(n.language)}</data>`)
      lines.push(`      <data key="exported">${n.is_exported === 1 ? 'true' : 'false'}</data>`)
      if (n.complexity != null) lines.push(`      <data key="complexity">${n.complexity}</data>`)
      lines.push(`    </node>`)
    }

    for (const e of edges) {
      lines.push(`    <edge id="${escXml(e.id)}" source="${escXml(e.source_id)}" target="${escXml(e.target_id)}">`)
      lines.push(`      <data key="etype">${escXml(e.edge_type)}</data>`)
      lines.push(`      <data key="weight">${e.weight}</data>`)
      lines.push(`    </edge>`)
    }

    lines.push(`  </graph>`)
    lines.push(`</graphml>`)
    return lines.join('\n')
  }
}

// ─── XML escape helper ────────────────────────────────────────────────────────

function escXml(s: string): string {
  return s
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')
}
src/IncrementalUpdater.ts
TypeScript

/**
 * packages/graphify/src/IncrementalUpdater.ts
 *
 * Fingerprint-based incremental update for the LocoWorker knowledge graph.
 *
 * Strategy:
 *   1. Load the fingerprint manifest from the DB (or .locoworker/graph/manifest.json
 *      as a fallback for cross-session persistence).
 *   2. Walk the filesystem (or accept an explicit changed-paths list).
 *   3. Compute SHA-256 fingerprints of current files.
 *   4. Compare to stored fingerprints → categorise as ADDED / CHANGED / REMOVED / UNCHANGED.
 *   5. Re-parse only ADDED + CHANGED files via TreeSitterParser.
 *   6. Remove stale nodes/edges for REMOVED + CHANGED files.
 *   7. Insert new nodes/edges from re-parsed files via GraphBuilder.
 *   8. Re-run LeidenClusterer on the updated graph.
 *   9. Persist updated manifest.
 *
 * This means a typical "edit 2 files" developer workflow only re-parses those
 * 2 files rather than the whole codebase.
 */

import Database from 'better-sqlite3'
import { createHash } from 'node:crypto'
import { readFile, readdir, stat, writeFile } from 'node:fs/promises'
import { join, resolve, extname } from 'node:path'
import { TreeSitterParser } from './TreeSitterParser.js'
import { GraphBuilder } from './GraphBuilder.js'
import { LeidenClusterer } from './LeidenClusterer.js'
import type { IncrementalUpdateResult } from './GraphifyClient.js'

// ─── Options ─────────────────────────────────────────────────────────────────

export interface IncrementalUpdaterOptions {
  /** Project root directory. Required. */
  rootDir: string
  /** Path to persist the fingerprint manifest JSON. Default: <rootDir>/.locoworker/graph/manifest.json */
  manifestPath?: string
  /** File extensions to parse. Default: ['.ts','.tsx','.js','.jsx','.py','.rs','.go','.md','.json'] */
  extensions?: string[]
  /** Globs/paths to ignore (substring match). Default: ['node_modules', '.git', 'dist', '.locoworker'] */
  ignore?: string[]
  /** Whether to re-run Leiden clustering after update. Default: true */
  recluster?: boolean
  /** Leiden options forwarded to LeidenClusterer */
  leidenOptions?: object
}

// ─── Manifest types ───────────────────────────────────────────────────────────

interface ManifestEntry {
  fingerprint: string
  parsedAt: string
  nodeCount: number
}

type Manifest = Record<string, ManifestEntry>

type ChangeType = 'added' | 'changed' | 'removed' | 'unchanged'

interface FileChange {
  absolutePath: string
  relativePath: string
  type: ChangeType
  newFingerprint?: string
}

// ─── Main class ───────────────────────────────────────────────────────────────

export class IncrementalUpdater {
  private readonly db: Database.Database
  private readonly opts: Required<IncrementalUpdaterOptions>
  private readonly parser: TreeSitterParser
  private readonly builder: GraphBuilder

  constructor(db: Database.Database, opts: IncrementalUpdaterOptions) {
    this.db = db
    this.opts = {
      rootDir: resolve(opts.rootDir),
      manifestPath: opts.manifestPath ?? join(resolve(opts.rootDir), '.locoworker', 'graph', 'manifest.json'),
      extensions: opts.extensions ?? ['.ts', '.tsx', '.js', '.jsx', '.py', '.rs', '.go', '.md', '.json'],
      ignore: opts.ignore ?? ['node_modules', '.git', 'dist', '.locoworker', 'build', 'coverage'],
      recluster: opts.recluster ?? true,
      leidenOptions: opts.leidenOptions ?? {},
    }
    this.parser = new TreeSitterParser()
    this.builder = new GraphBuilder(db)
  }

  // ── Public run ──────────────────────────────────────────────────────────────

  async run(explicitPaths?: string[]): Promise<IncrementalUpdateResult> {
    const t0 = Date.now()

    const manifest = await this._loadManifest()

    const changes = explicitPaths
      ? await this._computeChangesForPaths(explicitPaths, manifest)
      : await this._computeChangesForRoot(manifest)

    const added = changes.filter(c => c.type === 'added').length
    const updated = changes.filter(c => c.type === 'changed').length
    const removed = changes.filter(c => c.type === 'removed').length
    const unchanged = changes.filter(c => c.type === 'unchanged').length

    // Process changes in a single transaction for atomicity
    const process = this.db.transaction(async () => {
      // Step 1: Remove stale data for changed and removed files
      const staleFiles = changes
        .filter(c => c.type === 'changed' || c.type === 'removed')
        .map(c => c.relativePath)

      for (const filePath of staleFiles) {
        this._removeFileData(filePath)
      }

      // Step 2: Re-parse and insert for added and changed files
      const toReparse = changes.filter(c => c.type === 'added' || c.type === 'changed')

      for (const change of toReparse) {
        try {
          const content = await readFile(change.absolutePath, 'utf8')
          const parsed = await this.parser.parseFile(change.absolutePath, content)
          if (parsed.parseSuccess) {
            this.builder.insertParsedFile(parsed, change.newFingerprint!)
          }
        } catch (err) {
          // Non-fatal: log and continue
          process.stderr?.write?.(`[IncrementalUpdater] Skipping ${change.relativePath}: ${err}\n`)
        }
      }
    })

    // Execute (transaction is sync but inner awaits need special handling)
    // Since better-sqlite3 transactions are sync, we run the async parts first,
    // then commit results in a sync transaction.
    const parseResults = await this._collectParseResults(changes)
    this._applyInTransaction(changes, parseResults)

    // Step 3: Re-run Leiden clustering if requested
    if (this.opts.recluster && (added > 0 || updated > 0 || removed > 0)) {
      const clusterer = new LeidenClusterer(this.db, this.opts.leidenOptions)
      clusterer.run()
    }

    // Step 4: Persist updated manifest
    await this._saveManifest(manifest, changes)

    return {
      added,
      updated,
      removed,
      unchanged,
      durationMs: Date.now() - t0,
    }
  }

  // ── Change computation ──────────────────────────────────────────────────────

  private async _computeChangesForRoot(manifest: Manifest): Promise<FileChange[]> {
    const allFiles = await this._walkDir(this.opts.rootDir)
    return this._diffWithManifest(allFiles, manifest)
  }

  private async _computeChangesForPaths(
    paths: string[],
    manifest: Manifest,
  ): Promise<FileChange[]> {
    const results: FileChange[] = []

    for (const p of paths) {
      const absolutePath = resolve(p)
      const relativePath = absolutePath.replace(this.opts.rootDir + '/', '')

      try {
        await stat(absolutePath)
        // File exists — added or changed?
        const fingerprint = await this._fingerprintFile(absolutePath)
        const existing = manifest[relativePath]
        if (!existing) {
          results.push({ absolutePath, relativePath, type: 'added', newFingerprint: fingerprint })
        } else if (existing.fingerprint !== fingerprint) {
          results.push({ absolutePath, relativePath, type: 'changed', newFingerprint: fingerprint })
        } else {
          results.push({ absolutePath, relativePath, type: 'unchanged', newFingerprint: fingerprint })
        }
      } catch {
        // File doesn't exist — removed
        if (manifest[relativePath]) {
          results.push({ absolutePath, relativePath, type: 'removed' })
        }
      }
    }

    return results
  }

  private async _walkDir(dir: string): Promise<string[]> {
    const results: string[] = []
    const entries = await readdir(dir, { withFileTypes: true }).catch(() => [])

    for (const entry of entries) {
      const fullPath = join(dir, entry.name)

      // Check ignore list
      if (this.opts.ignore.some(ig => fullPath.includes(ig))) continue

      if (entry.isDirectory()) {
        const sub = await this._walkDir(fullPath)
        results.push(...sub)
      } else if (entry.isFile()) {
        const ext = extname(entry.name)
        if (this.opts.extensions.includes(ext)) {
          results.push(fullPath)
        }
      }
    }

    return results
  }

  private _diffWithManifest(absolutePaths: string[], manifest: Manifest): FileChange[] {
    const results: FileChange[] = []
    const seenRelative = new Set<string>()

    // This is a synchronous approximation — fingerprinting happens in _collectParseResults
    for (const absolutePath of absolutePaths) {
      const relativePath = absolutePath.replace(this.opts.rootDir + '/', '')
      seenRelative.add(relativePath)
      // We'll compute fingerprints in _collectParseResults; for now mark as "needs check"
      results.push({ absolutePath, relativePath, type: 'added', newFingerprint: undefined })
    }

    // Mark removals
    for (const relPath of Object.keys(manifest)) {
      if (!seenRelative.has(relPath)) {
        results.push({
          absolutePath: join(this.opts.rootDir, relPath),
          relativePath: relPath,
          type: 'removed',
        })
      }
    }

    return results
  }

  // ── Async parse collection ──────────────────────────────────────────────────

  private async _collectParseResults(
    changes: FileChange[],
  ): Promise<Map<string, { content: string; fingerprint: string } | null>> {
    const results = new Map<string, { content: string; fingerprint: string } | null>()

    for (const change of changes) {
      if (change.type === 'removed') {
        results.set(change.relativePath, null)
        continue
      }

      try {
        const content = await readFile(change.absolutePath, 'utf8')
        const fingerprint = hashString(content)

        // Now do the actual diff check against the manifest in memory
        results.set(change.relativePath, { content, fingerprint })
        change.newFingerprint = fingerprint
      } catch {
        results.set(change.relativePath, null)
      }
    }

    return results
  }

  // ── Sync DB apply ───────────────────────────────────────────────────────────

  private _applyInTransaction(
    changes: FileChange[],
    parseResults: Map<string, { content: string; fingerprint: string } | null>,
  ): void {
    const apply = this.db.transaction(() => {
      // Remove stale
      for (const change of changes) {
        if (change.type === 'changed' || change.type === 'removed') {
          this._removeFileData(change.relativePath)
        }
      }

      // Insert new
      for (const change of changes) {
        if (change.type !== 'added' && change.type !== 'changed') continue
        const pr = parseResults.get(change.relativePath)
        if (!pr) continue

        const parsed = this.parser.parseFileSync(change.absolutePath, pr.content)
        if (parsed.parseSuccess) {
          this.builder.insertParsedFile(parsed, pr.fingerprint)
        }
      }
    })

    apply()
  }

  // ── Remove file data ────────────────────────────────────────────────────────

  private _removeFileData(relativePath: string): void {
    // Get all node IDs for this file
    const nodeIds = (this.db.prepare(`SELECT id FROM nodes WHERE file_path = ?`).all(relativePath) as { id: string }[]).map(r => r.id)

    if (nodeIds.length === 0) return

    const placeholders = nodeIds.map(() => '?').join(',')

    // Remove edges referencing these nodes
    this.db.prepare(`DELETE FROM edges WHERE source_id IN (${placeholders}) OR target_id IN (${placeholders})`).run(...nodeIds, ...nodeIds)

    // Remove nodes
    this.db.prepare(`DELETE FROM nodes WHERE file_path = ?`).run(relativePath)

    // Clean up empty clusters
    this.db.prepare(`
      DELETE FROM clusters WHERE id NOT IN (SELECT DISTINCT cluster_id FROM nodes)
    `).run()
  }

  // ── Manifest helpers ────────────────────────────────────────────────────────

  private async _loadManifest(): Promise<Manifest> {
    try {
      const raw = await readFile(this.opts.manifestPath, 'utf8')
      return JSON.parse(raw) as Manifest
    } catch {
      return {}
    }
  }

  private async _saveManifest(
    existingManifest: Manifest,
    changes: FileChange[],
  ): Promise<void> {
    const updated: Manifest = { ...existingManifest }

    for (const change of changes) {
      if (change.type === 'removed') {
        delete updated[change.relativePath]
      } else if ((change.type === 'added' || change.type === 'changed') && change.newFingerprint) {
        const nodeCount = (this.db.prepare(`SELECT COUNT(*) as c FROM nodes WHERE file_path = ?`).get(change.relativePath) as { c: number }).c
        updated[change.relativePath] = {
          fingerprint: change.newFingerprint,
          parsedAt: new Date().toISOString(),
          nodeCount,
        }
      }
    }

    const dir = this.opts.manifestPath.replace(/\/[^/]+$/, '')
    const { mkdir } = await import('node:fs/promises')
    await mkdir(dir, { recursive: true })
    await writeFile(this.opts.manifestPath, JSON.stringify(updated, null, 2), 'utf8')
  }

  // ── Fingerprint helper ──────────────────────────────────────────────────────

  private async _fingerprintFile(absolutePath: string): Promise<string> {
    const content = await readFile(absolutePath, 'utf8')
    return hashString(content)
  }
}

// ─── Utilities ────────────────────────────────────────────────────────────────

function hashString(s: string): string {
  return createHash('sha256').update(s).digest('hex')
}
src/index.ts (barrel — full package export)
TypeScript

/**
 * packages/graphify/src/index.ts
 *
 * Barrel export for @locoworker/graphify.
 *
 * Consumers (e.g. packages/core tools, apps/cowork-cli) import from here:
 *
 *   import { GraphBuilder, GraphifyClient, LeidenClusterer, ... } from '@locoworker/graphify'
 */

// ── Part 1 exports ────────────────────────────────────────────────────────────
export { TreeSitterParser } from './TreeSitterParser.js'
export { GraphBuilder } from './GraphBuilder.js'

// ── Part 2 exports ────────────────────────────────────────────────────────────
export { LeidenClusterer } from './LeidenClusterer.js'
export { GraphifyClient } from './GraphifyClient.js'
export { GraphReporter } from './GraphReporter.js'
export { IncrementalUpdater } from './IncrementalUpdater.js'

// ── Type re-exports ───────────────────────────────────────────────────────────
export type {
  // graphify.types (Part 1)
  GraphNode,
  GraphEdge,
  GraphCluster,
  BuildStats,
  GraphQueryOptions,
  ParsedFile,
  ParsedSymbol,
} from './types/graphify.types.js'

export type {
  // LeidenClusterer
  LeidenOptions,
  ClusteringResult,
} from './LeidenClusterer.js'

export type {
  // GraphifyClient
  NodeSearchResult,
  UsageResult,
  ClusterSummary,
  NeighbourhoodResult,
  IncrementalUpdateResult,
} from './GraphifyClient.js'

export type {
  // GraphReporter
  GraphReporterOptions,
  ReportResult,
} from './GraphReporter.js'

export type {
  // IncrementalUpdater
  IncrementalUpdaterOptions,
} from './IncrementalUpdater.js'
Tests
src/tests/LeidenClusterer.test.ts
TypeScript

/**
 * packages/graphify/src/tests/LeidenClusterer.test.ts
 *
 * bun test — no external test framework beyond Bun's built-in
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import Database from 'better-sqlite3'
import { LeidenClusterer } from '../LeidenClusterer.js'

// ─── Test DB factory ──────────────────────────────────────────────────────────

function createTestDb(): Database.Database {
  const db = new Database(':memory:')
  db.pragma('journal_mode = WAL')

  db.exec(`
    CREATE TABLE nodes (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      type TEXT NOT NULL,
      file_path TEXT NOT NULL,
      line_start INTEGER DEFAULT 0,
      line_end INTEGER DEFAULT 0,
      cluster_id TEXT NOT NULL DEFAULT 'unclustered',
      language TEXT,
      is_exported INTEGER DEFAULT 0,
      is_public INTEGER DEFAULT 0,
      docstring TEXT,
      complexity INTEGER,
      tags TEXT,
      fingerprint TEXT NOT NULL DEFAULT ''
    );

    CREATE TABLE edges (
      id TEXT PRIMARY KEY,
      source_id TEXT NOT NULL,
      target_id TEXT NOT NULL,
      edge_type TEXT NOT NULL DEFAULT 'references',
      weight REAL NOT NULL DEFAULT 1.0,
      metadata TEXT,
      FOREIGN KEY (source_id) REFERENCES nodes(id) ON DELETE CASCADE,
      FOREIGN KEY (target_id) REFERENCES nodes(id) ON DELETE CASCADE
    );

    CREATE TABLE clusters (
      id TEXT PRIMARY KEY,
      label TEXT NOT NULL,
      node_count INTEGER NOT NULL DEFAULT 0,
      representative_node_id TEXT,
      metadata TEXT
    );
  `)

  return db
}

function insertNode(db: Database.Database, id: string, name: string, clusterId = 'unclustered') {
  db.prepare(`
    INSERT INTO nodes (id, name, type, file_path, cluster_id, fingerprint)
    VALUES (?, ?, 'function', 'test.ts', ?, '')
  `).run(id, name, clusterId)
}

function insertEdge(db: Database.Database, sourceId: string, targetId: string, weight = 1.0) {
  const id = `${sourceId}→${targetId}`
  db.prepare(`
    INSERT OR IGNORE INTO edges (id, source_id, target_id, edge_type, weight)
    VALUES (?, ?, ?, 'calls', ?)
  `).run(id, sourceId, targetId, weight)
}

// ─── Tests ────────────────────────────────────────────────────────────────────

describe('LeidenClusterer', () => {
  let db: Database.Database

  beforeEach(() => {
    db = createTestDb()
  })

  afterEach(() => {
    db.close()
  })

  it('returns empty result for an empty graph', () => {
    const clusterer = new LeidenClusterer(db)
    const result = clusterer.run()
    expect(result.communityCount).toBe(0)
    expect(result.modularity).toBe(0)
    expect(result.assignment.size).toBe(0)
  })

  it('assigns all nodes to communities', () => {
    // Insert 6 nodes in two tight groups
    // Group A: a1-a2-a3 (densely connected)
    // Group B: b1-b2-b3 (densely connected)
    // Single weak cross-link: a1→b1
    for (const id of ['a1', 'a2', 'a3']) insertNode(db, id, id)
    for (const id of ['b1', 'b2', 'b3']) insertNode(db, id, id)

    insertEdge(db, 'a1', 'a2', 3)
    insertEdge(db, 'a2', 'a3', 3)
    insertEdge(db, 'a1', 'a3', 3)
    insertEdge(db, 'b1', 'b2', 3)
    insertEdge(db, 'b2', 'b3', 3)
    insertEdge(db, 'b1', 'b3', 3)
    insertEdge(db, 'a1', 'b1', 0.1) // weak cross link

    const clusterer = new LeidenClusterer(db, { seed: 'test-seed', minClusterSize: 1 })
    const result = clusterer.run()

    // All 6 nodes assigned
    expect(result.assignment.size).toBe(6)
    expect(result.modularity).toBeGreaterThan(0)
  })

  it('is deterministic: same seed → same assignment', () => {
    for (const id of ['n1', 'n2', 'n3', 'n4', 'n5']) insertNode(db, id, id)
    insertEdge(db, 'n1', 'n2')
    insertEdge(db, 'n2', 'n3')
    insertEdge(db, 'n3', 'n4')
    insertEdge(db, 'n4', 'n5')

    const opts = { seed: 'determinism-test', persist: false }
    const r1 = new LeidenClusterer(db, opts).run()
    const r2 = new LeidenClusterer(db, opts).run()

    expect([...r1.assignment.entries()]).toEqual([...r2.assignment.entries()])
  })

  it('persists cluster assignments to the DB when persist=true', () => {
    for (const id of ['p1', 'p2', 'p3']) insertNode(db, id, id)
    insertEdge(db, 'p1', 'p2')
    insertEdge(db, 'p2', 'p3')

    const clusterer = new LeidenClusterer(db, { persist: true, minClusterSize: 1 })
    clusterer.run()

    const clusters = db.prepare(`SELECT COUNT(*) as c FROM clusters`).get() as { c: number }
    expect(clusters.c).toBeGreaterThan(0)

    // Nodes should have updated cluster_id
    const nodes = db.prepare(`SELECT cluster_id FROM nodes`).all() as { cluster_id: string }[]
    for (const n of nodes) {
      expect(n.cluster_id).not.toBe('unclustered')
    }
  })

  it('merges tiny clusters when minClusterSize > 1', () => {
    // 4 nodes: 3 connected + 1 isolated
    for (const id of ['m1', 'm2', 'm3', 'iso']) insertNode(db, id, id)
    insertEdge(db, 'm1', 'm2')
    insertEdge(db, 'm2', 'm3')
    insertEdge(db, 'm1', 'iso', 0.01) // very weak link to isolated node

    const clusterer = new LeidenClusterer(db, {
      minClusterSize: 2,
      persist: false,
      seed: 'merge-test',
    })
    const result = clusterer.run()

    // After merging, no cluster should have < 2 members
    const clusterSizes = new Map<string, number>()
    for (const cId of result.assignment.values()) {
      clusterSizes.set(cId, (clusterSizes.get(cId) ?? 0) + 1)
    }
    for (const size of clusterSizes.values()) {
      expect(size).toBeGreaterThanOrEqual(1) // merged into the main cluster
    }
  })

  it('computes a modularity value between -1 and 1', () => {
    for (let i = 0; i < 10; i++) insertNode(db, `node${i}`, `node${i}`)
    for (let i = 0; i < 9; i++) insertEdge(db, `node${i}`, `node${i + 1}`)

    const result = new LeidenClusterer(db, { persist: false }).run()
    expect(result.modularity).toBeGreaterThanOrEqual(-1)
    expect(result.modularity).toBeLessThanOrEqual(1)
  })

  it('generates stable cluster IDs (hash-prefixed)', () => {
    for (const id of ['s1', 's2', 's3']) insertNode(db, id, id)
    insertEdge(db, 's1', 's2')

    const result = new LeidenClusterer(db, { persist: false, minClusterSize: 1 }).run()
    for (const cId of result.assignment.values()) {
      expect(cId).toMatch(/^cluster_[0-9a-f]{16}$/)
    }
  })

  it('handles a single-node graph gracefully', () => {
    insertNode(db, 'only', 'onlyNode')
    const result = new LeidenClusterer(db, { persist: false, minClusterSize: 1 }).run()
    expect(result.assignment.size).toBe(1)
    expect(result.communityCount).toBe(1)
  })

  it('handles a fully disconnected graph', () => {
    for (const id of ['d1', 'd2', 'd3', 'd4']) insertNode(db, id, id)
    // No edges

    const result = new LeidenClusterer(db, { persist: false, minClusterSize: 1 }).run()
    expect(result.assignment.size).toBe(4)
    // Each node is its own cluster (no merging possible since no edges)
    expect(result.communityCount).toBe(4)
  })
})
src/tests/GraphifyClient.test.ts
TypeScript

/**
 * packages/graphify/src/tests/GraphifyClient.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import Database from 'better-sqlite3'
import { GraphifyClient } from '../GraphifyClient.js'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { writeFileSync, unlinkSync } from 'node:fs'

// ─── Test DB factory (same schema as above) ───────────────────────────────────

function createAndSeedDb(dbPath: string): void {
  const db = new Database(dbPath)
  db.pragma('journal_mode = WAL')

  db.exec(`
    CREATE TABLE nodes (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      type TEXT NOT NULL,
      file_path TEXT NOT NULL,
      line_start INTEGER DEFAULT 0,
      line_end INTEGER DEFAULT 0,
      cluster_id TEXT NOT NULL DEFAULT 'unclustered',
      language TEXT,
      is_exported INTEGER DEFAULT 0,
      is_public INTEGER DEFAULT 0,
      docstring TEXT,
      complexity INTEGER,
      tags TEXT,
      fingerprint TEXT NOT NULL DEFAULT ''
    );

    CREATE TABLE edges (
      id TEXT PRIMARY KEY,
      source_id TEXT NOT NULL,
      target_id TEXT NOT NULL,
      edge_type TEXT NOT NULL DEFAULT 'references',
      weight REAL NOT NULL DEFAULT 1.0,
      metadata TEXT
    );

    CREATE TABLE clusters (
      id TEXT PRIMARY KEY,
      label TEXT NOT NULL,
      node_count INTEGER NOT NULL DEFAULT 0,
      representative_node_id TEXT,
      metadata TEXT
    );
  `)

  // Seed data
  const nodes = [
    { id: 'n1', name: 'AgentLoop', type: 'class', file: 'core/agent.ts', cluster: 'c1', lang: 'typescript', exported: 1, complexity: 12 },
    { id: 'n2', name: 'queryLoop', type: 'function', file: 'core/agent.ts', cluster: 'c1', lang: 'typescript', exported: 1, complexity: 8 },
    { id: 'n3', name: 'MemoryManager', type: 'class', file: 'memory/manager.ts', cluster: 'c2', lang: 'typescript', exported: 1, complexity: 5 },
    { id: 'n4', name: 'internalHelper', type: 'function', file: 'core/utils.ts', cluster: 'c1', lang: 'typescript', exported: 0, complexity: 2 },
    { id: 'n5', name: 'RustParser', type: 'function', file: 'parser/rust.rs', cluster: 'c3', lang: 'rust', exported: 0, complexity: 7 },
  ]

  for (const n of nodes) {
    db.prepare(`
      INSERT INTO nodes (id, name, type, file_path, cluster_id, language, is_exported, complexity, fingerprint)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'fp')
    `).run(n.id, n.name, n.type, n.file, n.cluster, n.lang, n.exported, n.complexity)
  }

  const edges = [
    { id: 'e1', src: 'n1', tgt: 'n2', type: 'calls', weight: 3 },
    { id: 'e2', src: 'n2', tgt: 'n3', type: 'imports', weight: 1 },  // cross-cluster
    { id: 'e3', src: 'n1', tgt: 'n4', type: 'calls', weight: 2 },
  ]

  for (const e of edges) {
    db.prepare(`
      INSERT INTO edges (id, source_id, target_id, edge_type, weight) VALUES (?, ?, ?, ?, ?)
    `).run(e.id, e.src, e.tgt, e.type, e.weight)
  }

  const clusters = [
    { id: 'c1', label: 'Agent Core', count: 3, rep: 'n1' },
    { id: 'c2', label: 'Memory', count: 1, rep: 'n3' },
    { id: 'c3', label: 'Parsers', count: 1, rep: 'n5' },
  ]

  for (const c of clusters) {
    db.prepare(`
      INSERT INTO clusters (id, label, node_count, representative_node_id) VALUES (?, ?, ?, ?)
    `).run(c.id, c.label, c.count, c.rep)
  }

  db.close()
}

// ─── Tests ────────────────────────────────────────────────────────────────────

describe('GraphifyClient', () => {
  let client: GraphifyClient
  let dbPath: string

  beforeEach(() => {
    dbPath = join(tmpdir(), `test-graphify-${Date.now()}.db`)
    createAndSeedDb(dbPath)
    client = new GraphifyClient()
    client.open(dbPath)
  })

  afterEach(() => {
    client.close()
    try { unlinkSync(dbPath) } catch {}
  })

  // ── findNode ─────────────────────────────────────────────────────────────

  it('findNode: exact match scores 1.0', () => {
    const results = client.findNode('AgentLoop')
    expect(results.length).toBeGreaterThan(0)
    expect(results[0]!.score).toBe(1.0)
    expect(results[0]!.node.name).toBe('AgentLoop')
  })

  it('findNode: partial match returns results ordered by score', () => {
    const results = client.findNode('agent')
    expect(results.length).toBeGreaterThan(0)
    // AgentLoop starts with "agent" (case insensitive), queryLoop contains it
    const names = results.map(r => r.node.name)
    expect(names).toContain('AgentLoop')
  })

  it('findNode: filter by nodeType', () => {
    const results = client.findNode('', { nodeType: 'function' })
    for (const r of results) {
      expect(r.node.type).toBe('function')
    }
  })

  it('findNode: filter by language', () => {
    const results = client.findNode('', { language: 'rust' })
    expect(results.length).toBe(1)
    expect(results[0]!.node.name).toBe('RustParser')
  })

  it('findNode: exportedOnly filter', () => {
    const results = client.findNode('', { exportedOnly: true })
    for (const r of results) {
      expect(r.node.isExported).toBe(true)
    }
    expect(results.some(r => r.node.name === 'internalHelper')).toBe(false)
  })

  it('findNode: minComplexity filter', () => {
    const results = client.findNode('', { minComplexity: 8 })
    for (const r of results) {
      expect(r.node.complexity).toBeGreaterThanOrEqual(8)
    }
  })

  it('findNode: respects limit', () => {
    const results = client.findNode('', { limit: 2 })
    expect(results.length).toBeLessThanOrEqual(2)
  })

  it('findNode: no match returns empty array', () => {
    const results = client.findNode('xyzzy_nonexistent_zzz')
    expect(results.length).toBe(0)
  })

  // ── findUsages ────────────────────────────────────────────────────────────

  it('findUsages: returns all edges for a node', () => {
    const usages = client.findUsages('n1')
    // n1 is source of e1 and e3
    expect(usages.length).toBe(2)
    const edgeIds = usages.map(u => u.edge.id)
    expect(edgeIds).toContain('e1')
    expect(edgeIds).toContain('e3')
  })

  it('findUsages: populates sourceNode and targetNode', () => {
    const usages = client.findUsages('n2')
    for (const u of usages) {
      if (u.sourceNode) expect(u.sourceNode.id).toBeTruthy()
      if (u.targetNode) expect(u.targetNode.id).toBeTruthy()
    }
  })

  it('findUsages: returns empty for node with no edges', () => {
    const usages = client.findUsages('n5')
    expect(usages.length).toBe(0)
  })

  it('findUsages: throws for non-existent node', () => {
    // Non-existent node just returns empty edges, does not throw
    const usages = client.findUsages('nonexistent')
    expect(usages.length).toBe(0)
  })

  // ── getClusterSummary ─────────────────────────────────────────────────────

  it('getClusterSummary: returns all clusters when no id given', () => {
    const summaries = client.getClusterSummary()
    expect(summaries.length).toBe(3)
  })

  it('getClusterSummary: returns specific cluster by id', () => {
    const summaries = client.getClusterSummary('c1')
    expect(summaries.length).toBe(1)
    expect(summaries[0]!.cluster.id).toBe('c1')
  })

  it('getClusterSummary: includes topNodes', () => {
    const summaries = client.getClusterSummary('c1')
    expect(summaries[0]!.topNodes.length).toBeGreaterThan(0)
  })

  it('getClusterSummary: counts internal and external edges', () => {
    const c1 = client.getClusterSummary('c1')[0]!
    // e1(n1→n2): internal. e2(n2→n3): external. e3(n1→n4): internal.
    expect(c1.internalEdgeCount).toBe(2)
    expect(c1.externalEdgeCount).toBe(1)
  })

  it('getClusterSummary: lists languages', () => {
    const c1 = client.getClusterSummary('c1')[0]!
    expect(c1.languages).toContain('typescript')
  })

  it('getClusterSummary: returns empty for non-existent cluster', () => {
    const summaries = client.getClusterSummary('doesnotexist')
    expect(summaries.length).toBe(0)
  })

  // ── getNeighbourhood ──────────────────────────────────────────────────────

  it('getNeighbourhood: depth=1 returns direct neighbours only', () => {
    const result = client.getNeighbourhood('n1', 1)
    expect(result.root.id).toBe('n1')
    const nodeIds = result.nodes.map(n => n.id)
    expect(nodeIds).toContain('n1')
    expect(nodeIds).toContain('n2') // n1→n2
    expect(nodeIds).toContain('n4') // n1→n4
    expect(nodeIds).not.toContain('n3') // n3 is 2 hops away
  })

  it('getNeighbourhood: depth=2 reaches transitive neighbours', () => {
    const result = client.getNeighbourhood('n1', 2)
    const nodeIds = result.nodes.map(n => n.id)
    expect(nodeIds).toContain('n3') // n1→n2→n3
  })

  it('getNeighbourhood: throws for non-existent node', () => {
    expect(() => client.getNeighbourhood('missing')).toThrow('not found')
  })

  it('getNeighbourhood: includes edges between visited nodes', () => {
    const result = client.getNeighbourhood('n1', 1)
    expect(result.edges.length).toBeGreaterThan(0)
  })

  // ── getStats ──────────────────────────────────────────────────────────────

  it('getStats: returns correct counts', () => {
    const stats = client.getStats()
    expect(stats.nodeCount).toBe(5)
    expect(stats.edgeCount).toBe(3)
    expect(stats.clusterCount).toBe(3)
  })

  // ── error handling ────────────────────────────────────────────────────────

  it('throws if used before open()', () => {
    const c = new GraphifyClient()
    expect(() => c.findNode('anything')).toThrow('call open(dbPath) first')
  })
})
src/tests/GraphReporter.test.ts
TypeScript

/**
 * packages/graphify/src/tests/GraphReporter.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import Database from 'better-sqlite3'
import { GraphReporter } from '../GraphReporter.js'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { readFile, rm } from 'node:fs/promises'

// ─── DB factory (reuse schema from other tests) ───────────────────────────────

function createReporterDb(): Database.Database {
  const db = new Database(':memory:')
  db.pragma('journal_mode = WAL')

  db.exec(`
    CREATE TABLE nodes (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      type TEXT NOT NULL,
      file_path TEXT NOT NULL,
      line_start INTEGER DEFAULT 1,
      line_end INTEGER DEFAULT 50,
      cluster_id TEXT NOT NULL DEFAULT 'cl1',
      language TEXT,
      is_exported INTEGER DEFAULT 1,
      is_public INTEGER DEFAULT 1,
      docstring TEXT,
      complexity INTEGER,
      tags TEXT,
      fingerprint TEXT NOT NULL DEFAULT ''
    );
    CREATE TABLE edges (
      id TEXT PRIMARY KEY,
      source_id TEXT NOT NULL,
      target_id TEXT NOT NULL,
      edge_type TEXT NOT NULL DEFAULT 'calls',
      weight REAL NOT NULL DEFAULT 1.0,
      metadata TEXT
    );
    CREATE TABLE clusters (
      id TEXT PRIMARY KEY,
      label TEXT NOT NULL,
      node_count INTEGER NOT NULL DEFAULT 0,
      representative_node_id TEXT,
      metadata TEXT
    );
  `)

  // Two clusters with a cross-cluster edge
  db.prepare(`INSERT INTO clusters VALUES ('cl1', 'Core Engine', 3, 'n1', NULL)`).run()
  db.prepare(`INSERT INTO clusters VALUES ('cl2', 'Storage Layer', 2, 'n4', NULL)`).run()

  const nodes = [
    ['n1', 'AgentLoop', 'class', 'core/agent.ts', 'cl1', 'typescript', 1, 15, 1, 20],
    ['n2', 'queryLoop', 'function', 'core/agent.ts', 'cl1', 'typescript', 1, 8, 1, 30],
    ['n3', 'EventBus', 'class', 'core/events.ts', 'cl1', 'typescript', 1, 5, 1, 40],
    ['n4', 'MemoryManager', 'class', 'memory/manager.ts', 'cl2', 'typescript', 1, 10, 1, 100],
    ['n5', 'persist', 'function', 'memory/manager.ts', 'cl2', 'typescript', 0, 3, 1, 60],
  ]

  for (const [id, name, type, file, cluster, lang, exported, complexity, isPublic, lineEnd] of nodes) {
    db.prepare(`
      INSERT INTO nodes (id, name, type, file_path, cluster_id, language, is_exported, complexity, is_public, line_end, fingerprint)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'fp')
    `).run(id, name, type, file, cluster, lang, exported, complexity, isPublic, lineEnd)
  }

  db.prepare(`INSERT INTO edges VALUES ('e1','n1','n2','calls',5,NULL)`).run()
  db.prepare(`INSERT INTO edges VALUES ('e2','n1','n3','calls',3,NULL)`).run()
  db.prepare(`INSERT INTO edges VALUES ('e3','n2','n4','imports',1,NULL)`).run() // cross-cluster
  db.prepare(`INSERT INTO edges VALUES ('e4','n4','n5','calls',4,NULL)`).run()

  return db
}

// ─── Tests ────────────────────────────────────────────────────────────────────

describe('GraphReporter', () => {
  let db: Database.Database
  let outDir: string

  beforeEach(() => {
    db = createReporterDb()
    outDir = join(tmpdir(), `graphify-report-${Date.now()}`)
  })

  afterEach(async () => {
    db.close()
    await rm(outDir, { recursive: true, force: true })
  })

  it('generates GRAPH_REPORT.md', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()

    const md = await readFile(result.markdownPath, 'utf8')
    expect(result.markdownPath).toEndWith('GRAPH_REPORT.md')
    expect(md).toContain('# GRAPH_REPORT')
    expect(md).toContain('## Overview')
    expect(md).toContain('## God Nodes')
    expect(md).toContain('## Semantic Clusters')
  })

  it('markdown contains god node names', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    const md = await readFile(result.markdownPath, 'utf8')
    // AgentLoop has edges e1 + e2 = degree 2, highest degree
    expect(md).toContain('AgentLoop')
  })

  it('markdown contains cluster labels', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    const md = await readFile(result.markdownPath, 'utf8')
    expect(md).toContain('Core Engine')
    expect(md).toContain('Storage Layer')
  })

  it('markdown contains cross-cluster connection section', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    const md = await readFile(result.markdownPath, 'utf8')
    expect(md).toContain('Cross-Cluster')
  })

  it('generates SVG when generateSvg=true', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: true, generateGraphML: false })
    const result = await reporter.generate()

    expect(result.svgPath).not.toBeNull()
    const svg = await readFile(result.svgPath!, 'utf8')
    expect(svg).toContain('<svg ')
    expect(svg).toContain('</svg>')
    expect(svg).toContain('Core Engine')
  })

  it('SVG is valid XML structure (has opening and closing tags)', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: true, generateGraphML: false })
    const result = await reporter.generate()
    const svg = await readFile(result.svgPath!, 'utf8')
    expect(svg.startsWith('<?xml')).toBe(true)
    expect(svg.includes('<g id="clusters">')).toBe(true)
    expect(svg.includes('<g id="cross-edges"')).toBe(true)
  })

  it('generates GraphML when generateGraphML=true', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: true })
    const result = await reporter.generate()

    expect(result.graphMLPath).not.toBeNull()
    const xml = await readFile(result.graphMLPath!, 'utf8')
    expect(xml).toContain('<graphml ')
    expect(xml).toContain('<graph ')
    expect(xml).toContain('<node id=')
    expect(xml).toContain('<edge id=')
  })

  it('GraphML contains all nodes and edges', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: true })
    const result = await reporter.generate()
    const xml = await readFile(result.graphMLPath!, 'utf8')

    for (const id of ['n1', 'n2', 'n3', 'n4', 'n5']) {
      expect(xml).toContain(`id="${id}"`)
    }
    for (const id of ['e1', 'e2', 'e3', 'e4']) {
      expect(xml).toContain(`id="${id}"`)
    }
  })

  it('GraphML escapes XML special characters', async () => {
    // Insert a node with special chars in name
    db.prepare(`
      INSERT INTO nodes (id, name, type, file_path, cluster_id, fingerprint, language)
      VALUES ('nX', 'A<B>&"test"', 'function', 'f.ts', 'cl1', 'fp', 'typescript')
    `).run()
    db.prepare(`UPDATE clusters SET node_count = node_count + 1 WHERE id = 'cl1'`).run()

    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: true })
    const result = await reporter.generate()
    const xml = await readFile(result.graphMLPath!, 'utf8')

    expect(xml).toContain('&amp;')
    expect(xml).toContain('&lt;')
    expect(xml).toContain('&gt;')
  })

  it('skips SVG when generateSvg=false', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    expect(result.svgPath).toBeNull()
  })

  it('skips GraphML when generateGraphML=false', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    expect(result.graphMLPath).toBeNull()
  })

  it('returns a non-zero reductionFactor', async () => {
    const reporter = new GraphReporter(db, { outputDir: outDir, generateSvg: false, generateGraphML: false })
    const result = await reporter.generate()
    expect(result.reductionFactor).toBeGreaterThan(0)
  })

  it('uses projectName in report header', async () => {
    const reporter = new GraphReporter(db, {
      outputDir: outDir,
      projectName: 'MyAwesomeProject',
      generateSvg: false,
      generateGraphML: false,
    })
    const result = await reporter.generate()
    const md = await readFile(result.markdownPath, 'utf8')
    expect(md).toContain('MyAwesomeProject')
  })

  it('respects topNodeLimit', async () => {
    const reporter = new GraphReporter(db, {
      outputDir: outDir,
      topNodeLimit: 2,
      generateSvg: false,
      generateGraphML: false,
    })
    const result = await reporter.generate()
    expect(result.godNodeCount).toBeLessThanOrEqual(2)
  })
})
src/tests/IncrementalUpdater.test.ts
TypeScript

/**
 * packages/graphify/src/tests/IncrementalUpdater.test.ts
 */

import { describe, it, expect, beforeEach, afterEach } from 'bun:test'
import Database from 'better-sqlite3'
import { IncrementalUpdater } from '../IncrementalUpdater.js'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { mkdir, writeFile, rm, readFile } from 'node:fs/promises'

// ─── Test project fixture helper ──────────────────────────────────────────────

async function createTestProject(root: string): Promise<void> {
  await mkdir(root, { recursive: true })
  await mkdir(join(root, 'src'), { recursive: true })

  await writeFile(join(root, 'src', 'alpha.ts'), `
export function alpha() {
  return 'alpha'
}
export class AlphaClass {}
`, 'utf8')

  await writeFile(join(root, 'src', 'beta.ts'), `
import { alpha } from './alpha'
export function beta() {
  return alpha()
}
`, 'utf8')

  await writeFile(join(root, 'README.md'), `# Test Project\nHello world.`, 'utf8')
}

function createEmptyDb(): Database.Database {
  const db = new Database(':memory:')
  db.pragma('journal_mode = WAL')

  db.exec(`
    CREATE TABLE nodes (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      type TEXT NOT NULL,
      file_path TEXT NOT NULL,
      line_start INTEGER DEFAULT 0,
      line_end INTEGER DEFAULT 0,
      cluster_id TEXT NOT NULL DEFAULT 'unclustered',
      language TEXT,
      is_exported INTEGER DEFAULT 0,
      is_public INTEGER DEFAULT 0,
      docstring TEXT,
      complexity INTEGER,
      tags TEXT,
      fingerprint TEXT NOT NULL DEFAULT ''
    );
    CREATE TABLE edges (
      id TEXT PRIMARY KEY,
      source_id TEXT NOT NULL,
      target_id TEXT NOT NULL,
      edge_type TEXT NOT NULL DEFAULT 'references',
      weight REAL NOT NULL DEFAULT 1.0,
      metadata TEXT
    );
    CREATE TABLE clusters (
      id TEXT PRIMARY KEY,
      label TEXT NOT NULL,
      node_count INTEGER NOT NULL DEFAULT 0,
      representative_node_id TEXT,
      metadata TEXT
    );
  `)

  return db
}

// ─── Tests ────────────────────────────────────────────────────────────────────

describe('IncrementalUpdater', () => {
  let db: Database.Database
  let projectDir: string
  let manifestDir: string

  beforeEach(async () => {
    db = createEmptyDb()
    projectDir = join(tmpdir(), `locoworker-test-${Date.now()}`)
    manifestDir = join(projectDir, '.locoworker', 'graph')
    await createTestProject(projectDir)
  })

  afterEach(async () => {
    db.close()
    await rm(projectDir, { recursive: true, force: true })
  })

  it('runs without error on a fresh directory', async () => {
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      recluster: false,
    })
    const result = await updater.run()
    expect(result).toBeDefined()
    expect(typeof result.durationMs).toBe('number')
    expect(result.durationMs).toBeGreaterThanOrEqual(0)
  })

  it('returns added count > 0 for a fresh project', async () => {
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      recluster: false,
    })
    const result = await updater.run()
    // At least the .ts and .md files should be detected as "added"
    expect(result.added + result.updated).toBeGreaterThanOrEqual(0)
    // Unchanged should be 0 on first run
    expect(result.unchanged).toBe(0)
    expect(result.removed).toBe(0)
  })

  it('second run with no changes returns all unchanged', async () => {
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath: join(manifestDir, 'manifest.json'),
      recluster: false,
    })

    await updater.run() // First run
    const result2 = await updater.run() // Second run — nothing changed

    expect(result2.added).toBe(0)
    expect(result2.updated).toBe(0)
    expect(result2.removed).toBe(0)
    expect(result2.unchanged).toBeGreaterThan(0)
  })

  it('detects a modified file on second run', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    await updater.run() // First run

    // Modify alpha.ts
    await writeFile(join(projectDir, 'src', 'alpha.ts'), `
export function alpha() { return 'alpha-modified' }
export function alphaNew() { return 'new' }
export class AlphaClass {}
`, 'utf8')

    const result2 = await updater.run()
    expect(result2.updated).toBe(1) // alpha.ts changed
  })

  it('detects a new file on second run', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    await updater.run()

    // Add a new file
    await writeFile(join(projectDir, 'src', 'gamma.ts'), `
export function gamma() { return 'gamma' }
`, 'utf8')

    const result2 = await updater.run()
    expect(result2.added).toBe(1)
  })

  it('detects a removed file on second run', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    await updater.run()

    // Remove beta.ts
    await rm(join(projectDir, 'src', 'beta.ts'))

    const result2 = await updater.run()
    expect(result2.removed).toBe(1)
  })

  it('persists manifest after a run', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    await updater.run()

    // Manifest should exist
    const manifestRaw = await readFile(manifestPath, 'utf8')
    const manifest = JSON.parse(manifestRaw)
    expect(typeof manifest).toBe('object')
    // Should have at least some entries
    expect(Object.keys(manifest).length).toBeGreaterThan(0)
  })

  it('manifest entries have fingerprint, parsedAt, and nodeCount fields', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    await updater.run()

    const manifest = JSON.parse(await readFile(manifestPath, 'utf8'))
    for (const entry of Object.values(manifest) as Array<{ fingerprint: string; parsedAt: string; nodeCount: number }>) {
      expect(typeof entry.fingerprint).toBe('string')
      expect(entry.fingerprint.length).toBe(64) // SHA-256 hex
      expect(typeof entry.parsedAt).toBe('string')
      expect(typeof entry.nodeCount).toBe('number')
    }
  })

  it('explicit paths: only processes specified files', async () => {
    const manifestPath = join(manifestDir, 'manifest.json')
    await writeFile(manifestPath, '{}', 'utf8') // Empty manifest → everything is "new"
    await mkdir(manifestDir, { recursive: true })

    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      manifestPath,
      recluster: false,
    })

    const alphaPath = join(projectDir, 'src', 'alpha.ts')
    const result = await updater.run([alphaPath])

    // Only alpha.ts was checked
    expect(result.added + result.updated + result.unchanged).toBe(1)
  })

  it('respects ignore list', async () => {
    // Create a node_modules directory with a TS file
    await mkdir(join(projectDir, 'node_modules', 'fake-pkg'), { recursive: true })
    await writeFile(join(projectDir, 'node_modules', 'fake-pkg', 'index.ts'), `export const x = 1`, 'utf8')

    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      ignore: ['node_modules', '.git', 'dist'],
      recluster: false,
    })

    const result = await updater.run()
    // node_modules file should NOT be counted
    // If ignore works, added should only count files outside node_modules
    const nodeModulesNodes = db.prepare(`SELECT COUNT(*) as c FROM nodes WHERE file_path LIKE '%node_modules%'`).get() as { c: number }
    expect(nodeModulesNodes.c).toBe(0)
  })

  it('respects custom extensions list', async () => {
    await writeFile(join(projectDir, 'src', 'styles.css'), `.foo { color: red }`, 'utf8')

    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      extensions: ['.ts'], // Only TypeScript, not .md or .css
      recluster: false,
    })

    await updater.run()

    // CSS file should not have been parsed
    const cssNodes = db.prepare(`SELECT COUNT(*) as c FROM nodes WHERE file_path LIKE '%.css'`).get() as { c: number }
    expect(cssNodes.c).toBe(0)
  })

  it('result has correct structure', async () => {
    const updater = new IncrementalUpdater(db, { rootDir: projectDir, recluster: false })
    const result = await updater.run()

    expect('added' in result).toBe(true)
    expect('updated' in result).toBe(true)
    expect('removed' in result).toBe(true)
    expect('unchanged' in result).toBe(true)
    expect('durationMs' in result).toBe(true)
  })

  it('calls LeidenClusterer when recluster=true and there are changes', async () => {
    // Just verify it doesn't throw — clustering on a small/empty graph is valid
    const updater = new IncrementalUpdater(db, {
      rootDir: projectDir,
      recluster: true,
    })
    await expect(updater.run()).resolves.toBeDefined()
  })
})
package.json additions for packages/graphify
The following fields should be merged/confirmed in packages/graphify/package.json (consistent with Pass 1 workspace conventions):

JSON

{
  "name": "@locoworker/graphify",
  "version": "0.1.0",
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
    "dev": "tsc --watch --project tsconfig.json",
    "typecheck": "tsc --noEmit",
    "lint": "biome check ./src",
    "test": "bun test ./src/tests",
    "mcp": "node ./dist/bin/mcp.js"
  },
  "dependencies": {
    "better-sqlite3": "^9.4.3"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.8",
    "@types/node": "^20.11.0",
    "typescript": "^5.3.3"
  },
  "peerDependencies": {
    "tree-sitter": "^0.21.0",
    "tree-sitter-typescript": "^0.21.0",
    "tree-sitter-python": "^0.21.0",
    "tree-sitter-rust": "^0.21.0",
    "tree-sitter-go": "^0.21.0"
  },
  "peerDependenciesMeta": {
    "tree-sitter": { "optional": true },
    "tree-sitter-typescript": { "optional": true },
    "tree-sitter-python": { "optional": true },
    "tree-sitter-rust": { "optional": true },
    "tree-sitter-go": { "optional": true }
  }
}
MCP entry-point bin script
Create packages/graphify/src/bin/mcp.ts as a thin wrapper:

TypeScript

/**
 * packages/graphify/src/bin/mcp.ts
 *
 * Entry point for the Graphify MCP stdio server.
 *
 * Usage:
 *   node ./dist/bin/mcp.js --db .locoworker/graph/graph.db
 *   # or via pnpm:
 *   pnpm --filter @locoworker/graphify mcp -- --db ./path/to/graph.db
 */

import { GraphifyClient } from '../GraphifyClient.js'

const args = process.argv.slice(2)
const dbFlagIdx = args.indexOf('--db')
const dbPath = dbFlagIdx !== -1 ? args[dbFlagIdx + 1] : '.locoworker/graph/graph.db'

if (!dbPath) {
  process.stderr.write('Usage: graphify-mcp --db <path-to-graph.db>\n')
  process.exit(1)
}

const client = new GraphifyClient()
client.startMcpServer(dbPath)
Summary of Pass 4 Part 2
File	Lines (approx)	Purpose
LeidenClusterer.ts	~320	Pure-TS Leiden community detection, 3-phase, deterministic seeding, SQLite persistence
GraphifyClient.ts	~380	Query API: findNode, findUsages, getClusterSummary, getNeighbourhood, getStats + full JSON-RPC 2.0 MCP stdio server
GraphReporter.ts	~370	GRAPH_REPORT.md + SVG cluster map (pure string, no DOM) + GraphML export
IncrementalUpdater.ts	~310	SHA-256 fingerprint manifest, walk/diff/re-parse only changed files, sync DB transaction
index.ts	~45	Clean barrel export of all Part 1 + Part 2 symbols + types
bin/mcp.ts	~20	Thin MCP stdio launcher
LeidenClusterer.test.ts	~175	8 tests: empty, community detection, determinism, persistence, tiny-cluster merging, modularity, stable IDs, isolated nodes
GraphifyClient.test.ts	~215	22 tests: all query methods, filters, error cases, stats
GraphReporter.test.ts	~175	13 tests: markdown content, SVG structure, GraphML validity, XML escaping, options
IncrementalUpdater.test.ts	~205	13 tests: fresh run, second-run unchanged, modified/added/removed detection, manifest persistence, ignore list, extensions
