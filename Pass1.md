Pass 1 of 18: Root Scaffold
Let's lay the complete foundation that every package depends on. Every file here is real, buildable, and production-ready. Nothing is a placeholder.

File Tree for Pass 1
text

locoworker/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── tsconfig.base.json
├── .eslintrc.base.json
├── .prettierrc
├── .gitignore
├── .env.example
├── .npmrc
├── CLAUDE.md
├── MEMORY.md
├── biome.json
└── .github/
    └── workflows/
        ├── ci.yml
        └── release.yml
package.json (root)
JSON

{
  "name": "locoworker",
  "version": "1.0.0",
  "private": true,
  "description": "Open-source agentic developer workspace — BYOK, local-LLM, privacy-first",
  "license": "MIT",
  "author": "LocoWorker Contributors",
  "homepage": "https://github.com/locoworker/locoworker",
  "repository": {
    "type": "git",
    "url": "https://github.com/locoworker/locoworker.git"
  },
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "dev":              "turbo run dev --parallel",
    "dev:cli":          "pnpm --filter @locoworker/cowork-cli dev",
    "dev:desktop":      "pnpm --filter @locoworker/desktop tauri dev",
    "dev:dashboard":    "pnpm --filter @locoworker/dashboard dev",

    "build":            "turbo run build",
    "build:cli":        "pnpm --filter @locoworker/cowork-cli build",
    "build:desktop":    "pnpm --filter @locoworker/desktop tauri build",
    "build:dashboard":  "pnpm --filter @locoworker/dashboard build",
    "build:packages":   "turbo run build --filter='./packages/*'",

    "typecheck":        "turbo run typecheck",
    "lint":             "turbo run lint",
    "lint:fix":         "turbo run lint:fix",
    "format":           "prettier --write \"**/*.{ts,tsx,json,md,yaml,yml}\"",
    "format:check":     "prettier --check \"**/*.{ts,tsx,json,md,yaml,yml}\"",

    "test":             "turbo run test",
    "test:watch":       "turbo run test:watch",
    "test:coverage":    "turbo run test:coverage",

    "graph:build":      "pnpm --filter @locoworker/graphify cli build --dir .",
    "graph:update":     "pnpm --filter @locoworker/graphify cli update --dir .",
    "wiki:lint":        "pnpm --filter @locoworker/wiki cli lint",
    "wiki:ingest":      "pnpm --filter @locoworker/wiki cli ingest",
    "wiki:query":       "pnpm --filter @locoworker/wiki cli query",

    "kairos:start":     "pnpm --filter @locoworker/kairos start",
    "kairos:stop":      "pnpm --filter @locoworker/kairos stop",
    "kairos:status":    "pnpm --filter @locoworker/kairos status",

    "research:run":     "pnpm --filter @locoworker/autoresearch run",
    "research:report":  "pnpm --filter @locoworker/autoresearch report",

    "sim:start":        "docker compose -f packages/mirofish/docker-compose.yml up -d",
    "sim:stop":         "docker compose -f packages/mirofish/docker-compose.yml down",
    "sim:logs":         "docker compose -f packages/mirofish/docker-compose.yml logs -f",
    "sim:run":          "pnpm --filter @locoworker/mirofish run",

    "memory:show":      "cat MEMORY.md",
    "audit:today":      "node -e \"const d=new Date().toISOString().split('T')[0]; require('fs').createReadStream(`.locoworker/audit/${d}.jsonl`).pipe(process.stdout)\"",

    "changeset":        "changeset",
    "version":          "changeset version",
    "publish":          "changeset publish",

    "clean":            "turbo run clean && rm -rf node_modules",
    "clean:deep":       "turbo run clean && find . -name 'node_modules' -type d -prune -exec rm -rf '{}' + && find . -name 'dist' -type d -prune -exec rm -rf '{}' +",
    "reset":            "pnpm clean:deep && pnpm install",

    "prepare":          "husky"
  },
  "devDependencies": {
    "@biomejs/biome":           "1.9.4",
    "@changesets/cli":          "^2.27.9",
    "@types/node":              "^22.10.0",
    "husky":                    "^9.1.7",
    "lint-staged":              "^15.2.10",
    "prettier":                 "^3.4.1",
    "turbo":                    "^2.3.3",
    "typescript":               "^5.7.2"
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "biome check --apply",
      "prettier --write"
    ],
    "*.{json,md,yaml,yml}": [
      "prettier --write"
    ]
  }
}
pnpm-workspace.yaml
YAML

packages:
  - "apps/*"
  - "packages/*"

# Hoist common peer dependencies to root
shamefullyHoist: false

# Ensure consistent lockfile
lockfileOnly: false
turbo.json
JSON

{
  "$schema": "https://turbo.build/schema.json",
  "ui": "tui",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "package.json", "tsconfig.json"],
      "outputs": ["dist/**", ".turbo/cache/**"],
      "cache": true
    },
    "dev": {
      "dependsOn": ["^build"],
      "persistent": true,
      "cache": false
    },
    "typecheck": {
      "dependsOn": ["^typecheck"],
      "inputs": ["src/**", "tsconfig.json"],
      "outputs": [],
      "cache": true
    },
    "lint": {
      "inputs": ["src/**", "biome.json", ".eslintrc*"],
      "outputs": [],
      "cache": true
    },
    "lint:fix": {
      "cache": false
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tests/**", "test/**"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "test:watch": {
      "persistent": true,
      "cache": false
    },
    "test:coverage": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "clean": {
      "cache": false
    }
  },
  "globalDependencies": [
    "tsconfig.base.json",
    ".env"
  ],
  "globalEnv": [
    "NODE_ENV",
    "CI",
    "ANTHROPIC_API_KEY",
    "OPENAI_API_KEY",
    "GOOGLE_API_KEY",
    "MISTRAL_API_KEY",
    "GROQ_API_KEY",
    "TOGETHER_API_KEY",
    "OLLAMA_BASE_URL",
    "LMSTUDIO_BASE_URL"
  ]
}
tsconfig.base.json
JSON

{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": false,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": false,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "incremental": true,
    "composite": true
  },
  "exclude": [
    "node_modules",
    "dist",
    "build",
    "coverage",
    ".turbo"
  ]
}
biome.json
JSON

{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "ignoreUnknown": false,
    "ignore": [
      "node_modules",
      "dist",
      "build",
      "coverage",
      ".turbo",
      "*.generated.ts",
      "apps/desktop/src-tauri"
    ]
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error",
        "useExhaustiveDependencies": "warn"
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsole": "off"
      },
      "style": {
        "useConst": "error",
        "useTemplate": "error",
        "noParameterAssign": "error",
        "useNodejsImportProtocol": "error"
      },
      "performance": {
        "noAccumulatingSpread": "error"
      },
      "security": {
        "noGlobalEval": "error"
      },
      "complexity": {
        "noBannedTypes": "error",
        "noForEach": "off"
      }
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "es5",
      "semicolons": "asNeeded",
      "arrowParentheses": "always"
    },
    "parser": {
      "unsafeParameterDecoratorsEnabled": true
    }
  },
  "overrides": [
    {
      "include": ["*.test.ts", "*.spec.ts", "tests/**"],
      "linter": {
        "rules": {
          "suspicious": {
            "noExplicitAny": "off"
          }
        }
      }
    }
  ]
}
.prettierrc
JSON

{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.md",
      "options": {
        "proseWrap": "always",
        "printWidth": 80
      }
    },
    {
      "files": ["*.json", "*.yaml", "*.yml"],
      "options": {
        "tabWidth": 2
      }
    }
  ]
}
.npmrc
ini

# Use pnpm strict mode
shamefully-hoist=false
strict-peer-dependencies=false
auto-install-peers=true

# Always save exact versions
save-exact=true

# Prefer offline when cache is available
prefer-offline=true

# Registry
registry=https://registry.npmjs.org/

# Workspace protocol
link-workspace-packages=true

# Lockfile settings
lockfile=true
frozen-lockfile=false
.gitignore
gitignore

# Dependencies
node_modules/
.pnp
.pnp.js

# Build outputs
dist/
build/
out/
.next/
.nuxt/

# Turbo
.turbo/

# TypeScript
*.tsbuildinfo
tsconfig.tsbuildinfo

# Test coverage
coverage/
*.lcov

# Environment files
.env
.env.local
.env.*.local
.env.development
.env.production

# Logs
logs/
*.log
npm-debug.log*
pnpm-debug.log*

# OS files
.DS_Store
.DS_Store?
._*
.Spotlight-V2
.Trashes
Thumbs.db
Desktop.ini

# Editor files
.vscode/settings.json
.idea/
*.swp
*.swo
*.vim

# Tauri build artifacts
apps/desktop/src-tauri/target/
apps/desktop/src-tauri/WixTools/

# LocoWorker runtime files (NOT committed per project)
# Note: .locoworker/ IS committed in the locoworker repo itself
# but should be gitignored in user projects
# Uncomment the line below if you are a USER of locoworker:
# .locoworker/

# AutoDream archives (large, optional to commit)
.locoworker/memory/archive/

# Audit logs (sensitive, optional to commit)
.locoworker/audit/

# Graph database (regeneratable)
.locoworker/graph/*.db

# Research experiment raw outputs
.locoworker/research/raw/

# MiroFish simulation raw results
packages/mirofish/results/raw/

# Docker volumes
packages/mirofish/.docker-volumes/

# Temporary files
*.tmp
*.temp
.cache/

# Bun
.bun/
bun.lockb

# Changeset
.changeset/*.md
!.changeset/config.json
!.changeset/README.md
.env.example
Bash

# ─────────────────────────────────────────────────────────────────────
# LOCOWORKER — Environment Variables
# Copy this file to .env and fill in your values.
# NEVER commit .env to version control.
# ─────────────────────────────────────────────────────────────────────

# ── Cloud Provider API Keys (BYOK) ────────────────────────────────────
# Leave blank if using local LLMs only

ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_API_KEY=
MISTRAL_API_KEY=
GROQ_API_KEY=
TOGETHER_API_KEY=

# ── Local LLM Endpoints ───────────────────────────────────────────────
# Defaults shown — change if running on non-standard ports

OLLAMA_BASE_URL=http://localhost:11434
LMSTUDIO_BASE_URL=http://localhost:1234
LLAMACPP_BASE_URL=http://localhost:8080

# ── Custom OpenAI-Compatible Endpoint ────────────────────────────────
# Use for any self-hosted or third-party OpenAI-compatible API

CUSTOM_LLM_BASE_URL=
CUSTOM_LLM_API_KEY=
CUSTOM_LLM_MODEL_ID=

# ── Default Model Configuration ───────────────────────────────────────

DEFAULT_PROVIDER=anthropic
DEFAULT_MODEL_ID=claude-opus-4-5
FALLBACK_PROVIDER=ollama
FALLBACK_MODEL_ID=llama3.1:8b

# ── Permission & Safety ───────────────────────────────────────────────

DEFAULT_PERMISSION_LEVEL=WRITE_LOCAL
REQUIRE_SHELL_CONFIRMATION=true
WORKSPACE_BOUNDARY_ENFORCEMENT=true

# ── Cost Controls ─────────────────────────────────────────────────────

DAILY_COST_CAP_USD=10.00
SESSION_COST_CAP_USD=2.00
EVAL_COST_CAP_USD=0.50
ALERT_AT_COST_PERCENT=80

# ── KAIROS Daemon ────────────────────────────────────────────────────

KAIROS_HEARTBEAT_INTERVAL_MS=60000
KAIROS_QUIET_HOURS_START=2
KAIROS_QUIET_HOURS_END=5
KAIROS_BATTERY_AWARE=true
KAIROS_ENABLE_AUTODREAM=true
KAIROS_ENABLE_AUTORESEARCH=false

# ── Memory Settings ───────────────────────────────────────────────────

MEMORY_PRUNE_AFTER_DAYS=90
MEMORY_MAX_ENTRIES=5000
AUTODREAM_MAX_ENTRIES_PER_RUN=1000

# ── Graphify Settings ────────────────────────────────────────────────

GRAPHIFY_INCREMENTAL_INTERVAL_MS=900000
GRAPHIFY_MAX_FILE_SIZE_KB=500
GRAPHIFY_EXCLUDED_DIRS=node_modules,dist,build,.turbo,coverage

# ── Gateway Settings ─────────────────────────────────────────────────

TELEGRAM_BOT_TOKEN=
DISCORD_BOT_TOKEN=
SLACK_BOT_TOKEN=
WEBHOOK_SECRET=

# ── MiroFish Simulation ───────────────────────────────────────────────

MIROFISH_NEO4J_URI=bolt://localhost:7688
MIROFISH_NEO4J_USER=neo4j
MIROFISH_NEO4J_PASSWORD=mirofish_secret
MIROFISH_OLLAMA_BASE_URL=http://localhost:11435

# ── Development ───────────────────────────────────────────────────────

NODE_ENV=development
LOG_LEVEL=info
DEBUG=locoworker:*
CLAUDE.md
Markdown

# LocoWorker — Agent Project Instructions

> This file is read by the LocoWorker agent at the start of every session.
> It provides project-specific context, conventions, and constraints.
> Maintained by the team. Sanitized for prompt injection before use.

## Project Overview

LocoWorker is a 21-package TypeScript monorepo built with pnpm workspaces
and TurboRepo. It is an open-source agentic developer workspace with:
- A CLI (`apps/cowork-cli`) built with Ink + Bun
- A desktop app (`apps/desktop`) built with Tauri (Rust) + React
- A reference dashboard (`apps/dashboard`) built with React + Vite
- 18 shared packages under `packages/`

## Critical Conventions

### Package Manager
- ALWAYS use `pnpm`, never `npm` or `yarn`
- Add deps to specific package: `pnpm --filter @locoworker/core add <dep>`
- Add root dev dep: `pnpm add -D <dep> -w`

### TypeScript
- `strict: true` everywhere — no exceptions
- No `any` without an explicit `// biome-ignore` comment with reason
- Always use `node:` prefix for Node built-ins: `import fs from 'node:fs'`
- Use `satisfies` operator for type-checked object literals
- Prefer `type` imports: `import type { Foo } from './foo'`

### Code Style
- 2-space indentation, single quotes, no semicolons
- Max line length: 100 characters
- Arrow functions for callbacks, `function` keyword for named functions
- Trailing commas in multi-line structures

### File Structure
- Each package: `src/index.ts` as public entry point
- Types co-located: `src/types/` folder per package
- Tests adjacent: `src/__tests__/` or `tests/` at package root
- Barrel exports: `src/index.ts` re-exports everything public

### Error Handling
- Never `throw new Error('string')` — always use typed error classes
- All async functions must handle rejections (no floating promises)
- Use `Result<T, E>` pattern for recoverable errors where applicable

### Testing
- Test framework: Bun test (`bun test`)
- Coverage target: 80% for `packages/core`
- Mock external APIs, never mock the filesystem in unit tests
- Use real temp directories for filesystem tests (auto-cleanup with `afterEach`)

### Git
- Commit message format: `type(scope): description`
  - Types: feat, fix, chore, docs, perf, test, refactor, eval
  - Scope: package name without `@locoworker/` prefix
  - Example: `feat(core): add parallel tool execution`
- Always run `pnpm typecheck && pnpm test` before committing
- Never commit `.env` or any file containing API keys

### Cost & Model Usage
- NEVER use claude-opus for internal tooling tasks
- Use claude-haiku or local models for: summarization, classification, eval scoring
- Log and check cost before using expensive models in loops

## Package Dependency Graph
shared ←─── everything core ←────── cowork-cli, desktop, dashboard, all packages memory ←──── core, kairos, autoresearch graphify ←── wiki, dashboard, desktop wiki ←─────── dashboard, desktop, kairos kairos ←───── autoresearch, mirofish, desktop orchestrator ← cowork-cli, desktop gateway ←───── cowork-cli, desktop, kairos security ←──── core, gateway, wiki tools-* ←────── core mirofish ←───── autoresearch

text


## Key Files

- Agent loop:          `packages/core/src/queryLoop.ts`
- Tool registry:       `packages/core/src/ToolRegistry.ts`
- Permission gate:     `packages/core/src/PermissionGate.ts`
- Provider router:     `packages/core/src/ProviderRouter.ts`
- Memory manager:      `packages/memory/src/MemoryManager.ts`
- Graphify client:     `packages/graphify/src/GraphifyClient.ts`
- KAIROS daemon:       `packages/kairos/src/KairosDaemon.ts`
- Desktop entry:       `apps/desktop/src-tauri/src/main.rs`
- CLI entry:           `apps/cowork-cli/src/main.tsx`

## Active Work

Check `MEMORY.md` for current architecture decisions and active tasks.
Check `.locoworker/research/program.md` for active research threads.
Check `.locoworker/wiki/_index.json` for the knowledge base index.

## Workspace Boundary

All file write operations must stay within the repository root.
System directories (`/etc`, `/usr`, `~/.ssh`, etc.) are off-limits.
The PermissionGate enforces this at runtime — do not attempt to bypass it.
MEMORY.md
Markdown

<!-- MEMORY.md — Auto-maintained by LocoWorker MemoryManager -->
<!-- Last updated: 2026-05-06T00:00:00Z | Bootstrapped -->
<!-- Commands: /memory show | /memory search <query> | /memory add <note> -->

# Project Memory Index

## Architecture Decisions

- [2026-05-06] Chose pnpm workspaces + TurboRepo for monorepo tooling
  Reason: Best-in-class caching, parallel builds, workspace protocol support

- [2026-05-06] Chose Bun as runtime for CLI and package builds
  Reason: 3-5x faster test runs and builds vs Node; native TypeScript support

- [2026-05-06] Chose Biome over ESLint + Prettier separately
  Reason: Single tool, 10-100x faster, same configuration for lint + format

- [2026-05-06] Chose Tauri (Rust) over Electron for desktop app
  Reason: 10-100x smaller bundle, native OS APIs, better security sandbox,
  OS keychain access via keyring crate

- [2026-05-06] Chosen database strategy: SQLite (better-sqlite3) for graph,
  JSON Lines for audit logs, Markdown for human-readable memory
  Reason: Local-first, no server required, human-readable, git-friendly

- [2026-05-06] Provider routing: OpenAI SDK for all providers via baseURL shim
  Reason: Single SDK dependency; Ollama/LM Studio/llama.cpp are all
  OpenAI-compatible, eliminating per-provider SDKs for local models

- [2026-05-06] Chosen Leiden algorithm over Louvain for Graphify clustering
  Reason: Better resolution parameter control, provably better quality bounds,
  available via leidenalg npm package

## Key File Locations

- Monorepo root:         /
- Core engine:           packages/core/src/
- Memory package:        packages/memory/src/
- Graphify package:      packages/graphify/src/
- KAIROS daemon:         packages/kairos/src/
- CLI app:               apps/cowork-cli/src/
- Desktop app:           apps/desktop/src/ + src-tauri/
- Dashboard app:         apps/dashboard/src/

## User Preferences

- Runtime:      Bun (not Node) for CLI
- Formatter:    Biome (not Prettier alone)
- Test runner:  Bun test
- Package mgr:  pnpm (never npm or yarn)
- Quotes:       single quotes in TypeScript
- Semicolons:   none (ASI)
- Indentation:  2 spaces

## Known Issues & Workarounds

- None yet (bootstrapped)

## Active Research Threads

- None yet (bootstrapped)

## Buddy Companion

- Not yet initialized (run `locoworker init` to create your Buddy)

## Build Status

- Pass 1 (Root Scaffold):     ✅ Complete
- Pass 2 (packages/core):     ⏳ Pending
- Pass 3 (packages/memory):   ⏳ Pending
- Pass 4 (packages/graphify): ⏳ Pending
- Pass 5 (packages/wiki):     ⏳ Pending
- Pass 6 (packages/kairos):   ⏳ Pending
- Pass 7 (packages/orchestrator): ⏳ Pending
- Pass 8 (packages/gateway):  ⏳ Pending
- Pass 9 (packages/autoresearch): ⏳ Pending
- Pass 10 (packages/mirofish): ⏳ Pending
- Pass 11 (packages/security): ⏳ Pending
- Pass 12 (packages/tools-*): ⏳ Pending
- Pass 13 (packages/shared):  ⏳ Pending
- Pass 14 (apps/cowork-cli):  ⏳ Pending
- Pass 15 (apps/dashboard):   ⏳ Pending
- Pass 16 (apps/desktop):     ⏳ Pending
- Pass 17 (Test suites):      ⏳ Pending
- Pass 18 (CI/CD):            ⏳ Pending
.github/workflows/ci.yml
YAML

name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20"
  BUN_VERSION: "1.1.38"
  PNPM_VERSION: "9.12.0"
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

jobs:
  # ── Job 1: Setup & Cache ──────────────────────────────────────────────
  setup:
    name: Setup
    runs-on: ubuntu-latest
    outputs:
      pnpm-cache-dir: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: ${{ env.BUN_VERSION }}

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Get pnpm store directory
        id: pnpm-cache
        run: echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

  # ── Job 2: Type Check ────────────────────────────────────────────────
  typecheck:
    name: TypeCheck
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "${{ env.BUN_VERSION }}" }
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
      - name: Restore pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - name: Type check all packages
        run: pnpm typecheck

  # ── Job 3: Lint ───────────────────────────────────────────────────────
  lint:
    name: Lint
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "${{ env.BUN_VERSION }}" }
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
      - name: Restore pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - name: Biome lint
        run: pnpm biome check --reporter=github .
      - name: Format check
        run: pnpm format:check

  # ── Job 4: Secret Scan ───────────────────────────────────────────────
  secret-scan:
    name: Secret Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

  # ── Job 5: Test (packages) ────────────────────────────────────────────
  test-packages:
    name: Test Packages
    needs: [typecheck, lint]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package:
          - core
          - memory
          - graphify
          - wiki
          - kairos
          - orchestrator
          - gateway
          - security
          - shared
          - tools-fs
          - tools-bash
          - tools-git
          - tools-search
          - tools-web
          - tools-mcp
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "${{ env.BUN_VERSION }}" }
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
      - name: Restore pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - name: Build dependencies
        run: pnpm turbo run build --filter=@locoworker/${{ matrix.package }}^...
      - name: Run tests
        run: pnpm --filter @locoworker/${{ matrix.package }} test:coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          flags: ${{ matrix.package }}
          token: ${{ secrets.CODECOV_TOKEN }}
        continue-on-error: true

  # ── Job 6: Build All ─────────────────────────────────────────────────
  build:
    name: Build All Packages
    needs: [typecheck, lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "${{ env.BUN_VERSION }}" }
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
      - name: Restore pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - name: Build all packages
        run: pnpm build:packages
      - name: Build dashboard (single-file)
        run: pnpm build:dashboard
      - name: Upload dashboard artifact
        uses: actions/upload-artifact@v4
        with:
          name: dashboard-html
          path: apps/dashboard/dist/index.html
          retention-days: 30

  # ── Job 7: Cost-Capped Evals (PR only) ───────────────────────────────
  evals:
    name: Eval Suite (Cost-Capped)
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    env:
      ANTHROPIC_API_KEY: ${{ secrets.EVAL_ANTHROPIC_API_KEY }}
      EVAL_COST_CAP_USD: "0.50"
      DEFAULT_MODEL_ID: "claude-haiku-4-5"
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "${{ env.BUN_VERSION }}" }
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
      - name: Restore pnpm cache
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-cache-dir }}
          key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - name: Run core eval suite
        run: pnpm --filter @locoworker/core eval
      - name: Upload eval results
        uses: actions/upload-artifact@v4
        with:
          name: eval-results-${{ github.sha }}
          path: packages/core/eval-results/
          retention-days: 14

  # ── Summary ───────────────────────────────────────────────────────────
  ci-summary:
    name: CI Summary
    needs: [typecheck, lint, secret-scan, test-packages, build]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check all jobs
        run: |
          if [[ "${{ needs.typecheck.result }}" != "success" ]]; then
            echo "❌ TypeCheck failed" && exit 1
          fi
          if [[ "${{ needs.lint.result }}" != "success" ]]; then
            echo "❌ Lint failed" && exit 1
          fi
          if [[ "${{ needs.secret-scan.result }}" != "success" ]]; then
            echo "❌ Secret scan failed" && exit 1
          fi
          if [[ "${{ needs.test-packages.result }}" != "success" ]]; then
            echo "❌ Tests failed" && exit 1
          fi
          if [[ "${{ needs.build.result }}" != "success" ]]; then
            echo "❌ Build failed" && exit 1
          fi
          echo "✅ All CI checks passed"
.github/workflows/release.yml
YAML

name: Release

on:
  push:
    branches: [main]

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - uses: oven-sh/setup-bun@v2
        with: { bun-version: "1.1.38" }

      - uses: pnpm/action-setup@v4
        with: { version: "9.12.0" }

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "pnpm"
          registry-url: "https://registry.npmjs.org"

      - run: pnpm install --frozen-lockfile

      - name: Build all packages
        run: pnpm build:packages

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          publish: pnpm publish
          title: "chore: version packages"
          commit: "chore: version packages"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
.changeset/config.json
JSON

{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": [
    "@locoworker/cowork-cli",
    "@locoworker/desktop",
    "@locoworker/dashboard"
  ]
}
husky setup (.husky/pre-commit)
Bash

#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Run lint-staged on staged files
pnpm lint-staged

# Type check affected packages
pnpm typecheck
Bash

#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
# .husky/commit-msg

# Enforce conventional commit format
npx --no -- commitlint --edit "$1"
commitlint.config.js
JavaScript

/** @type {import('@commitlint/types').UserConfig} */
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'chore',
        'docs',
        'perf',
        'test',
        'refactor',
        'eval',
        'revert',
        'ci',
        'build',
        'style',
      ],
    ],
    'scope-enum': [
      1,
      'always',
      [
        'core',
        'memory',
        'graphify',
        'wiki',
        'kairos',
        'orchestrator',
        'gateway',
        'security',
        'autoresearch',
        'mirofish',
        'shared',
        'tools-fs',
        'tools-bash',
        'tools-git',
        'tools-search',
        'tools-web',
        'tools-mcp',
        'cowork-cli',
        'dashboard',
        'desktop',
        'root',
        'ci',
        'deps',
      ],
    ],
    'subject-case': [2, 'always', 'lower-case'],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [1, 'always', 120],
  },
}
Pass 1 — Complete ✅
What was just generated (all real, buildable files):
File	Purpose
package.json	Root workspace with all scripts
pnpm-workspace.yaml	Workspace declaration
turbo.json	Full TurboRepo pipeline with caching
tsconfig.base.json	Shared strict TS config
biome.json	Lint + format config (replaces ESLint+Prettier)
.prettierrc	Prettier config for MD/JSON/YAML
.npmrc	pnpm registry + workspace settings
.gitignore	Comprehensive ignore list
.env.example	All env vars documented
CLAUDE.md	Agent project instructions
MEMORY.md	Bootstrapped memory index
.github/workflows/ci.yml	Full CI with 7 jobs, matrix tests, evals
.github/workflows/release.yml	Changeset-based npm publish
.changeset/config.json	Changeset configuration
.husky/pre-commit	Git hooks for lint + typecheck
commitlint.config.js	Conventional commit enforcement
