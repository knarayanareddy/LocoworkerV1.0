Pass 16 — Part 1: Root Config Normalization

File tree (Pass 16 Part 1 — files touched)
text

locoworker/
├── .env.example                   ← REPLACED (complete final)
├── .gitignore                     ← REPLACED (complete final)
├── .nvmrc                         ← REPLACED (complete final)
├── biome.json                     ← REPLACED (complete final)
├── .prettierrc                    ← REPLACED (complete final)
├── package.json                   ← REPLACED (complete final, no tauri/cowork-cli refs)
├── pnpm-workspace.yaml            ← REPLACED (complete final, all 16 packages + 4 apps + tests)
├── tsconfig.base.json             ← REPLACED (complete final)
├── tsconfig.json                  ← REPLACED (complete final)
└── turbo.json                     ← REPLACED (complete final)
pnpm-workspace.yaml
YAML

# ─────────────────────────────────────────────────────────
# LocoWorker monorepo — pnpm workspace config
# Pass 16 Part 1 — canonical / complete final version
# Rule: later pass wins; this replaces the Pass 1 version.
# ─────────────────────────────────────────────────────────

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

  # ── Research + simulation (depend on core + capability packages)
  - "packages/autoresearch"
  - "packages/mirofish"

  # ── Tool packages (depend on core + shared + security)
  - "packages/tools-fs"
  - "packages/tools-bash"
  - "packages/tools-git"
  - "packages/tools-search"
  - "packages/tools-web"

  # ── Applications (depend on all packages above)
  - "apps/cli"
  - "apps/gateway"
  - "apps/desktop"
  - "apps/dashboard"

  # ── Test suites (not published; depend on all apps + packages)
  - "tests/integration"
  - "tests/e2e"
turbo.json
JSON

{
  "$schema": "https://turbo.build/schema.json",
  "globalEnv": [
    "NODE_ENV",
    "LOG_LEVEL",
    "LOCOWORKER_WORKSPACE_ROOT",
    "ANTHROPIC_API_KEY",
    "OPENAI_API_KEY",
    "GEMINI_API_KEY",
    "MISTRAL_API_KEY",
    "GROQ_API_KEY",
    "OLLAMA_BASE_URL",
    "LMSTUDIO_BASE_URL",
    "LLAMACPP_BASE_URL",
    "DEFAULT_PROVIDER",
    "DEFAULT_MODEL",
    "LOCOWORKER_COST_CAP_USD",
    "GATEWAY_PORT",
    "GATEWAY_JWT_SECRET",
    "GATEWAY_CORS_ORIGINS",
    "MEMORY_SQLITE_PATH",
    "GRAPHIFY_SQLITE_PATH",
    "WIKI_SQLITE_PATH",
    "KAIROS_SQLITE_PATH",
    "SECURITY_SQLITE_PATH",
    "ENABLE_KAIROS",
    "ENABLE_AUTORESEARCH",
    "ENABLE_MIROFISH",
    "ENABLE_ORCHESTRATOR",
    "CI"
  ],
  "pipeline": {
    "topo": {
      "dependsOn": ["^topo"]
    },

    "build": {
      "dependsOn": ["^build"],
      "inputs": [
        "src/**",
        "tsconfig*.json",
        "package.json",
        "vite.config.*",
        "electron.vite.config.*"
      ],
      "outputs": ["dist/**", ".next/**", "out/**"],
      "cache": true
    },

    "typecheck": {
      "dependsOn": ["^typecheck"],
      "inputs": ["src/**", "tsconfig*.json", "package.json"],
      "outputs": [],
      "cache": true
    },

    "lint": {
      "inputs": ["src/**", "biome.json", ".prettierrc", "package.json"],
      "outputs": [],
      "cache": true
    },

    "format": {
      "inputs": ["src/**", "biome.json", ".prettierrc"],
      "outputs": [],
      "cache": false
    },

    "test": {
      "dependsOn": ["build"],
      "inputs": [
        "src/**",
        "tests/**",
        "__tests__/**",
        "vitest.config.*",
        "tsconfig*.json"
      ],
      "outputs": ["coverage/**"],
      "cache": true,
      "env": ["CI"]
    },

    "test:integration": {
      "dependsOn": ["^build"],
      "inputs": ["tests/integration/**", "vitest.config.*"],
      "outputs": ["coverage/integration/**"],
      "cache": false
    },

    "test:e2e": {
      "dependsOn": ["^build"],
      "inputs": ["tests/e2e/**", "vitest.config.*"],
      "outputs": [],
      "cache": false
    },

    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },

    "dev:packages": {
      "dependsOn": [],
      "cache": false,
      "persistent": true
    },

    "clean": {
      "cache": false
    },

    "db:migrate": {
      "cache": false
    },

    "db:seed": {
      "dependsOn": ["db:migrate"],
      "cache": false
    }
  }
}
package.json (root)
JSON

{
  "name": "locoworker-monorepo",
  "version": "0.1.0",
  "private": true,
  "description": "LocoWorker — open-source agentic developer workspace (monorepo root)",
  "license": "MIT",
  "author": "LocoWorker Contributors",
  "homepage": "https://github.com/knarayanareddy/LocoworkerV1.0",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/knarayanareddy/LocoworkerV1.0.git"
  },
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.12.0",
  "scripts": {
    "─── dev ──────────────────────────────────────────": "",
    "dev": "turbo run dev --parallel",
    "dev:packages": "turbo run dev:packages --parallel --filter='./packages/*'",
    "dev:cli": "pnpm --filter @locoworker/cli dev",
    "dev:gateway": "pnpm --filter @locoworker/gateway-app dev",
    "dev:desktop": "pnpm --filter @locoworker/desktop dev",
    "dev:dashboard": "pnpm --filter @locoworker/dashboard dev",

    "─── build ────────────────────────────────────────": "",
    "build": "turbo run build",
    "build:packages": "turbo run build --filter='./packages/*'",
    "build:apps": "turbo run build --filter='./apps/*'",
    "build:cli": "pnpm --filter @locoworker/cli build",
    "build:gateway": "pnpm --filter @locoworker/gateway-app build",
    "build:desktop": "pnpm --filter @locoworker/desktop build",
    "build:dashboard": "pnpm --filter @locoworker/dashboard build",

    "─── typecheck ────────────────────────────────────": "",
    "typecheck": "turbo run typecheck",
    "typecheck:packages": "turbo run typecheck --filter='./packages/*'",

    "─── lint / format ────────────────────────────────": "",
    "lint": "turbo run lint",
    "lint:fix": "biome check --write . && prettier --write .",
    "format": "biome format --write . && prettier --write .",
    "format:check": "biome format . && prettier --check .",

    "─── test ─────────────────────────────────────────": "",
    "test": "turbo run test",
    "test:packages": "turbo run test --filter='./packages/*'",
    "test:integration": "turbo run test:integration --filter='./tests/integration'",
    "test:e2e": "turbo run test:e2e --filter='./tests/e2e'",
    "test:ci": "turbo run test test:integration --concurrency=4",
    "coverage": "turbo run test -- --coverage",

    "─── run (production) ─────────────────────────────": "",
    "cli": "node apps/cli/dist/index.js",
    "cli:dev": "pnpm --filter @locoworker/cli start:dev",
    "gateway:start": "node apps/gateway/dist/index.js",
    "desktop:start": "pnpm --filter @locoworker/desktop start",

    "─── db ───────────────────────────────────────────": "",
    "db:migrate": "turbo run db:migrate",
    "db:seed": "turbo run db:seed",

    "─── tooling ──────────────────────────────────────": "",
    "clean": "turbo run clean && rimraf node_modules .turbo",
    "clean:dist": "turbo run clean",
    "clean:full": "pnpm clean && pnpm store prune",
    "env:check": "node scripts/env-check.mjs",
    "prepare": "node scripts/prepare.mjs",
    "preinstall": "npx only-allow pnpm",
    "changeset": "changeset",
    "changeset:version": "changeset version",
    "changeset:publish": "changeset publish",
    "release": "pnpm build && changeset publish"
  },
  "devDependencies": {
    "@changesets/cli": "^2.27.7",
    "@types/node": "^20.14.0",
    "biome": "^1.8.3",
    "prettier": "^3.3.3",
    "rimraf": "^6.0.1",
    "turbo": "^2.1.3",
    "typescript": "^5.5.4",
    "vitest": "^2.0.5"
  },
  "pnpm": {
    "overrides": {
      "typescript": "^5.5.4"
    },
    "peerDependencyRules": {
      "ignoreMissing": [
        "react",
        "react-dom",
        "@types/react"
      ]
    }
  }
}
tsconfig.base.json
JSON

{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "LocoWorker base TypeScript config (Pass 16 Part 1 — canonical)",
  "compilerOptions": {
    "─── language / emit ──────────────────────────────": "",
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": false,

    "─── output ───────────────────────────────────────": "",
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "inlineSources": false,
    "removeComments": false,

    "─── type checking (strict) ───────────────────────": "",
    "strict": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitAny": true,
    "noImplicitThis": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true,
    "allowUnusedLabels": false,
    "allowUnreachableCode": false,

    "─── module resolution helpers ────────────────────": "",
    "forceConsistentCasingInFileNames": true,
    "allowImportingTsExtensions": false,
    "isolatedModules": true,

    "─── paths (override in each package's tsconfig) ──": "",
    "baseUrl": ".",
    "paths": {}
  },
  "exclude": [
    "node_modules",
    "dist",
    "coverage",
    "**/*.test.ts",
    "**/*.spec.ts",
    "**/__tests__/**"
  ]
}
tsconfig.json (root — references all packages + apps)
JSON

{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "LocoWorker root tsconfig — project references (Pass 16 Part 1)",
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
    { "path": "packages/autoresearch" },
    { "path": "packages/mirofish" },
    { "path": "packages/tools-fs" },
    { "path": "packages/tools-bash" },
    { "path": "packages/tools-git" },
    { "path": "packages/tools-search" },
    { "path": "packages/tools-web" },
    { "path": "apps/cli" },
    { "path": "apps/gateway" },
    { "path": "apps/desktop" },
    { "path": "apps/dashboard" },
    { "path": "tests/integration" },
    { "path": "tests/e2e" }
  ]
}
.nvmrc
text

20.15.0
biome.json
JSON

{
  "$schema": "https://biomejs.dev/schemas/1.8.3/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noExcessiveCognitiveComplexity": "warn",
        "noForEach": "off"
      },
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error",
        "useExhaustiveDependencies": "warn"
      },
      "style": {
        "noNonNullAssertion": "warn",
        "useConst": "error",
        "useTemplate": "error",
        "noParameterAssign": "error",
        "useNamingConvention": {
          "level": "warn",
          "options": {
            "strictCase": false,
            "conventions": [
              {
                "selector": { "kind": "typeLike" },
                "formats": ["PascalCase"]
              },
              {
                "selector": { "kind": "variable" },
                "formats": ["camelCase", "CONSTANT_CASE", "PascalCase"]
              },
              {
                "selector": { "kind": "function" },
                "formats": ["camelCase", "PascalCase"]
              }
            ]
          }
        }
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsoleLog": "warn",
        "noArrayIndexKey": "warn"
      },
      "performance": {
        "noDelete": "warn"
      },
      "security": {
        "noGlobalEval": "error"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf",
    "attributePosition": "auto"
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always",
      "bracketSpacing": true,
      "bracketSameLine": false
    }
  },
  "json": {
    "formatter": {
      "trailingCommas": "none"
    }
  },
  "files": {
    "include": [
      "packages/*/src/**",
      "apps/*/src/**",
      "tests/**",
      "scripts/**",
      "*.json",
      "*.mjs",
      "*.cjs"
    ],
    "ignore": [
      "node_modules",
      "dist",
      ".next",
      "out",
      "coverage",
      ".turbo",
      "*.min.js",
      "pnpm-lock.yaml"
    ]
  }
}
.prettierrc
JSON

{
  "semi": true,
  "singleQuote": true,
  "jsxSingleQuote": false,
  "trailingComma": "all",
  "tabWidth": 2,
  "useTabs": false,
  "printWidth": 100,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "proseWrap": "preserve",
  "overrides": [
    {
      "files": ["*.md", "*.mdx"],
      "options": {
        "proseWrap": "always",
        "printWidth": 80
      }
    },
    {
      "files": ["*.json", "*.jsonc"],
      "options": {
        "trailingComma": "none"
      }
    },
    {
      "files": ["*.tsx", "*.jsx"],
      "options": {
        "jsxSingleQuote": false
      }
    }
  ]
}
.gitignore
gitignore

# ─────────────────────────────────────────────────────────
# LocoWorker monorepo .gitignore
# Pass 16 Part 1 — complete final version
# ─────────────────────────────────────────────────────────

# ── Node / pnpm ──────────────────────────────────────────
node_modules/
.pnp
.pnp.js
.pnp.cjs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# ── Build outputs ────────────────────────────────────────
dist/
build/
out/
.next/
.nuxt/
.output/
*.tsbuildinfo

# ── TurboRepo ────────────────────────────────────────────
.turbo/

# ── TypeScript ───────────────────────────────────────────
*.js.map
*.d.ts.map

# ── Test + coverage ──────────────────────────────────────
coverage/
.nyc_output/
*.lcov
test-results/
playwright-report/

# ── Environment / secrets ────────────────────────────────
.env
.env.local
.env.*.local
.env.development
.env.production
.env.test
!.env.example

# ── LocoWorker runtime artifacts ─────────────────────────
.locoworker/
locoworker.db
locoworker-memory.db
locoworker-graphify.db
locoworker-wiki.db
locoworker-kairos.db
locoworker-security.db
locoworker-gateway.db
*.locoworker.db
*.locoworker.db-shm
*.locoworker.db-wal

# ── SQLite ───────────────────────────────────────────────
*.db
*.db-shm
*.db-wal
*.sqlite
*.sqlite3

# ── Electron ─────────────────────────────────────────────
apps/desktop/release/
apps/desktop/.electron-vendors.cache.json

# ── Desktop / Dashboard Vite ─────────────────────────────
apps/desktop/dist/
apps/dashboard/dist/
apps/dashboard/.vite/

# ── Logs ─────────────────────────────────────────────────
logs/
*.log
npm-debug.log*
lerna-debug.log*

# ── OS ───────────────────────────────────────────────────
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
desktop.ini

# ── IDEs ─────────────────────────────────────────────────
.vscode/*
!.vscode/extensions.json
!.vscode/settings.json
!.vscode/launch.json
.idea/
*.swp
*.swo
*.suo
*.ntvs*
*.njsproj
*.sln

# ── Changesets ───────────────────────────────────────────
.changeset/*.md
!.changeset/config.json
!.changeset/README.md

# ── Misc ─────────────────────────────────────────────────
*.tgz
*.tar.gz
.cache/
tmp/
temp/
.env.example
dotenv

# ─────────────────────────────────────────────────────────
# LocoWorker .env.example
# Pass 16 Part 1 — complete final version
# Copy this file to .env and fill in values.
# NEVER commit .env to version control.
# ─────────────────────────────────────────────────────────

# ── Node environment ─────────────────────────────────────
NODE_ENV=development
LOG_LEVEL=info

# ── Workspace ────────────────────────────────────────────
# Absolute path to the workspace root LocoWorker operates within.
# Agent tools (fs, bash, git) are hard-constrained to this boundary.
LOCOWORKER_WORKSPACE_ROOT=/path/to/your/project

# ─────────────────────────────────────────────────────────
# PROVIDERS — Cloud (BYOK)
# You only need the keys for providers you intend to use.
# ─────────────────────────────────────────────────────────
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GEMINI_API_KEY=
MISTRAL_API_KEY=
GROQ_API_KEY=
COHERE_API_KEY=

# ─────────────────────────────────────────────────────────
# PROVIDERS — Local (Ollama / LM Studio / llama.cpp)
# Leave blank to disable local provider routing.
# ─────────────────────────────────────────────────────────
OLLAMA_BASE_URL=http://localhost:11434
LMSTUDIO_BASE_URL=http://localhost:1234
LLAMACPP_BASE_URL=http://localhost:8080

# ─────────────────────────────────────────────────────────
# PROVIDER ROUTING
# ─────────────────────────────────────────────────────────

# Which provider to use by default if none specified in session
# Options: anthropic | openai | gemini | mistral | groq | cohere | ollama | lmstudio | llamacpp
DEFAULT_PROVIDER=anthropic

# Default model within the chosen provider
DEFAULT_MODEL=claude-opus-4-5

# Fallback provider if default is unavailable or rate-limited
FALLBACK_PROVIDER=openai
FALLBACK_MODEL=gpt-4o

# Provider routing strategy: single | round-robin | least-cost | least-latency
PROVIDER_ROUTING_STRATEGY=single

# ─────────────────────────────────────────────────────────
# COST + BUDGET CONTROLS
# ─────────────────────────────────────────────────────────

# Hard cost cap in USD per session (agent stops at this amount)
LOCOWORKER_COST_CAP_USD=5.00

# Hard cost cap in USD per day across all sessions
LOCOWORKER_DAILY_COST_CAP_USD=50.00

# Warn user when a session reaches this fraction of the cap (0.0–1.0)
LOCOWORKER_COST_WARN_THRESHOLD=0.80

# ─────────────────────────────────────────────────────────
# PERMISSIONS
# ─────────────────────────────────────────────────────────

# Default permission tier for new sessions
# Options: read_only | read_write | full | admin
DEFAULT_PERMISSION_TIER=read_write

# Whether to require explicit confirmation for destructive file operations
REQUIRE_CONFIRMATION_FOR_DESTRUCTIVE=true

# Whether to require explicit confirmation for shell execution
REQUIRE_CONFIRMATION_FOR_BASH=true

# Comma-separated additional allowed paths outside workspace root (use sparingly)
ALLOWED_EXTRA_PATHS=

# Comma-separated glob patterns to deny access to within workspace
DENIED_PATH_PATTERNS=**/.env,**/.env.*,**/secrets/**,**/.git/config

# ─────────────────────────────────────────────────────────
# MEMORY PACKAGE
# ─────────────────────────────────────────────────────────
MEMORY_SQLITE_PATH=.locoworker/memory.db
MEMORY_MAX_SESSIONS=100
MEMORY_SESSION_TTL_DAYS=90

# AutoDream consolidation — runs nightly if Kairos is enabled
AUTODREAM_ENABLED=true
AUTODREAM_CRON=0 3 * * *

# ─────────────────────────────────────────────────────────
# GRAPHIFY PACKAGE
# ─────────────────────────────────────────────────────────
GRAPHIFY_SQLITE_PATH=.locoworker/graphify.db
GRAPHIFY_ENABLED=true
# Comma-separated globs to include in graph scanning
GRAPHIFY_INCLUDE=**/*.ts,**/*.tsx,**/*.js,**/*.jsx,**/*.py,**/*.go,**/*.rs
# Comma-separated globs to exclude
GRAPHIFY_EXCLUDE=**/node_modules/**,**/dist/**,**/build/**,**/.turbo/**

# ─────────────────────────────────────────────────────────
# WIKI PACKAGE
# ─────────────────────────────────────────────────────────
WIKI_SQLITE_PATH=.locoworker/wiki.db
WIKI_ROOT_PATH=.locoworker/wiki
WIKI_WATCH_ENABLED=true

# ─────────────────────────────────────────────────────────
# KAIROS (temporal / background autonomy)
# ─────────────────────────────────────────────────────────
ENABLE_KAIROS=false
KAIROS_SQLITE_PATH=.locoworker/kairos.db
# Kairos tick interval in milliseconds
KAIROS_TICK_INTERVAL_MS=60000
# Max concurrent background tasks
KAIROS_MAX_CONCURRENT_TASKS=2

# ─────────────────────────────────────────────────────────
# ORCHESTRATOR (multi-agent)
# ─────────────────────────────────────────────────────────
ENABLE_ORCHESTRATOR=false
ORCHESTRATOR_MAX_AGENTS=4
ORCHESTRATOR_SQLITE_PATH=.locoworker/orchestrator.db

# ─────────────────────────────────────────────────────────
# SECURITY PACKAGE
# ─────────────────────────────────────────────────────────
SECURITY_SQLITE_PATH=.locoworker/security.db
SECURITY_AUDIT_LOG_ENABLED=true
SECURITY_SECRET_DETECTION_ENABLED=true
SECURITY_INJECTION_DETECTION_ENABLED=true
# Rate limit: max tool calls per minute per session
SECURITY_TOOL_RATE_LIMIT_PER_MINUTE=60

# ─────────────────────────────────────────────────────────
# GATEWAY (HTTP / WebSocket API)
# ─────────────────────────────────────────────────────────
GATEWAY_PORT=3000
GATEWAY_HOST=127.0.0.1
GATEWAY_JWT_SECRET=replace-this-with-a-long-random-string-in-production
GATEWAY_JWT_EXPIRY_SECONDS=3600
# Comma-separated allowed CORS origins
GATEWAY_CORS_ORIGINS=http://localhost:5173,http://localhost:4173
# Max request body size
GATEWAY_MAX_BODY_SIZE=10mb
# Whether to enable admin routes (dangerous — disable in production)
GATEWAY_ENABLE_ADMIN_ROUTES=true

# ─────────────────────────────────────────────────────────
# AUTORESEARCH
# ─────────────────────────────────────────────────────────
ENABLE_AUTORESEARCH=false
AUTORESEARCH_MAX_SOURCES=20
AUTORESEARCH_MAX_DEPTH=3
# Report output path (relative to workspace root)
AUTORESEARCH_REPORT_OUTPUT_DIR=.locoworker/research

# ─────────────────────────────────────────────────────────
# MIROFISH (simulation studio)
# ─────────────────────────────────────────────────────────
ENABLE_MIROFISH=false
MIROFISH_DOCKER_ENABLED=false
MIROFISH_RESULTS_DIR=.locoworker/simulations

# ─────────────────────────────────────────────────────────
# DESKTOP APP (Electron)
# ─────────────────────────────────────────────────────────
# Port the embedded gateway listens on when launched from desktop
DESKTOP_EMBEDDED_GATEWAY_PORT=3001
# Whether the desktop app shows DevTools on startup
DESKTOP_DEVTOOLS=false
What Part 2 of Pass 16 will cover
Part 1 above gives you a clean, deterministic "root config layer." As soon as Part 1 is applied, pnpm install && pnpm turbo run build should work against the package set from Pass 1–15 with no manual merging needed.

Pass 16 Part 2 will complete the normalization layer with:

File	What it fixes
.github/workflows/ci.yml	Replaces both the Pass 1 stub and any Pass 14 partial; one authoritative CI workflow
.github/workflows/pr.yml	Complete PR gate: typecheck → lint → test (packages only, fast)
.github/workflows/release.yml	Complete changesets-based release workflow
scripts/env-check.mjs	Node script that validates .env has required keys before dev/build
scripts/prepare.mjs	postinstall hook (husky setup, workspace validation)
scripts/clean.mjs	Deterministic dist/ + .turbo/ cleanup across all workspaces
.changeset/config.json	Authoritative changesets config (correct package scopes)
.vscode/settings.json	Biome + Prettier integration settings so the IDE matches CI
.vscode/extensions.json	Recommended extensions list for the monorepo
FullScaffoldGeneration.md	Replaced with authoritative pass map (Pass 1–18 with correct descriptions)





Pass 16 — Part 2: CI/CD, Scripts, Tooling Config & Authoritative Pass Map
Rule: Every file below is a complete final version. When replaying passes in order, this pass's version fully replaces any earlier version written by Pass 1, Pass 12, or Pass 14. No merging required.

Applies after: Pass 16 Part 1 (root configs must already exist)

Result: After both Part 1 and Part 2 are applied, the entire root of the monorepo is normalized. pnpm install && pnpm turbo run build is deterministic with zero human merging.

File tree (Pass 16 Part 2 — files touched)
text

locoworker/
├── .changeset/
│   ├── config.json                     ← REPLACED (complete final)
│   └── README.md                       ← NEW
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md        ← NEW
│   └── workflows/
│       ├── ci.yml                      ← REPLACED (complete final)
│       ├── pr.yml                      ← REPLACED (complete final)
│       └── release.yml                 ← REPLACED (complete final)
├── .vscode/
│   ├── extensions.json                 ← REPLACED (complete final)
│   ├── launch.json                     ← NEW
│   └── settings.json                   ← REPLACED (complete final)
├── scripts/
│   ├── clean.mjs                       ← REPLACED (complete final)
│   ├── env-check.mjs                   ← REPLACED (complete final)
│   └── prepare.mjs                     ← REPLACED (complete final)
└── FullScaffoldGeneration.md           ← REPLACED (authoritative pass map, Pass 1–18)
.changeset/config.json
JSON

{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "privatePackages": {
    "version": true,
    "tag": false
  },
  "ignore": [
    "@locoworker/cli-app",
    "@locoworker/gateway-app",
    "@locoworker/desktop",
    "@locoworker/dashboard",
    "@locoworker/tests-integration",
    "@locoworker/tests-e2e"
  ]
}
.changeset/README.md
Markdown

# Changesets

This directory is managed by [Changesets](https://github.com/changesets/changesets).

## Workflow

1. When you make a change to a **publishable package** (`packages/*`), run:
   ```bash
   pnpm changeset
Follow the prompts to select which packages changed and the semver bump type.
Commit the generated .md file in .changeset/ alongside your code changes.
On merge to main, the release workflow will open a "Version Packages" PR that bumps all affected packages and updates CHANGELOG.md files.
Merging the "Version Packages" PR triggers publishing to the registry.
Publishable packages (managed by changesets)
Package	npm scope
packages/shared	@locoworker/shared
packages/core	@locoworker/core
packages/memory	@locoworker/memory
packages/graphify	@locoworker/graphify
packages/wiki	@locoworker/wiki
packages/kairos	@locoworker/kairos
packages/orchestrator	@locoworker/orchestrator
packages/security	@locoworker/security
packages/gateway	@locoworker/gateway
packages/autoresearch	@locoworker/autoresearch
packages/mirofish	@locoworker/mirofish
packages/tools-fs	@locoworker/tools-fs
packages/tools-bash	@locoworker/tools-bash
packages/tools-git	@locoworker/tools-git
packages/tools-search	@locoworker/tools-search
packages/tools-web	@locoworker/tools-web
Non-publishable (apps + tests — ignored by changesets)
apps/cli, apps/gateway, apps/desktop, apps/dashboard, tests/integration, tests/e2e

text


---

## `.github/PULL_REQUEST_TEMPLATE.md`

```markdown
## Summary

<!-- What does this PR do? 1-3 sentences. -->

## Type of change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Refactor (no functional changes, no API changes)
- [ ] Documentation update
- [ ] CI/tooling update
- [ ] Pass scaffold (new pass document added)

## Packages / apps affected

<!-- List packages and apps changed by this PR -->

## Checklist

- [ ] My code follows the project style guidelines (`pnpm lint`)
- [ ] I have run `pnpm typecheck` and it passes
- [ ] I have added or updated tests for new functionality
- [ ] All existing tests pass (`pnpm test`)
- [ ] I have added a changeset if this affects a publishable package (`pnpm changeset`)
- [ ] I have updated relevant documentation

## Testing

<!-- How was this tested? Include commands to reproduce. -->

## Notes for reviewers

<!-- Anything specific to call out for reviewers? -->
.github/workflows/ci.yml
YAML

# ─────────────────────────────────────────────────────────
# LocoWorker — Main CI workflow
# Pass 16 Part 2 — complete final version
# Triggers: push to main/develop, PRs to main/develop
# ─────────────────────────────────────────────────────────

name: CI

on:
  push:
    branches:
      - main
      - develop
      - "release/**"
  pull_request:
    branches:
      - main
      - develop

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20.15.0"
  PNPM_VERSION: "9.12.0"
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
  TURBO_REMOTE_ONLY: "false"

jobs:
  # ── Job 1: Install & cache ──────────────────────────────
  setup:
    name: Setup
    runs-on: ubuntu-latest
    outputs:
      pnpm-store-path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Get pnpm store path
        id: pnpm-cache
        shell: bash
        run: echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

  # ── Job 2: Typecheck ────────────────────────────────────
  typecheck:
    name: Typecheck
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Restore pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-store-path }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-typecheck-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            turbo-typecheck-${{ runner.os }}-

      - name: Typecheck
        run: pnpm typecheck

  # ── Job 3: Lint ─────────────────────────────────────────
  lint:
    name: Lint
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Restore pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-store-path }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm lint

      - name: Format check
        run: pnpm format:check

  # ── Job 4: Build ────────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [typecheck, lint]
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Restore pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-store-path }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-build-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            turbo-build-${{ runner.os }}-

      - name: Build all packages
        run: pnpm build:packages

      - name: Build apps
        run: pnpm build:apps

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-artifacts-${{ github.sha }}
          path: |
            packages/*/dist
            apps/cli/dist
            apps/gateway/dist
          retention-days: 3

  # ── Job 5: Unit tests ───────────────────────────────────
  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Restore pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-store-path }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifacts-${{ github.sha }}

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-test-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            turbo-test-${{ runner.os }}-

      - name: Run unit tests
        run: pnpm test:packages
        env:
          NODE_ENV: test

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-unit-${{ github.sha }}
          path: packages/*/coverage
          retention-days: 7

  # ── Job 6: Integration tests ────────────────────────────
  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Restore pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ needs.setup.outputs.pnpm-store-path }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifacts-${{ github.sha }}

      - name: Run integration tests
        run: pnpm test:integration
        env:
          NODE_ENV: test
          LOCOWORKER_WORKSPACE_ROOT: ${{ github.workspace }}/tmp/test-workspace
          DEFAULT_PROVIDER: ollama
          OLLAMA_BASE_URL: ""

      - name: Upload integration coverage
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-integration-${{ github.sha }}
          path: tests/integration/coverage
          retention-days: 7

  # ── Job 7: CI gate (all must pass) ─────────────────────
  ci-gate:
    name: CI Gate
    runs-on: ubuntu-latest
    needs: [typecheck, lint, build, test-unit, test-integration]
    if: always()
    steps:
      - name: Check all jobs passed
        run: |
          if [[ "${{ needs.typecheck.result }}" != "success" ]] || \
             [[ "${{ needs.lint.result }}" != "success" ]] || \
             [[ "${{ needs.build.result }}" != "success" ]] || \
             [[ "${{ needs.test-unit.result }}" != "success" ]] || \
             [[ "${{ needs.test-integration.result }}" != "success" ]]; then
            echo "One or more required CI jobs failed."
            exit 1
          fi
          echo "All CI jobs passed."
.github/workflows/pr.yml
YAML

# ─────────────────────────────────────────────────────────
# LocoWorker — Pull Request fast gate
# Pass 16 Part 2 — complete final version
# Purpose: fast feedback on PRs (typecheck + lint only,
#          no full build/test — that runs in ci.yml).
# ─────────────────────────────────────────────────────────

name: PR Gate

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches:
      - main
      - develop

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20.15.0"
  PNPM_VERSION: "9.12.0"

jobs:
  # ── Validate PR metadata ────────────────────────────────
  pr-meta:
    name: PR Metadata
    runs-on: ubuntu-latest
    steps:
      - name: Check PR title format
        uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            perf
            test
            build
            ci
            chore
            revert
            pass
          requireScope: false
          subjectPattern: ^.{1,100}$
          subjectPatternError: |
            PR title must be 1–100 characters after the type prefix.

  # ── Fast typecheck + lint ───────────────────────────────
  fast-check:
    name: Typecheck + Lint
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Get pnpm store path
        id: pnpm-cache
        shell: bash
        run: echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-pr-${{ runner.os }}-${{ github.event.pull_request.head.sha }}
          restore-keys: |
            turbo-pr-${{ runner.os }}-

      - name: Typecheck (affected packages only)
        run: pnpm turbo run typecheck --filter="...[HEAD~1]"

      - name: Lint (affected packages only)
        run: pnpm turbo run lint --filter="...[HEAD~1]"

      - name: Format check
        run: pnpm format:check

  # ── Changeset check ─────────────────────────────────────
  changeset-check:
    name: Changeset Check
    runs-on: ubuntu-latest
    if: |
      github.event.pull_request.draft == false &&
      !startsWith(github.event.pull_request.title, 'chore: version packages')
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Check for changeset
        uses: changesets/action@v1
        with:
          version: echo "Skipping version bump on PR"
          publish: echo "Skipping publish on PR"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        continue-on-error: true
.github/workflows/release.yml
YAML

# ─────────────────────────────────────────────────────────
# LocoWorker — Release workflow
# Pass 16 Part 2 — complete final version
# Trigger: push to main with changeset files present
# ─────────────────────────────────────────────────────────

name: Release

on:
  push:
    branches:
      - main

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

env:
  NODE_VERSION: "20.15.0"
  PNPM_VERSION: "9.12.0"
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ secrets.TURBO_TEAM }}

jobs:
  # ── Release or open "Version Packages" PR ──────────────
  release:
    name: Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"
          registry-url: "https://registry.npmjs.org"

      - name: Get pnpm store path
        id: pnpm-cache
        shell: bash
        run: echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_OUTPUT

      - name: Cache pnpm store
        uses: actions/cache@v4
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: pnpm-store-${{ runner.os }}-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            pnpm-store-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Cache turbo
        uses: actions/cache@v4
        with:
          path: .turbo
          key: turbo-release-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            turbo-release-${{ runner.os }}-

      - name: Build all packages
        run: pnpm build:packages

      - name: Run tests
        run: pnpm test:packages
        env:
          NODE_ENV: test

      # Changesets action:
      # - If there are changeset files → opens / updates "Version Packages" PR.
      # - If there are no changeset files and the commit is "chore: version packages"
      #   → publishes to npm.
      - name: Create release PR or publish
        uses: changesets/action@v1
        with:
          version: pnpm changeset:version
          publish: pnpm changeset:publish
          title: "chore: version packages"
          commit: "chore: version packages"
          createGithubReleases: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

  # ── Tag & GitHub Release (after publish) ───────────────
  github-release:
    name: GitHub Release
    runs-on: ubuntu-latest
    needs: release
    if: needs.release.result == 'success'
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate release notes
        uses: actions/github-script@v7
        with:
          script: |
            const { data: latestRelease } = await github.rest.repos.getLatestRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
            }).catch(() => ({ data: null }));

            const tag = process.env.GITHUB_REF_NAME;
            console.log(`Latest release: ${latestRelease?.tag_name ?? 'none'}`);
            console.log(`Current ref: ${tag}`);
scripts/env-check.mjs
JavaScript

#!/usr/bin/env node
// ─────────────────────────────────────────────────────────
// env-check.mjs
// Pass 16 Part 2 — complete final version
//
// Validates that required env vars are set before dev/build.
// Run automatically via: pnpm env:check
// Also called by prepare.mjs and the CLI bootstrap.
// ─────────────────────────────────────────────────────────

import { existsSync, readFileSync } from 'node:fs';
import { resolve, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));
const ROOT = resolve(__dirname, '..');

// ── Colour helpers (no deps) ─────────────────────────────
const c = {
  reset: '\x1b[0m',
  bold: '\x1b[1m',
  red: '\x1b[31m',
  yellow: '\x1b[33m',
  green: '\x1b[32m',
  cyan: '\x1b[36m',
  grey: '\x1b[90m',
};
const fmt = (color, msg) => `${color}${msg}${c.reset}`;

// ── Load .env if present (manual parse — no dotenv dep) ──
const envPath = resolve(ROOT, '.env');
if (existsSync(envPath)) {
  const lines = readFileSync(envPath, 'utf8').split('\n');
  for (const line of lines) {
    const trimmed = line.trim();
    if (!trimmed || trimmed.startsWith('#')) continue;
    const eqIdx = trimmed.indexOf('=');
    if (eqIdx === -1) continue;
    const key = trimmed.slice(0, eqIdx).trim();
    const value = trimmed.slice(eqIdx + 1).trim();
    if (key && !process.env[key]) {
      process.env[key] = value;
    }
  }
}

// ─────────────────────────────────────────────────────────
// Rule definitions
// Each rule has:
//   key      — env var name
//   level    — 'error' | 'warn' | 'info'
//   message  — shown if the rule fails
//   check    — optional custom validator (returns string | null)
// ─────────────────────────────────────────────────────────
const rules = [
  // ── Workspace ─────────────────────────────────────────
  {
    key: 'LOCOWORKER_WORKSPACE_ROOT',
    level: 'warn',
    message: 'Agent tools will default to process.cwd() as the workspace root.',
  },

  // ── Provider: at least one key or a local endpoint ────
  {
    key: '__PROVIDER_CHECK__',
    level: 'warn',
    message:
      'No provider configured. Set at least one of: ANTHROPIC_API_KEY, OPENAI_API_KEY, ' +
      'GEMINI_API_KEY, MISTRAL_API_KEY, GROQ_API_KEY, OLLAMA_BASE_URL, LMSTUDIO_BASE_URL, ' +
      'LLAMACPP_BASE_URL.',
    check: () => {
      const providerEnvs = [
        'ANTHROPIC_API_KEY',
        'OPENAI_API_KEY',
        'GEMINI_API_KEY',
        'MISTRAL_API_KEY',
        'GROQ_API_KEY',
        'COHERE_API_KEY',
        'OLLAMA_BASE_URL',
        'LMSTUDIO_BASE_URL',
        'LLAMACPP_BASE_URL',
      ];
      const hasProvider = providerEnvs.some((k) => process.env[k]?.trim());
      return hasProvider ? null : 'No provider key or local endpoint set.';
    },
  },

  // ── Gateway JWT secret ─────────────────────────────────
  {
    key: 'GATEWAY_JWT_SECRET',
    level: 'warn',
    message: 'Using default JWT secret. Set a strong random value for production.',
    check: (val) => {
      if (!val || val === 'replace-this-with-a-long-random-string-in-production') {
        return 'GATEWAY_JWT_SECRET is not set or is still the example placeholder.';
      }
      if (val.length < 32) {
        return `GATEWAY_JWT_SECRET is too short (${val.length} chars). Use at least 32 chars.`;
      }
      return null;
    },
  },

  // ── Node version ──────────────────────────────────────
  {
    key: '__NODE_VERSION__',
    level: 'error',
    message: 'LocoWorker requires Node.js >= 20.0.0.',
    check: () => {
      const [major] = process.versions.node.split('.').map(Number);
      return major >= 20 ? null : `Node.js ${process.versions.node} is below minimum (20.0.0).`;
    },
  },

  // ── SQLite paths: warn if they point to root ──────────
  {
    key: '__SQLITE_PATHS__',
    level: 'info',
    message: 'SQLite databases will be created in .locoworker/ (default).',
    check: () => {
      const dbKeys = [
        'MEMORY_SQLITE_PATH',
        'GRAPHIFY_SQLITE_PATH',
        'WIKI_SQLITE_PATH',
        'KAIROS_SQLITE_PATH',
        'SECURITY_SQLITE_PATH',
      ];
      const rootPaths = dbKeys.filter((k) => {
        const v = process.env[k];
        return v && !v.startsWith('.locoworker/') && !v.startsWith('/');
      });
      return rootPaths.length > 0
        ? `These SQLite paths look unusual: ${rootPaths.join(', ')}`
        : null;
    },
  },
];

// ─────────────────────────────────────────────────────────
// Run checks
// ─────────────────────────────────────────────────────────
let errorCount = 0;
let warnCount = 0;

console.log(fmt(c.bold + c.cyan, '\n🔍 LocoWorker env-check\n'));

for (const rule of rules) {
  let message = null;

  if (rule.check) {
    const val = rule.key.startsWith('__') ? undefined : process.env[rule.key];
    message = rule.check(val);
  } else {
    const val = process.env[rule.key];
    if (!val || !val.trim()) {
      message = rule.message;
    }
  }

  if (!message) {
    if (!rule.key.startsWith('__')) {
      console.log(`  ${fmt(c.green, '✓')} ${fmt(c.grey, rule.key)}`);
    }
    continue;
  }

  if (rule.level === 'error') {
    console.log(`  ${fmt(c.red, '✗ ERROR')} ${rule.key}: ${message}`);
    errorCount++;
  } else if (rule.level === 'warn') {
    console.log(`  ${fmt(c.yellow, '⚠ WARN')}  ${rule.key}: ${message}`);
    warnCount++;
  } else {
    console.log(`  ${fmt(c.cyan, 'ℹ INFO')}  ${rule.key}: ${message}`);
  }
}

console.log('');

if (errorCount > 0) {
  console.log(fmt(c.red + c.bold, `✗ env-check failed: ${errorCount} error(s), ${warnCount} warning(s).\n`));
  process.exit(1);
} else if (warnCount > 0) {
  console.log(fmt(c.yellow, `⚠ env-check passed with ${warnCount} warning(s). Check .env.example for guidance.\n`));
  process.exit(0);
} else {
  console.log(fmt(c.green + c.bold, '✓ env-check passed.\n'));
  process.exit(0);
}
scripts/prepare.mjs
JavaScript

#!/usr/bin/env node
// ─────────────────────────────────────────────────────────
// prepare.mjs
// Pass 16 Part 2 — complete final version
//
// Runs after `pnpm install` (postinstall / prepare hook).
// Responsibilities:
//   1. Skip in CI (just installs, doesn't need prepare hooks).
//   2. Create .locoworker/ runtime dir if missing.
//   3. Copy .env.example → .env if .env doesn't exist yet.
//   4. Install husky hooks if .git exists.
// ─────────────────────────────────────────────────────────

import {
  existsSync,
  mkdirSync,
  copyFileSync,
  writeFileSync,
  chmodSync,
} from 'node:fs';
import { resolve, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import { execSync } from 'node:child_process';

const __dirname = dirname(fileURLToPath(import.meta.url));
const ROOT = resolve(__dirname, '..');

// ── Skip in CI ───────────────────────────────────────────
if (process.env.CI === 'true' || process.env.CI === '1') {
  console.log('prepare.mjs: CI detected — skipping postinstall hooks.');
  process.exit(0);
}

// ── Colour helpers ───────────────────────────────────────
const c = {
  reset: '\x1b[0m',
  bold: '\x1b[1m',
  green: '\x1b[32m',
  cyan: '\x1b[36m',
  yellow: '\x1b[33m',
  grey: '\x1b[90m',
};
const fmt = (color, msg) => `${color}${msg}${c.reset}`;
const log = (msg) => console.log(`  ${msg}`);

console.log(fmt(c.bold + c.cyan, '\n⚙  LocoWorker prepare\n'));

// ── 1. Create .locoworker/ runtime directory ─────────────
const locoworkerDir = resolve(ROOT, '.locoworker');
if (!existsSync(locoworkerDir)) {
  mkdirSync(locoworkerDir, { recursive: true });
  log(fmt(c.green, '✓') + fmt(c.grey, ' Created .locoworker/'));
} else {
  log(fmt(c.grey, '· .locoworker/ already exists'));
}

// Sub-directories used by packages
const runtimeDirs = [
  '.locoworker/wiki',
  '.locoworker/research',
  '.locoworker/simulations',
  '.locoworker/logs',
];
for (const dir of runtimeDirs) {
  const full = resolve(ROOT, dir);
  if (!existsSync(full)) {
    mkdirSync(full, { recursive: true });
    log(fmt(c.green, '✓') + fmt(c.grey, ` Created ${dir}/`));
  }
}

// ── 2. Copy .env.example → .env if needed ───────────────
const envExample = resolve(ROOT, '.env.example');
const envFile = resolve(ROOT, '.env');

if (!existsSync(envFile) && existsSync(envExample)) {
  copyFileSync(envExample, envFile);
  log(fmt(c.green, '✓') + ' Created .env from .env.example');
  log(fmt(c.yellow, '  ⚠  Edit .env and set at least one provider key before running.'));
} else if (existsSync(envFile)) {
  log(fmt(c.grey, '· .env already exists — not overwritten.'));
} else {
  log(fmt(c.yellow, '⚠  .env.example not found — skipping .env creation.'));
}

// ── 3. Create MEMORY.md + CLAUDE.md if missing ──────────
const claudeMd = resolve(ROOT, 'CLAUDE.md');
if (!existsSync(claudeMd)) {
  writeFileSync(
    claudeMd,
    [
      '# CLAUDE.md — Agent Instructions',
      '',
      '> This file is injected into the agent context on every session.',
      '> It acts as standing instructions for the AI.',
      '> Edit this file to customise agent behaviour for your project.',
      '',
      '## Project',
      '',
      '<!-- Describe the project here -->',
      '',
      '## Conventions',
      '',
      '<!-- Coding style, commit conventions, anything the agent should follow -->',
      '',
      '## Off-limits',
      '',
      '<!-- Any files, directories or actions the agent should never touch -->',
      '',
    ].join('\n'),
    'utf8',
  );
  log(fmt(c.green, '✓') + fmt(c.grey, ' Created CLAUDE.md'));
}

const memoryMd = resolve(ROOT, 'MEMORY.md');
if (!existsSync(memoryMd)) {
  writeFileSync(
    memoryMd,
    [
      '# MEMORY.md — Agent Memory Index',
      '',
      '> Managed by the LocoWorker memory package.',
      '> Do not edit manually — this file is updated by AutoDream.',
      '',
      '## Sessions',
      '',
      '<!-- Session summaries will be appended here by AutoDream -->',
      '',
    ].join('\n'),
    'utf8',
  );
  log(fmt(c.green, '✓') + fmt(c.grey, ' Created MEMORY.md'));
}

// ── 4. Install husky if .git exists ─────────────────────
const gitDir = resolve(ROOT, '.git');
if (existsSync(gitDir)) {
  try {
    // Only install if husky is available
    execSync('node node_modules/.bin/husky install', {
      cwd: ROOT,
      stdio: 'pipe',
    });
    log(fmt(c.green, '✓') + fmt(c.grey, ' Husky hooks installed.'));

    // Create pre-commit hook: lint-staged
    const huskyDir = resolve(ROOT, '.husky');
    if (!existsSync(huskyDir)) mkdirSync(huskyDir, { recursive: true });
    const preCommit = resolve(huskyDir, 'pre-commit');
    if (!existsSync(preCommit)) {
      writeFileSync(
        preCommit,
        ['#!/usr/bin/env sh', '. "$(dirname -- "$0")/_/husky.sh"', '', 'pnpm lint-staged', ''].join('\n'),
        'utf8',
      );
      chmodSync(preCommit, 0o755);
      log(fmt(c.green, '✓') + fmt(c.grey, ' Created .husky/pre-commit'));
    }
  } catch {
    log(fmt(c.grey, '· Husky not available — skipping git hooks.'));
  }
} else {
  log(fmt(c.grey, '· No .git directory — skipping husky install.'));
}

console.log(fmt(c.green + c.bold, '\n✓ prepare complete.\n'));
scripts/clean.mjs
JavaScript

#!/usr/bin/env node
// ─────────────────────────────────────────────────────────
// clean.mjs
// Pass 16 Part 2 — complete final version
//
// Deterministic dist/ + .turbo/ cleanup across all workspaces.
// Usage:
//   node scripts/clean.mjs           # dist + .turbo only
//   node scripts/clean.mjs --full    # also node_modules
// ─────────────────────────────────────────────────────────

import { rmSync, existsSync, readdirSync, statSync } from 'node:fs';
import { resolve, join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));
const ROOT = resolve(__dirname, '..');
const FULL = process.argv.includes('--full');

// ── Colour helpers ───────────────────────────────────────
const c = {
  reset: '\x1b[0m',
  bold: '\x1b[1m',
  red: '\x1b[31m',
  green: '\x1b[32m',
  cyan: '\x1b[36m',
  grey: '\x1b[90m',
};
const fmt = (color, msg) => `${color}${msg}${c.reset}`;

console.log(fmt(c.bold + c.cyan, `\n🧹 LocoWorker clean${FULL ? ' --full' : ''}\n`));

// ── Targets per workspace ─────────────────────────────────
const BASE_TARGETS = ['dist', '.turbo', 'coverage', '*.tsbuildinfo', 'out', '.next'];
const FULL_TARGETS = [...BASE_TARGETS, 'node_modules'];

// ── Workspaces to clean ───────────────────────────────────
const workspaces = [
  ROOT,
  ...getSubDirs(join(ROOT, 'packages')),
  ...getSubDirs(join(ROOT, 'apps')),
  ...getSubDirs(join(ROOT, 'tests')),
];

function getSubDirs(dir) {
  if (!existsSync(dir)) return [];
  return readdirSync(dir)
    .map((name) => join(dir, name))
    .filter((p) => statSync(p).isDirectory());
}

// ── Clean each workspace ──────────────────────────────────
let removed = 0;
const targets = FULL ? FULL_TARGETS : BASE_TARGETS;

for (const ws of workspaces) {
  for (const target of targets) {
    // Handle glob patterns like *.tsbuildinfo
    if (target.startsWith('*')) {
      const ext = target.slice(1);
      if (!existsSync(ws)) continue;
      const matches = readdirSync(ws).filter((f) => f.endsWith(ext));
      for (const match of matches) {
        const full = join(ws, match);
        rmSync(full, { force: true });
        console.log(`  ${fmt(c.red, '-')} ${fmt(c.grey, full.replace(ROOT, '.'))}`);
        removed++;
      }
    } else {
      const full = join(ws, target);
      if (existsSync(full)) {
        rmSync(full, { recursive: true, force: true });
        console.log(`  ${fmt(c.red, '-')} ${fmt(c.grey, full.replace(ROOT, '.'))}`);
        removed++;
      }
    }
  }
}

// ── Root-level .turbo ─────────────────────────────────────
const rootTurbo = resolve(ROOT, '.turbo');
if (existsSync(rootTurbo)) {
  rmSync(rootTurbo, { recursive: true, force: true });
  console.log(`  ${fmt(c.red, '-')} ${fmt(c.grey, '.turbo/')}`);
  removed++;
}

console.log(
  removed > 0
    ? fmt(c.green + c.bold, `\n✓ Removed ${removed} item(s).\n`)
    : fmt(c.grey, '\n· Nothing to clean.\n'),
);
.vscode/settings.json
JSON

{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.formatOnPaste": false,
  "editor.codeActionsOnSave": {
    "source.fixAll.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },

  "[typescript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[javascriptreact]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[json]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[jsonc]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[markdown]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.wordWrap": "on"
  },
  "[yaml]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },

  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "typescript.preferences.importModuleSpecifier": "non-relative",
  "typescript.preferences.quoteStyle": "single",

  "eslint.enable": false,

  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.turbo": true,
    "**/coverage": true,
    "**/.next": true,
    "**/out": true,
    "**/.locoworker/*.db": true,
    "**/.locoworker/*.db-shm": true,
    "**/.locoworker/*.db-wal": true
  },

  "files.exclude": {
    "**/.turbo": true,
    "**/node_modules": false,
    "**/dist": false
  },

  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/dist/**": true,
    "**/.turbo/**": true,
    "**/coverage/**": true,
    "**/.locoworker/**": true
  },

  "explorer.fileNesting.enabled": true,
  "explorer.fileNesting.patterns": {
    "*.ts": "${capture}.test.ts, ${capture}.spec.ts, ${capture}.d.ts",
    "*.tsx": "${capture}.test.tsx, ${capture}.spec.tsx",
    "package.json": "pnpm-lock.yaml, .npmrc, .nvmrc, tsconfig*.json",
    "turbo.json": ".turbo",
    ".env.example": ".env, .env.*"
  },

  "turboConsole.prefix": "[LocoWorker]",

  "vitest.enable": true,
  "vitest.commandLine": "pnpm vitest"
}
.vscode/extensions.json
JSON

{
  "recommendations": [
    "biomejs.biome",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "vitest.explorer",
    "GitHub.vscode-github-actions",
    "ms-azuretools.vscode-docker",
    "prisma.prisma",
    "bradlc.vscode-tailwindcss",
    "styled-components.vscode-styled-components",
    "PKief.material-icon-theme",
    "usernamehw.errorlens",
    "streetsidesoftware.code-spell-checker",
    "christian-kohler.path-intellisense",
    "wayou.vscode-todo-highlight",
    "gruntfuggly.todo-tree",
    "eamodio.gitlens",
    "mhutchie.git-graph"
  ],
  "unwantedRecommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode-next"
  ]
}
.vscode/launch.json
JSON

{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug: CLI",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/apps/cli/dist/index.js",
      "preLaunchTask": "build:cli",
      "cwd": "${workspaceFolder}",
      "env": {
        "NODE_ENV": "development",
        "LOG_LEVEL": "debug"
      },
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug: Gateway",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/apps/gateway/dist/index.js",
      "preLaunchTask": "build:gateway",
      "cwd": "${workspaceFolder}",
      "env": {
        "NODE_ENV": "development",
        "LOG_LEVEL": "debug",
        "GATEWAY_PORT": "3000"
      },
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug: Desktop",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "program": "${workspaceFolder}/apps/desktop/dist/main/index.js",
      "preLaunchTask": "build:desktop",
      "cwd": "${workspaceFolder}/apps/desktop",
      "env": {
        "NODE_ENV": "development",
        "ELECTRON_ENABLE_LOGGING": "true",
        "DESKTOP_DEVTOOLS": "true"
      },
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug: Vitest (current file)",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/.bin/vitest",
      "args": ["run", "${relativeFile}"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug: Vitest (all)",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/node_modules/.bin/vitest",
      "args": ["run"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
FullScaffoldGeneration.md ← REPLACED (authoritative pass map)
Markdown

# LocoWorker — Authoritative Pass Map

> **Pass 16 Part 2 — this document is the canonical pass map.**
> It replaces all earlier versions of `FullScaffoldGeneration.md`.
> Rule: later pass wins. When replaying from markdown, apply passes
> in ascending order. If a later pass writes a path that already
> exists, it fully replaces the earlier content.

---

## Replay order
Pass 1 → Pass 2 → Pass 3 → … → Pass 15 → Pass 16 (P1+P2) → Pass 17 → Pass 18

text


---

## Pass reference table

| Pass | Parts | What it creates | Status |
|------|-------|-----------------|--------|
| **1** | — | Monorepo spine: pnpm workspaces, TurboRepo, TS base config, root `.env.example`, `.gitignore`, CI stubs, `CLAUDE.md`, `MEMORY.md` | ✅ Done |
| **2** | — | `packages/core`: agent loop (`queryLoop`), `ToolRegistry`, `PermissionGate`, `ProviderRouter`, `TurnAssembler`, `AdaptiveCompactor`, `SessionManager`, `BuddyEngine` | ✅ Done |
| **3** | — | `packages/memory`: `MemoryManager`, `ConversationStore`, session snapshots, `AutoDream` consolidation pipeline | ✅ Done |
| **4** | — | `packages/graphify`: tree-sitter AST parser, SQLite symbol graph, `LeidenClusterer`, incremental updater, `GraphifyClient` | ✅ Done |
| **5** | — | `packages/wiki`: `WikiStore` (SQLite/FTS5), `WikiParser`, `WikiSearch`, `WikiLinker`, `WikiSyncAgent`, `WikiWatcher` | ✅ Done |
| **6** | P1+P2 | `packages/kairos`: `KairosStore`, `TaskQueue`, `CronScheduler`, `TemporalContext`, `TickEngine`, `ObservationLog`, `KairosAgent` | ✅ Done |
| **7** | P1+P2 | `packages/orchestrator`: `OrchestratorStore`, `AgentRegistry`, `AgentSpawner`, `MessageRouter`, `DelegationPlanner`, `FileLockManager`, `ResultAggregator`, `OrchestratorEngine` | ✅ Done |
| **8** | — | `packages/gateway`: auth manager, token store, rate limiter, quota manager, WebSocket server, connection manager, event streamer, REST routes, SQLite store | ✅ Done |
| **9** | P1+P2 | `packages/autoresearch`: research planner, multi-source querier, fetcher/extractor, evidence graph, synthesizer, report builder, runner loop, Kairos schedule hook | ✅ Done |
| **10** | P1+P2 | `packages/mirofish`: persona factory, scenario builder, container orchestrator, simulation runner, incident detector, agent harness | ✅ Done |
| **11** | — | `packages/tools-fs`, `packages/tools-bash`, `packages/tools-git`, `packages/tools-search`, `packages/tools-web`; `packages/shared` (logger, Result helpers, token utils, env helpers, constants) | ✅ Done |
| **12** | — | `packages/security` (audit log, secret detector, sanitizer, rate limiter, policy scanner); `apps/cli` (bootstrap, REPL, commands); `apps/gateway` (Fastify server, JWT middleware, SSE streaming, REST routes) | ✅ Done |
| **13** | — | `apps/desktop` (Electron + Vite, IPC contracts, main/preload/renderer); `apps/dashboard` (React SPA, chat components, state stores) | ✅ Done |
| **14** | — | `tests/integration` (harness + suites); `tests/e2e` (gateway SSE, WebSocket, CLI subprocess); `.github/workflows` stubs; `scripts/` stubs; root script additions | ✅ Done |
| **15** | — | Docker build artifacts (`docker/Dockerfile.*`, `docker/docker-compose.yml`); `docs/ARCHITECTURE.md`, `docs/DEPLOYMENT.md`; final `README.md`, `CONTRIBUTING.md`; AutoResearch Part 2 completion; MiroFish Part 2 completion | ✅ Done |
| **16** | **P1** | Root config normalization: `pnpm-workspace.yaml`, `turbo.json`, root `package.json` (no stale refs), `tsconfig.base.json`, `tsconfig.json`, `.nvmrc`, `biome.json`, `.prettierrc`, `.gitignore`, `.env.example` | ✅ Done |
| **16** | **P2** | CI/CD normalization: `.github/workflows/ci.yml`, `pr.yml`, `release.yml`; `scripts/env-check.mjs`, `prepare.mjs`, `clean.mjs`; `.vscode/settings.json`, `extensions.json`, `launch.json`; `.changeset/config.json`; this file | ✅ Done |
| **17** | — | Spec-gap scaffolding: `packages/tools-mcp` (stub + MCP client interface + tool registration skeleton); `packages/providers` (provider registry abstraction + ProviderRouter wiring points); `openclaw/` placeholder (README + `package.json` + interface contracts) | 🔜 Next |
| **18** | — | Spec implementation (incremental): implement `tools-mcp` fully + integrate into `ToolRegistry`; implement `packages/providers` registry fully + update `ProviderRouter` to use it; ADR decisions on Tauri vs Electron + Ink CLI vs current `apps/cli` | 🔜 Planned |

---

## Package dependency graph (logical)
shared └── core ├── memory ├── graphify ├── wiki ├── kairos ├── orchestrator ├── security ├── gateway ├── autoresearch ├── mirofish ├── tools-fs ├── tools-bash ├── tools-git ├── tools-search └── tools-web └── apps/cli └── apps/gateway └── apps/desktop └── apps/dashboard └── tests/integration └── tests/e2e

text


---

## "Later pass wins" file conflict map

The table below documents every file written by more than one pass.
When replaying, the **highest pass number** is the authoritative version.

| File path | First written | Final version in |
|-----------|--------------|-----------------|
| `package.json` (root) | Pass 1 | **Pass 16 P1** |
| `pnpm-workspace.yaml` | Pass 1 | **Pass 16 P1** |
| `turbo.json` | Pass 1 | **Pass 16 P1** |
| `tsconfig.base.json` | Pass 1 | **Pass 16 P1** |
| `tsconfig.json` | Pass 1 | **Pass 16 P1** |
| `.env.example` | Pass 1 | **Pass 16 P1** |
| `.gitignore` | Pass 1 | **Pass 16 P1** |
| `biome.json` | Pass 1 | **Pass 16 P1** |
| `.prettierrc` | Pass 1 | **Pass 16 P1** |
| `.nvmrc` | Pass 1 | **Pass 16 P1** |
| `.github/workflows/ci.yml` | Pass 1 | **Pass 16 P2** |
| `.github/workflows/pr.yml` | Pass 14 | **Pass 16 P2** |
| `.github/workflows/release.yml` | Pass 1 | **Pass 16 P2** |
| `scripts/env-check.mjs` | Pass 14 | **Pass 16 P2** |
| `scripts/prepare.mjs` | Pass 14 | **Pass 16 P2** |
| `scripts/clean.mjs` | Pass 14 | **Pass 16 P2** |
| `.vscode/settings.json` | Pass 1 | **Pass 16 P2** |
| `.vscode/extensions.json` | Pass 1 | **Pass 16 P2** |
| `.changeset/config.json` | Pass 1 | **Pass 16 P2** |
| `FullScaffoldGeneration.md` | Pass 1 | **Pass 16 P2** |

---

## Spec alignment status (vs `Completeproject.md`)

| Spec component | Scaffold status | Target pass |
|----------------|----------------|-------------|
| `packages/core` (agent loop) | ✅ Pass 2 | — |
| `packages/memory` | ✅ Pass 3 | — |
| `packages/graphify` | ✅ Pass 4 | — |
| `packages/wiki` | ✅ Pass 5 | — |
| `packages/kairos` | ✅ Pass 6 | — |
| `packages/orchestrator` | ✅ Pass 7 | — |
| `packages/gateway` | ✅ Pass 8 | — |
| `packages/autoresearch` | ✅ Pass 9 | — |
| `packages/mirofish` | ✅ Pass 10 | — |
| `packages/tools-*` (fs/bash/git/search/web) | ✅ Pass 11 | — |
| `packages/shared` | ✅ Pass 11 | — |
| `packages/security` | ✅ Pass 12 | — |
| `apps/cli` | ✅ Pass 12 | — |
| `apps/gateway` | ✅ Pass 12 | — |
| `apps/desktop` (Electron) | ✅ Pass 13 | ADR in Pass 18 |
| `apps/dashboard` | ✅ Pass 13 | — |
| `tests/integration` + `tests/e2e` | ✅ Pass 14 | — |
| Docker + docs | ✅ Pass 15 | — |
| **`packages/tools-mcp`** | ❌ Missing | **Pass 17** |
| **`packages/providers`** | ❌ Missing | **Pass 17** |
| **`openclaw/`** | ❌ Missing | **Pass 17** |
| Tauri desktop (spec calls for Tauri) | ⚠️ Electron used | ADR in **Pass 18** |
| Ink CLI (`cowork-cli`) | ⚠️ Node CLI used | ADR in **Pass 18** |
What comes next
Pass 16 (both parts) is now complete. The monorepo root is fully normalized. Here is the state of play:

Layer	State after Pass 16 P1 + P2
Root configs	✅ Single source of truth, no stale refs
Workspace definition	✅ All 16 packages + 4 apps + 2 test suites declared
Turbo pipeline	✅ All tasks wired with correct deps and caching
CI/CD workflows	✅ Three complete, final workflows (ci / pr / release)
Scripts	✅ env-check, prepare, clean all complete
VSCode DX	✅ Biome + Prettier + Vitest + launch configs
Changesets	✅ Configured with correct publishable vs ignored scopes
Pass map	✅ Authoritative — conflict map + spec alignment table included
Pass 17	🔜 packages/tools-mcp + packages/providers + openclaw/ stubs
Pass 18	🔜 Full implementation of spec-gap packages + ADRs
