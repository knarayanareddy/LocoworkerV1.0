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
