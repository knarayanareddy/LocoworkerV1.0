LocoWorker Pre-Build Checklist
Part 1 of 5 — Environment, Repository Structure & Root Configuration
Section 1 — System Prerequisites
These must be installed and verified before touching any project files.

1.1 Node.js
Bash

node --version
# Required: v20.x LTS or v22.x LTS
# Do NOT use v18 — some packages use fetch natively which requires v20+
# Do NOT use odd-numbered versions (v21, v23) — they are not LTS
If wrong version:

Bash

# Using nvm (recommended):
nvm install 20
nvm use 20
nvm alias default 20
1.2 pnpm
The entire monorepo uses pnpm workspaces. npm and yarn will not work.

Bash

pnpm --version
# Required: 8.x or 9.x
If not installed:

Bash

corepack enable
corepack prepare pnpm@9 --activate
Verify workspace support:

Bash

pnpm --version
# Should print 8.x.x or 9.x.x
1.3 Turborepo
Bash

pnpm add -g turbo
turbo --version
# Required: 1.13.x or 2.x
1.4 TypeScript (global, for diagnostics only)
Bash

pnpm add -g typescript
tsc --version
# Required: 5.4.x or higher
1.5 Git
Bash

git --version
# Any modern version is fine (2.x+)
1.6 Platform-specific system dependencies
macOS:

Bash

# Xcode Command Line Tools (required for native module compilation)
xcode-select --install

# Verify
xcode-select -p
# Should print: /Library/Developer/CommandLineTools

# Rosetta 2 (Apple Silicon only — needed for some x64 native modules)
softwareupdate --install-rosetta --agree-to-license
Linux (Ubuntu/Debian):

Bash

sudo apt-get update
sudo apt-get install -y \
  build-essential \
  libsecret-1-dev \
  libnss3-dev \
  libatk1.0-dev \
  libatk-bridge2.0-dev \
  libgdk-pixbuf2.0-dev \
  libgtk-3-dev \
  libgbm-dev \
  libasound2-dev \
  python3 \
  python3-pip

# libsecret-1-dev is critical — keytar (packages/keychain) will not compile without it
Linux (Fedora/RHEL/CentOS):

Bash

sudo dnf install -y \
  make \
  gcc \
  gcc-c++ \
  libsecret-devel \
  nss \
  atk \
  gtk3-devel \
  alsa-lib-devel \
  python3
Windows:

PowerShell

# Install Visual Studio Build Tools (required for native modules)
# Download from: https://visualstudio.microsoft.com/visual-cpp-build-tools/
# Select: "Desktop development with C++"

# Then install windows-build-tools
npm install -g windows-build-tools

# Verify
node -e "require('node-gyp')"
1.7 Electron dependencies (desktop app)
macOS: No extra steps — Xcode CLI tools cover it.

Linux:

Bash

# Already covered in 1.6 — double-check libgbm and libasound2 are present
dpkg -l libgbm-dev libasound2-dev libsecret-1-dev
Windows:

PowerShell

# Ensure Windows SDK is installed alongside VS Build Tools
# Check: C:\Program Files (x86)\Windows Kits\10 exists
Section 2 — Repository Structure Verification
Before running any pass, confirm the cloned repo has the exact structure the passes expect.

2.1 Clone and verify root
Bash

git clone https://github.com/knarayanareddy/LocoworkerV1.0.git locoworker
cd locoworker
ls -la
Expected root files:

text

.env.example
.eslintrc.js         (or eslint.config.js — check Pass 1)
.gitignore
.prettierrc
.prettierignore
Completeproject.md
FullScaffoldGeneration.md
missingspecelements.md
Pass1.md
pass2.md
...
pass24.md            ← generated in Pass 24 above
package.json
pnpm-workspace.yaml
tsconfig.base.json
turbo.json
README.md
tobodone.md
If any root config file is missing, it will be generated when you execute Pass 1.

2.2 Verify pnpm-workspace.yaml covers all packages
After executing Pass 1, open pnpm-workspace.yaml and confirm it reads:

YAML

packages:
  - 'apps/*'
  - 'packages/*'
⚠️ Critical: If Pass 1 generated an explicit list of packages by name instead of a glob, you must manually add every package. The full list of expected workspace members is:

apps:

text

apps/cli
apps/dashboard
apps/desktop
apps/gateway
packages:

text

packages/core
packages/memory
packages/graphify
packages/wiki
packages/kairos
packages/orchestrator
packages/autoresearch
packages/mirofish
packages/keychain        ← added in Pass 24
packages/shared
packages/tools-fs
packages/tools-shell
packages/tools-web
packages/tools-code
Any package not in pnpm-workspace.yaml will not be symlinked and cross-package imports will fail with Cannot find module '@locoworker/...'.

2.3 Verify root package.json
After Pass 1, root package.json should have:

JSON

{
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "clean": "turbo run clean",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "turbo": "^2.x.x",
    "typescript": "^5.4.x",
    "prettier": "^3.x.x",
    "eslint": "^8.x.x or ^9.x.x"
  }
}
⚠️ If "workspaces" key is absent from root package.json, pnpm workspace resolution will still work via pnpm-workspace.yaml — but turbo may not find packages. Confirm turbo.json is present and valid.

2.4 Verify root tsconfig.base.json
This is the shared TypeScript config that every package extends. After Pass 1 it must exist at ./tsconfig.base.json with at least:

JSON

{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "bundler",
    "target": "ES2022",
    "lib": ["ES2022"],
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
⚠️ moduleResolution: "bundler" — some packages (gateway, CLI) are Node.js processes, not bundled. Those packages' tsconfig.json should override with "moduleResolution": "node16" or "nodenext". Check passes 2, 8, 14 for gateway/cli/core tsconfigs specifically.

2.5 Verify root tsconfig.json project references
After all passes are applied, root tsconfig.json must reference every package:

JSON

{
  "files": [],
  "references": [
    { "path": "packages/shared" },
    { "path": "packages/core" },
    { "path": "packages/memory" },
    { "path": "packages/graphify" },
    { "path": "packages/wiki" },
    { "path": "packages/kairos" },
    { "path": "packages/orchestrator" },
    { "path": "packages/autoresearch" },
    { "path": "packages/mirofish" },
    { "path": "packages/keychain" },
    { "path": "packages/tools-fs" },
    { "path": "packages/tools-shell" },
    { "path": "packages/tools-web" },
    { "path": "packages/tools-code" },
    { "path": "apps/gateway" },
    { "path": "apps/cli" },
    { "path": "apps/dashboard" },
    { "path": "apps/desktop" }
  ]
}
⚠️ packages/keychain must be in this list — it was added in Pass 24 and may not have been added to the root references.

2.6 Verify turbo.json pipeline
After Pass 24's patch to turbo.json, the file must contain:

JSON

{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "clean": {
      "cache": false
    },
    "@locoworker/desktop#build": {
      "dependsOn": [
        "@locoworker/dashboard#build",
        "@locoworker/gateway#build",
        "^build"
      ],
      "outputs": ["dist/**", "out/**"]
    }
  }
}
⚠️ The @locoworker/desktop#build override is critical — without it, turbo build may build desktop before dashboard/gateway, causing electron-builder to package empty extraResources.

2.7 Verify .env.example is complete
The root .env.example must include every environment variable referenced across all passes. After Pass 24 it should contain at minimum:

Bash

# ─── LLM Provider Keys (stored in OS keychain for desktop; set as env vars for CLI/gateway) ───
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_API_KEY=

# ─── Gateway ──────────────────────────────────────────────────────────────────────────────────
LOCOWORKER_GATEWAY_HOST=127.0.0.1
LOCOWORKER_GATEWAY_PORT=8787
LOCOWORKER_WORKSPACE=

# CORS: comma-separated list of allowed origins
LOCOWORKER_CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173

# ─── Desktop embedded gateway port ────────────────────────────────────────────────────────────
# This is what the desktop app uses when spawning gateway as a child process.
# Must be different from LOCOWORKER_GATEWAY_PORT if you run both simultaneously.
DESKTOP_EMBEDDED_GATEWAY_PORT=3001

# ─── Desktop dev helpers ──────────────────────────────────────────────────────────────────────
# Set to true to prevent desktop from spawning an embedded gateway (use with separate gateway dev process)
LOCOWORKER_SKIP_EMBEDDED_GATEWAY=false

# ─── Dashboard ────────────────────────────────────────────────────────────────────────────────
# For standalone web / development use. Not used when running inside Electron.
VITE_LOCOWORKER_GATEWAY_URL=http://127.0.0.1:8787

# ─── CLI ──────────────────────────────────────────────────────────────────────────────────────
LOCOWORKER_CLI_GATEWAY_URL=http://127.0.0.1:8787

# ─── LLM / model config ───────────────────────────────────────────────────────────────────────
LOCOWORKER_DEFAULT_MODEL=claude-3-5-sonnet-20241022
LOCOWORKER_BUDGET_LIMIT=10.00

# ─── Memory / compaction ──────────────────────────────────────────────────────────────────────
LOCOWORKER_COMPACTION_MODE=auto
LOCOWORKER_CONTEXT_WINDOW_LIMIT=180000

# ─── Graphify ─────────────────────────────────────────────────────────────────────────────────
LOCOWORKER_GRAPH_DB_PATH=.locoworker/graph.db

# ─── Wiki ─────────────────────────────────────────────────────────────────────────────────────
LOCOWORKER_WIKI_DB_PATH=.locoworker/wiki.db

# ─── Kairos ───────────────────────────────────────────────────────────────────────────────────
LOCOWORKER_KAIROS_DB_PATH=.locoworker/kairos.db
LOCOWORKER_AUTODREAM_ENABLED=true
LOCOWORKER_AUTODREAM_HOUR=3

# ─── Orchestrator ─────────────────────────────────────────────────────────────────────────────
LOCOWORKER_MAX_PARALLEL_AGENTS=4

# ─── AutoResearch ─────────────────────────────────────────────────────────────────────────────
LOCOWORKER_AUTORESEARCH_MAX_SOURCES=10

# ─── Telemetry (opt-in only) ──────────────────────────────────────────────────────────────────
LOCOWORKER_TELEMETRY_ENABLED=false

# ─── Node ─────────────────────────────────────────────────────────────────────────────────────
NODE_ENV=development
Copy .env.example to .env before first run:

Bash

cp .env.example .env
# Then fill in your API keys
Section 3 — Pass 1 Verification Checklist
Execute Pass 1's scaffold generation first, then verify each item below before proceeding to Pass 2.

3.1 Root scaffold files — verify presence after Pass 1
text

✓ package.json
✓ pnpm-workspace.yaml
✓ turbo.json
✓ tsconfig.base.json
✓ tsconfig.json           (project references file — may be empty references list initially)
✓ .eslintrc.js            (or eslint.config.js / eslint.config.mjs)
✓ .prettierrc
✓ .prettierignore
✓ .gitignore
✓ .env.example
✓ .github/
✓   workflows/
✓     ci.yml              (or build.yml — whatever Pass 1 / Pass 18 generates)
✓ changeset/              (if Pass 1 includes changesets)
✓   config.json
3.2 Verify .gitignore covers all generated artifacts
Confirm these are present in .gitignore:

text

node_modules/
dist/
out/
.turbo/
.env
*.local
.locoworker/
*.db
*.db-shm
*.db-wal
coverage/
3.3 Verify GitHub Actions workflow (Pass 18 / Pass 1)
After Pass 18, .github/workflows/ci.yml (or build.yml) must include:

YAML

# Platform matrix must include all three OS
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]

# Linux step — must install libsecret BEFORE pnpm install
- name: Install system dependencies (Linux)
  if: runner.os == 'Linux'
  run: |
    sudo apt-get update
    sudo apt-get install -y libsecret-1-dev libnss3 libatk1.0-0 \
      libatk-bridge2.0-0 libgdk-pixbuf2.0-0 libgtk-3-0 libgbm1 libasound2

# Node setup
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: '20'

# pnpm setup BEFORE install
- name: Setup pnpm
  uses: pnpm/action-setup@v3
  with:
    version: 9

# Native module rebuild for keytar
- name: Rebuild native modules
  run: pnpm --filter @locoworker/desktop rebuild keytar
⚠️ If Pass 18's workflow does not include the libsecret-1-dev install step, add it manually. Linux CI builds will fail at keytar compilation otherwise.

3.4 Install dependencies (first time only)
Bash

# From repo root:
pnpm install

# This will:
# 1. Install all workspace packages
# 2. Symlink cross-package dependencies
# 3. Run postinstall scripts (including electron-rebuild for keytar)
Expected output includes something like:

text

Packages: +NNN
Progress: resolved NNN, reused NNN, downloaded NNN, added NNN, done

. prepare$ electron-rebuild -f -w keytar
If electron-rebuild fails:

Bash

# Try manual rebuild
cd apps/desktop
pnpm rebuild:native
3.5 Verify workspace symlinks
Bash

# After pnpm install, verify cross-package symlinks exist
ls node_modules/@locoworker/
# Should list: core, memory, graphify, wiki, kairos, orchestrator,
#              gateway, autoresearch, mirofish, keychain, shared,
#              tools-fs, tools-shell, tools-web, tools-code
If any package is missing from node_modules/@locoworker/, it means either:

It's not in pnpm-workspace.yaml → fix the glob or add it explicitly
Its package.json has the wrong "name" field → check each package's package.json name matches @locoworker/<package-name>
Section 4 — Root-level Configuration Patches
These are cross-cutting config files that affect every package and must be correct before building any individual package.

4.1 tsconfig.base.json — module resolution split
The tsconfig.base.json from Pass 1 uses a single moduleResolution setting. This needs to work for both:

Bundled packages (dashboard, vite-based): use "bundler"
Node.js processes (gateway, CLI, core, all packages): use "node16" or "nodenext"
Recommended approach: Set the base to "node16" (most compatible) and let the dashboard override it:

In tsconfig.base.json:

JSON

{
  "compilerOptions": {
    "moduleResolution": "node16",
    "module": "Node16",
    "target": "ES2022",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
In apps/dashboard/tsconfig.json (override for Vite):

JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "moduleResolution": "bundler",
    "module": "ESNext",
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "noEmit": true
  },
  "include": ["src"]
}
In apps/gateway/tsconfig.json (override for CJS child process spawn):

JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
⚠️ Gateway MUST be CommonJS — GatewayProcessManager in Pass 24 spawns it via process.execPath [gatewayEntry]. A Node.js child process spawned this way cannot load an ES module entry point without the --input-type=module flag or a type: "module" in the gateway's package.json. CJS is the safe, unambiguous choice.

In apps/desktop/tsconfig.json:

JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
4.2 turbo.json — final verified state
This is the canonical turbo.json after all passes. Write this as the final version:

JSON

{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "out/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**/*.ts", "test/**/*.ts", "vitest.config.*"]
    },
    "lint": {
      "outputs": [],
      "inputs": ["src/**/*.ts", ".eslintrc*", "eslint.config*"]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "clean": {
      "cache": false
    },
    "@locoworker/desktop#build": {
      "dependsOn": [
        "@locoworker/dashboard#build",
        "@locoworker/gateway#build",
        "^build"
      ],
      "outputs": ["dist/**", "out/**"]
    },
    "@locoworker/gateway#dev": {
      "cache": false,
      "persistent": true
    },
    "@locoworker/dashboard#dev": {
      "cache": false,
      "persistent": true
    }
  }
}
4.3 Prettier config — verify .prettierrc
JSON

{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false
}
And .prettierignore:

text

node_modules
dist
out
.turbo
coverage
*.md
pnpm-lock.yaml
4.4 ESLint — verify base config covers all packages
If Pass 1 generated .eslintrc.js:

JavaScript

module.exports = {
  root: true,
  parser: "@typescript-eslint/parser",
  plugins: ["@typescript-eslint"],
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
  ],
  rules: {
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/no-unused-vars": ["warn", { "argsIgnorePattern": "^_" }],
    "no-console": "off",
  },
  ignorePatterns: ["dist/", "out/", "node_modules/", "*.js"],
};
Add *.js to ignorePatterns so ESLint doesn't try to lint compiled output.

Section 5 — .locoworker/ workspace directory
Several packages write SQLite databases and cache files to a .locoworker/ directory relative to the workspace root.

5.1 Create the workspace directory structure
Bash

mkdir -p .locoworker
Add to .gitignore (verify it's already there from 3.2):

text

.locoworker/
Packages that write to .locoworker/:

Package	File	Env var
packages/graphify	.locoworker/graph.db	LOCOWORKER_GRAPH_DB_PATH
packages/wiki	.locoworker/wiki.db	LOCOWORKER_WIKI_DB_PATH
packages/kairos	.locoworker/kairos.db	LOCOWORKER_KAIROS_DB_PATH
packages/memory	.locoworker/memory.db	(check Pass 3 for exact var name)
packages/orchestrator	.locoworker/locks/	(file-based advisory locks)
⚠️ If any package resolves its DB path relative to process.cwd() rather than LOCOWORKER_WORKSPACE, running the CLI or gateway from a different directory will create .locoworker/ in the wrong place. Verify each package uses the env var, not process.cwd().

End of Part 1
Before proceeding to Part 2, confirm:

 All system prerequisites installed (Node 20, pnpm 9, platform libs)
 Repo cloned, root structure matches Section 2
 pnpm-workspace.yaml uses packages/* glob
 Root tsconfig.base.json uses "moduleResolution": "node16"
 Root turbo.json has @locoworker/desktop#build dependency override
 .env.example contains all vars from Section 2.7; .env created from it
 pnpm install completes without errors
 ls node_modules/@locoworker/ shows all 14 packages
 apps/gateway/tsconfig.json uses "module": "CommonJS"
 .github/workflows/ci.yml includes libsecret-1-dev install for Linux
Part 2 of 5 — Packages: Core, Memory, Graphify, Wiki, Kairos
Section 6 — packages/shared
packages/shared is the base utility layer that every other package imports. It must be built first.

6.1 Verify packages/shared/package.json
JSON

{
  "name": "@locoworker/shared",
  "version": "0.1.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.js",
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  }
}
⚠️ The exports field must have both require and import pointing to the CJS output. If later passes import @locoworker/shared as an ES module but the output is CJS, you'll get runtime errors. The safest approach for shared is to build to CJS only.

6.2 Verify packages/shared exports
packages/shared/src/index.ts must re-export everything that other packages need. Common items expected based on the spec:

TypeScript

// Check that these are all exported from packages/shared/src/index.ts:
export * from "./types/index.js";       // shared type definitions
export * from "./utils/index.js";       // utility functions
export * from "./errors/index.js";      // base error classes
export * from "./events/index.js";      // shared event type definitions
export * from "./logger/index.js";      // shared logger (pino or similar)
6.3 Build packages/shared first
Bash

pnpm --filter @locoworker/shared build
# Verify dist/ is created:
ls packages/shared/dist/
# Should contain: index.js, index.d.ts, index.js.map, index.d.ts.map
Section 7 — packages/core
Core is the central engine. Every app (gateway, CLI, desktop) depends on it.

7.1 Verify packages/core/package.json dependencies
After Pass 2, core's package.json must list @locoworker/shared as a workspace dependency:

JSON

{
  "dependencies": {
    "@locoworker/shared": "workspace:*"
  }
}
If it's missing, add it. Without it, pnpm install won't symlink shared into core's node_modules.

7.2 Verify packages/core module resolution
Core's tsconfig.json must use the same "module": "CommonJS" / "moduleResolution": "node" override. Confirm:

JSON

{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "dist",
    "rootDir": "src"
  }
}
7.3 Verify core's queryLoop.ts is the Pass 19 version
Pass 19 rewrites queryLoop.ts as an async generator. The canonical signature should be:

TypeScript

export async function* queryLoop(
  input: QueryInput,
  context: QueryContext,
  options?: QueryOptions
): AsyncGenerator<AgentEvent>
If Pass 2's version is a Promise-returning function instead of an async generator, it has been superseded by Pass 19. Confirm the file you end up with is the async generator version — Pass 19 wins.

7.4 Verify AgentEvent union type is complete
After Pass 19, the AgentEvent union in packages/core/src/types/agent.types.ts (or wherever it's defined) must include all event variants referenced across the codebase:

TypeScript

export type AgentEvent =
  | { type: "turn_start"; sessionId: string; runId: string; }
  | { type: "model_delta"; delta: string; }
  | { type: "tool_call"; toolName: string; toolInput: unknown; toolCallId: string; }
  | { type: "tool_result"; toolCallId: string; result: unknown; error?: string; }
  | { type: "permission_prompt"; promptId: string; permission: string; description: string; }
  | { type: "permission_resolved"; promptId: string; granted: boolean; }
  | { type: "budget_warning"; used: number; limit: number; }
  | { type: "compaction_triggered"; mode: "micro" | "auto" | "full"; }
  | { type: "turn_end"; usage: TokenUsage; }
  | { type: "run_end"; reason: "complete" | "budget_exceeded" | "permission_denied" | "error"; }
  | { type: "error"; message: string; code?: string; };
⚠️ Dashboard (Pass 23) and CLI (Pass 21) both switch/match on these event type values. If any variant is missing or misnamed, those renderers will silently drop events.

7.5 Verify ToolRegistry and ToolExecutor are both present
Pass 2 introduces ToolRegistry. Pass 19 introduces (or rewrites) ToolExecutor. Both must be present and the executor must use the registry. Check:

Bash

ls packages/core/src/
# Should include: ToolRegistry.ts (or ToolRegistry/) and ToolExecutor.ts
7.6 Verify PermissionGate wires promptId (Pass 20 requirement)
Pass 20 adds promptId metadata to permission prompts so gateway can broker approvals. Check packages/core/src/PermissionGate.ts (or similar):

TypeScript

// Should emit something like:
yield {
  type: "permission_prompt",
  promptId: crypto.randomUUID(),   // ← must be present
  permission: tier,
  description: description,
};
If promptId is missing from the emitted event, gateway's permission resolution endpoints (Pass 20) will have no way to correlate approval responses back to the waiting queryLoop.

7.7 Build packages/core
Bash

pnpm --filter @locoworker/core build
ls packages/core/dist/
Section 8 — packages/memory
8.1 Verify SQLite dependency
Pass 3 uses SQLite for ConversationStore and MemoryIndex. Confirm the dependency in packages/memory/package.json:

JSON

{
  "dependencies": {
    "better-sqlite3": "^9.x.x"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.x.x"
  }
}
better-sqlite3 is a native module. After pnpm install it must be compiled for the current Node version. If it fails:

Bash

pnpm --filter @locoworker/memory rebuild better-sqlite3
8.2 Verify 4-layer memory hierarchy is implemented
The spec's 4-layer model (system prompt → project instructions → conversation history → memory index) must be reflected in MemoryManager. Check packages/memory/src/MemoryManager.ts for a method that assembles context in this order. If it only handles conversation history, the system prompt and project instructions layers are missing.

8.3 Verify AutoDream is registered with Kairos (Pass 22)
Pass 22 patches memory to register AutoDream as a Kairos-scheduled task. Check packages/memory/src/AutoDream.ts for a call to KairosScheduler.register(...) or equivalent. If it's a standalone function that never gets scheduled, AutoDream will never run.

8.4 Verify LOCOWORKER_WORKSPACE is used for DB path resolution
TypeScript

// In packages/memory/src/ConversationStore.ts (or wherever the DB is opened):
const dbPath = process.env.LOCOWORKER_WORKSPACE
  ? path.join(process.env.LOCOWORKER_WORKSPACE, ".locoworker", "memory.db")
  : path.join(process.cwd(), ".locoworker", "memory.db");
8.5 Build packages/memory
Bash

pnpm --filter @locoworker/memory build
Section 9 — packages/graphify
9.1 Verify Tree-sitter dependencies
Pass 4 uses Tree-sitter for AST parsing. Check packages/graphify/package.json:

JSON

{
  "dependencies": {
    "tree-sitter": "^0.21.x",
    "tree-sitter-typescript": "^0.21.x",
    "tree-sitter-javascript": "^0.21.x",
    "tree-sitter-python": "^0.21.x",
    "tree-sitter-rust": "^0.21.x"
  }
}
Tree-sitter is a native module. After pnpm install:

Bash

pnpm --filter @locoworker/graphify rebuild
9.2 Verify SQLite is used for graph storage
Bash

grep -r "better-sqlite3" packages/graphify/
# Should find it in package.json and the graph store implementation
9.3 Verify graph DB path uses LOCOWORKER_WORKSPACE
Same pattern as memory — DB path must resolve relative to workspace, not process.cwd().

9.4 Verify graph query client is exported
TypeScript

// packages/graphify/src/index.ts should export:
export { GraphifyClient } from "./client/GraphifyClient.js";
export { GraphBuilder } from "./builder/GraphBuilder.js";
export type { GraphNode, GraphEdge, GraphCluster, GraphQuery } from "./types.js";
9.5 Build packages/graphify
Bash

pnpm --filter @locoworker/graphify build
Section 10 — packages/wiki
10.1 Verify markdown parser dependency
Pass 5 uses a markdown parser. Check packages/wiki/package.json for one of:

unified + remark-parse + remark-stringify
marked
micromark
Whichever is used, confirm it's in dependencies (not devDependencies).

10.2 Verify wikilinks dead-link detection
Pass 5 describes a dead-link detector. Check that packages/wiki/src/linker/ (or similar) has a function that scans for [[WikiLink]] patterns and cross-references them against the wiki store index. Without this, the wiki's backlinks feature will silently produce incorrect data.

10.3 Verify wiki DB path uses LOCOWORKER_WORKSPACE
Same workspace pattern as memory and graphify.

10.4 Build packages/wiki
Bash

pnpm --filter @locoworker/wiki build
Section 11 — packages/kairos
11.1 Verify Pass 22 "real engine" is the version used
Pass 6 created a hollow Kairos. Pass 22 rewrote it as a real durable scheduler. The canonical packages/kairos/src/scheduler/KairosScheduler.ts (or similar) must come from Pass 22 — it should include:

A persistent task queue (SQLite-backed)
Cron expression evaluation
A tick() method that the engine calls on a set interval
AutoDream registration support
If the file still looks like a stub, Pass 22's content has not been applied — re-apply it.

11.2 Verify AutoDream scheduling hook
After Pass 22 and the memory package's AutoDream implementation, Kairos must have AutoDream registered at startup. Check packages/kairos/src/engine/KairosEngine.ts for a call like:

TypeScript

this.scheduler.register({
  id: "autodream",
  cron: `0 ${process.env.LOCOWORKER_AUTODREAM_HOUR ?? 3} * * *`,
  handler: () => autoDream.run(),
  enabled: process.env.LOCOWORKER_AUTODREAM_ENABLED !== "false",
});
11.3 Verify Kairos DB path uses LOCOWORKER_WORKSPACE
Bash

grep -r "LOCOWORKER_WORKSPACE\|kairos.db" packages/kairos/src/
11.4 Verify Gateway adapter is present (Pass 22 addition)
Pass 22 adds a Gateway adapter for Kairos so gateway routes can start/stop/query the scheduler. Check packages/kairos/src/gateway/KairosGatewayAdapter.ts exists.

11.5 Build packages/kairos
Bash

pnpm --filter @locoworker/kairos build
Section 12 — Native module rebuild verification
After all packages are built, run a final native module check:

Bash

# Verify all native modules compiled correctly for current Node version
node -e "require('better-sqlite3')"
node -e "require('keytar')"
node -e "require('tree-sitter')"

# If any fail:
pnpm rebuild
# Or for a specific package:
cd packages/graphify && node-gyp rebuild
End of Part 2
Before proceeding to Part 3, confirm:

 packages/shared builds cleanly — dist/ populated
 packages/core builds cleanly — queryLoop is async generator (Pass 19 version)
 AgentEvent union has all variants including permission_prompt with promptId
 packages/memory builds — DB path uses LOCOWORKER_WORKSPACE
 packages/graphify builds — Tree-sitter native modules compiled
 packages/wiki builds — markdown parser in dependencies
 packages/kairos builds — Pass 22 real engine version (not Pass 6 stub)
 packages/keychain builds — keytar compiled for current Node version
 All native modules verified: better-sqlite3, keytar, tree-sitter
Part 3 of 5 — Packages: Orchestrator, AutoResearch, Mirofish, Tools
Section 13 — packages/orchestrator
13.1 Verify Pass 22 "real engine" is the version used
Like Kairos, Pass 7 created a hollow Orchestrator. Pass 22 replaced it. The canonical packages/orchestrator/src/engine/OrchestratorEngine.ts from Pass 22 must include:

DelegationPlanner — decomposes a goal into sub-tasks
FileLockManager — advisory file locks to prevent multi-agent write collisions
AgentSupervisor — runs work units through queryLoop from Pass 19
ResultAggregator — collects and merges outputs from parallel agents
If any of these are stubs, re-apply Pass 22's orchestrator sections.

13.2 Verify FileLockManager uses workspace-relative paths
FileLockManager must resolve lock files relative to LOCOWORKER_WORKSPACE, not process.cwd():

TypeScript

// Expected pattern:
const lockDir = path.join(
  process.env.LOCOWORKER_WORKSPACE ?? process.cwd(),
  ".locoworker",
  "locks"
);
13.3 Verify AgentSupervisor imports from packages/core
AgentSupervisor runs sub-agents via queryLoop. It must import from @locoworker/core:

TypeScript

import { queryLoop } from "@locoworker/core";
Check that packages/orchestrator/package.json has:

JSON

{
  "dependencies": {
    "@locoworker/core": "workspace:*",
    "@locoworker/shared": "workspace:*"
  }
}
13.4 Verify max parallel agents is respected
TypeScript

// packages/orchestrator/src/engine/OrchestratorEngine.ts
const maxParallel = parseInt(
  process.env.LOCOWORKER_MAX_PARALLEL_AGENTS ?? "4",
  10
);
13.5 Verify Gateway adapter is present (Pass 22 addition)
Pass 22 adds Gateway routes for orchestrator. Check packages/orchestrator/src/gateway/OrchestratorGatewayAdapter.ts exists and exports route handlers for:

POST /v1/orchestrator/jobs — start a multi-agent job
GET /v1/orchestrator/jobs/:jobId — get job status
DELETE /v1/orchestrator/jobs/:jobId — cancel a job
13.6 Build packages/orchestrator
Bash

pnpm --filter @locoworker/orchestrator build
Section 14 — packages/autoresearch
14.1 Verify multi-source adapters are implemented
Pass 9 describes adapters for: web, code (graphify), graph, wiki, memory. Check packages/autoresearch/src/sources/:

Bash

ls packages/autoresearch/src/sources/
# Expected: WebAdapter.ts, GraphifyAdapter.ts, WikiAdapter.ts, MemoryAdapter.ts
# (CodeAdapter may be merged with GraphifyAdapter)
If any adapter is a stub (returns empty results), it should be marked with a // TODO comment — leave those for post-build but note them.

14.2 Verify dependency chain in package.json
JSON

{
  "dependencies": {
    "@locoworker/core": "workspace:*",
    "@locoworker/graphify": "workspace:*",
    "@locoworker/wiki": "workspace:*",
    "@locoworker/memory": "workspace:*",
    "@locoworker/shared": "workspace:*"
  }
}
14.3 Verify Kairos integration for scheduled research
Pass 9 describes Kairos integration for long-running research. Check that packages/autoresearch does not directly import @locoworker/kairos in a way that creates a circular dependency (kairos → core → autoresearch → kairos). If circular dependency exists, break it by using a callback/handler pattern rather than a direct import.

14.4 Build packages/autoresearch
Bash

pnpm --filter @locoworker/autoresearch build
Section 15 — packages/mirofish
15.1 Verify Docker Compose integration is optional
Pass 10 describes Docker Compose scenario fragments. This must be optional (gracefully skipped if Docker is not installed). Check:

TypeScript

// packages/mirofish/src/orchestrator/ScenarioOrchestrator.ts
// Should have a Docker availability check:
const dockerAvailable = await checkDockerAvailable();
if (!dockerAvailable) {
  console.warn("[mirofish] Docker not available — skipping container-based scenarios");
}
⚠️ If mirofish unconditionally requires Docker, the build will fail in CI unless Docker is installed on every runner. Make Docker optional.

15.2 Verify persona factory uses seeded randomness
Pass 10 specifies "seeded deterministic traits" for persona generation. Check packages/mirofish/src/personas/PersonaFactory.ts uses a seedable PRNG (e.g., seedrandom or a custom LCG), not Math.random().

15.3 Build packages/mirofish
Bash

pnpm --filter @locoworker/mirofish build
Section 16 — packages/tools-*
Passes 11–12 generate the tool packages. These are straightforward but have a few critical checks.

16.1 Verify all tool packages are present
Bash

ls packages/
# Must include: tools-fs, tools-shell, tools-web, tools-code
16.2 Verify each tool package exports a register function
Every tool package must export a function that registers its tools with ToolRegistry:

TypeScript

// packages/tools-fs/src/index.ts
export function registerFsTools(registry: ToolRegistry): void {
  registry.register(readFileTool);
  registry.register(writeFileTool);
  registry.register(listDirectoryTool);
  // ...
}
If a tool package exports individual tool definitions but no register function, the gateway and CLI won't know how to load them without manual wiring.

16.3 Verify workspace boundary enforcement in tools-fs and tools-shell
Pass 19 introduces workspace boundary enforcement as a "central choke point." Every file-system and shell tool must check that paths stay within LOCOWORKER_WORKSPACE:

TypeScript

// Expected in tools-fs:
function assertWithinWorkspace(filePath: string): void {
  const workspace = process.env.LOCOWORKER_WORKSPACE ?? process.cwd();
  const resolved = path.resolve(filePath);
  if (!resolved.startsWith(path.resolve(workspace))) {
    throw new WorkspaceBoundaryError(
      `Path "${resolved}" is outside workspace "${workspace}"`
    );
  }
}
If this check is missing from tools-fs and tools-shell, any agent can read/write/execute arbitrary paths on the host — a critical security gap.

16.4 Verify tools-shell has command allowlist/denylist
Pass 11 or 12 should implement a command safety layer. Check packages/tools-shell/src/ for an allowlist or denylist of shell commands. If absent, any agent can run any shell command — add a basic denylist at minimum:

TypeScript

const DENIED_COMMANDS = [
  "rm -rf /",
  "sudo",
  "chmod 777",
  "curl | sh",
  "wget | sh",
  // ... etc
];
16.5 Build all tool packages
Bash

pnpm --filter "@locoworker/tools-*" build
End of Part 3
Before proceeding to Part 4, confirm:

 packages/orchestrator builds — Pass 22 engine (not Pass 7 stub)
 FileLockManager uses LOCOWORKER_WORKSPACE-relative paths
 packages/autoresearch builds — no circular Kairos import
 packages/mirofish builds — Docker is optional, not required
 All packages/tools-* build
 tools-fs and tools-shell have workspace boundary enforcement
 tools-shell has a command denylist
Part 4 of 5 — Apps: Gateway, CLI, Dashboard, Desktop
Section 17 — apps/gateway
The gateway is the most critical app — CLI, dashboard, and desktop all depend on it being correct.

17.1 Verify gateway entry point compiles to CJS at dist/main.js
Bash

pnpm --filter @locoworker/gateway build
ls apps/gateway/dist/
# Must contain: main.js (CJS format)

# Quick verify it's CJS not ESM:
head -5 apps/gateway/dist/main.js
# Should NOT start with: export default ...
# Should contain: Object.defineProperty(exports, "__esModule", ...
# OR use: module.exports = ...
17.2 Verify GatewayRuntime.fromEnv() reads all required env vars
Pass 20's GatewayRuntime must read:

TypeScript

static fromEnv(): GatewayRuntime {
  return new GatewayRuntime({
    host: process.env.LOCOWORKER_GATEWAY_HOST ?? "127.0.0.1",
    port: parseInt(process.env.LOCOWORKER_GATEWAY_PORT ?? "8787", 10),
    workspace: process.env.LOCOWORKER_WORKSPACE ?? process.cwd(),
    corsOrigins: (process.env.LOCOWORKER_CORS_ORIGINS ?? "")
      .split(",")
      .map((s) => s.trim())
      .filter(Boolean),
    defaultModel: process.env.LOCOWORKER_DEFAULT_MODEL,
    budgetLimit: process.env.LOCOWORKER_BUDGET_LIMIT
      ? parseFloat(process.env.LOCOWORKER_BUDGET_LIMIT)
      : undefined,
    apiKey: process.env.ANTHROPIC_API_KEY
      ?? process.env.OPENAI_API_KEY
      ?? process.env.GOOGLE_API_KEY,
  });
}
⚠️ If corsOrigins is not split on commas, LOCOWORKER_CORS_ORIGINS=http://127.0.0.1:5173,locoworker-app://dashboard will be treated as a single malformed origin, breaking all dashboard requests.

17.3 Verify /v1/health endpoint exists and returns 200
Bash

# Start gateway in one terminal:
pnpm --filter @locoworker/gateway dev

# In another terminal:
curl http://127.0.0.1:8787/v1/health
# Expected: {"status":"ok","version":"0.1.0"} or similar JSON with 200 status
If /v1/health returns 404, add it to the gateway router. This is the health check used by both GatewayProcessManager (Pass 24) and the dashboard's GatewayClient.health().

17.4 Verify SSE stream endpoint exists and streams AgentEvent JSON
Bash

# After creating a session and starting a run:
curl -N "http://127.0.0.1:8787/v1/sessions/test/runs/test/stream"
# Should stream: data: {"type":"...","...":...}\n\n lines
Verify the SSE response has correct headers:

text

Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Access-Control-Allow-Origin: <allowed origin>
17.5 Verify permission resolution endpoints exist (Pass 20)
Bash

# Check route exists:
curl -X POST http://127.0.0.1:8787/v1/permissions/promptId123/resolve \
  -H "Content-Type: application/json" \
  -d '{"granted": true}'
# Expected: 200 OK
If this returns 404, Pass 20's permission broker routes are not applied. These are required for the dashboard's permissions UI (Pass 23) to function.

17.6 Verify Kairos + Orchestrator adapter routes are registered (Pass 22)
Bash

curl http://127.0.0.1:8787/v1/kairos/tasks
# Expected: 200 with JSON array

curl http://127.0.0.1:8787/v1/orchestrator/jobs
# Expected: 200 with JSON array
17.7 Verify CORS allows dashboard origin
Bash

curl -I -X OPTIONS http://127.0.0.1:8787/v1/health \
  -H "Origin: http://127.0.0.1:5173" \
  -H "Access-Control-Request-Method: GET"
# Response must include:
# Access-Control-Allow-Origin: http://127.0.0.1:5173
And for production Electron:

Bash

curl -I -X OPTIONS http://127.0.0.1:3001/v1/health \
  -H "Origin: locoworker-app://dashboard" \
  -H "Access-Control-Request-Method: GET"
# Response must include:
# Access-Control-Allow-Origin: locoworker-app://dashboard
17.8 Verify all tool packages are registered at gateway startup
In apps/gateway/src/main.ts or gateway bootstrap:

TypeScript

import { registerFsTools } from "@locoworker/tools-fs";
import { registerShellTools } from "@locoworker/tools-shell";
import { registerWebTools } from "@locoworker/tools-web";
import { registerCodeTools } from "@locoworker/tools-code";

// All must be called before the first session is created:
registerFsTools(runtime.toolRegistry);
registerShellTools(runtime.toolRegistry);
registerWebTools(runtime.toolRegistry);
registerCodeTools(runtime.toolRegistry);
If tools are not registered, the agent will run but no tools will be available.

Section 18 — apps/cli
18.1 Verify Pass 21 CLI bodies are used (not Pass 14 stubs)
Pass 14 (original scaffold) had CLI commands with TODO bodies. Pass 21 rewrote them. The canonical apps/cli/src/commands/ must have real implementations for:

chat.ts — interactive REPL/chat mode
run.ts — single-shot query execution
session.ts — session management
config.ts — configuration management
If any command body is a stub, re-apply Pass 21's content.

18.2 Verify CLI has two operating modes: direct and gateway
Pass 21 explicitly implements:

Direct mode: imports queryLoop directly from @locoworker/core
Gateway mode: calls gateway HTTP API + streams SSE via EventSource
Check apps/cli/src/commands/run.ts for a mode flag:

TypeScript

const mode = options.gateway ? "gateway" : "direct";
If only one mode exists, the CLI is incomplete.

18.3 Verify AgentEvent terminal renderer exists
Pass 21 adds a terminal renderer for streaming events. Check:

Bash

ls apps/cli/src/
# Should include: renderer.ts (or render/index.ts)
The renderer must handle all AgentEvent types from Section 7.4.

18.4 Verify permission handler respects --yes flag and non-TTY
TypeScript

// In apps/cli/src/handlers/permissionHandler.ts (or similar):
async function handlePermissionPrompt(event: PermissionPromptEvent): Promise<boolean> {
  if (options.yes) return true;                    // --yes flag: auto-approve all
  if (!process.stdin.isTTY) return false;          // non-TTY: auto-deny safely
  return await promptUserInteractively(event);     // TTY: ask the user
}
If non-TTY behavior is missing, running the CLI in CI or piped contexts will hang waiting for input.

18.5 Verify CLI package.json has a bin entry
JSON

{
  "name": "@locoworker/cli",
  "bin": {
    "locoworker": "./dist/index.js",
    "lcw": "./dist/index.js"
  }
}
Without the bin entry, pnpm link and global installs won't create the CLI executable.

18.6 Verify CLI entry point has the Node.js shebang
TypeScript

// apps/cli/src/index.ts — first line must be:
#!/usr/bin/env node
If missing, the CLI binary won't be executable on Unix systems.

18.7 Build and smoke-test CLI
Bash

pnpm --filter @locoworker/cli build

# Smoke test (direct mode — no gateway required):
node apps/cli/dist/index.js --version
# Expected: print version number

node apps/cli/dist/index.js --help
# Expected: print command list
Section 19 — apps/dashboard
19.1 Verify base: "./" is in vite.config.ts (Pass 24 patch)
TypeScript

// apps/dashboard/vite.config.ts
export default defineConfig({
  base: "./",   // ← MUST be present for Electron production loading
  // ...
});
Without this, all asset src and href attributes in the built index.html will be absolute paths (/assets/...), which will 404 when loaded via locoworker-app://dashboard/index.html.

19.2 Verify resolveInitialGatewayUrl uses query param approach (Pass 24 fix)
In apps/dashboard/src/store/gatewaySlice.ts, confirm the URL resolution function uses ?gatewayUrl= query parameter, not window.__LOCOWORKER_GATEWAY_URL__:

TypeScript

function resolveInitialGatewayUrl(): string {
  if (typeof window !== "undefined") {
    const params = new URLSearchParams(window.location.search);
    const fromQuery = params.get("gatewayUrl");
    if (fromQuery) return fromQuery;
  }
  return import.meta.env.VITE_LOCOWORKER_GATEWAY_URL ?? "http://127.0.0.1:8787";
}
19.3 Verify window.locoworker TypeScript declaration
The file apps/dashboard/src/electron.d.ts must exist:

TypeScript

import type { LocoWorkerBridge } from "../../desktop/src/preload/index";

declare global {
  interface Window {
    locoworker?: LocoWorkerBridge;
  }
}

export {};
Without this, window.locoworker.gateway.getUrl() calls in dashboard components will produce TypeScript errors and the build will fail.

19.4 Verify App.tsx applies injected gateway URL on mount
TypeScript

// apps/dashboard/src/App.tsx
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  const injected = params.get("gatewayUrl");
  if (injected) {
    useStore.getState().setGatewayUrl(injected);
  }
}, []);
19.5 Verify EventSource SSE client handles reconnection
Pass 23's SSE client uses browser EventSource. Check apps/dashboard/src/api/sseClient.ts (or similar) for reconnection handling:

TypeScript

// EventSource auto-reconnects by default but you should handle errors:
es.onerror = (err) => {
  console.error("[sse] Connection error:", err);
  // Update store with connection error state
};
19.6 Verify permission approval UI calls the correct gateway endpoint
Pass 23's permissions panel must call POST /v1/permissions/:promptId/resolve. Check:

TypeScript

// In apps/dashboard/src/api/gatewayClient.ts or similar:
async resolvePermission(promptId: string, granted: boolean): Promise<void> {
  await fetch(`${this.baseUrl}/v1/permissions/${promptId}/resolve`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ granted }),
  });
}
19.7 Build dashboard
Bash

pnpm --filter @locoworker/dashboard build
ls apps/dashboard/dist/
# Must contain: index.html, assets/
Verify base: "./" worked:

Bash

grep 'src=' apps/dashboard/dist/index.html
# Should show: src="./assets/..." NOT src="/assets/..."
Section 20 — apps/desktop
20.1 Verify apps/desktop/build/entitlements.mac.plist exists
Bash

ls apps/desktop/build/
# Must contain: entitlements.mac.plist
If missing, create it with the content from Part 1 Section 5 (Critical item #5). macOS builds will fail at codesign step without it.

20.2 Verify packages/keychain is in desktop's package.json dependencies
JSON

{
  "dependencies": {
    "@locoworker/keychain": "workspace:*",
    "keytar": "^7.9.0"
  }
}
20.3 Verify LOCOWORKER_SKIP_EMBEDDED_GATEWAY dev guard exists
In apps/desktop/src/main/index.ts, verify the dev bypass guard (from Part 1 Section 3.12) is implemented. Without it, running pnpm dev in desktop will spawn a second gateway that conflicts with any separately-run gateway on the same port.

20.4 Verify loadURL uses ?gatewayUrl= query param (not executeJavaScript)
In apps/desktop/src/main/index.ts in the "ready" status handler:

TypeScript

// CORRECT (Pass 24 final approach):
const dashboardWithGateway = `${getDashboardUrl()}?gatewayUrl=${encodeURIComponent(gatewayManager.url)}`;
mainWindow.loadURL(dashboardWithGateway);

// WRONG (race condition — do NOT use):
// mainWindow.webContents.executeJavaScript(`window.__LOCOWORKER_GATEWAY_URL__ = "..."`)
20.5 Verify CORS origin is passed as origin not full URL
In apps/desktop/src/main/index.ts in the GatewayProcessManager constructor call:

TypeScript

LOCOWORKER_CORS_ORIGINS: IS_DEV
  ? "http://127.0.0.1:5173"
  : "locoworker-app://dashboard",   // ← origin only, NOT full URL with path
20.6 Verify electron-builder.yml extraResources paths are correct
The paths in extraResources are relative to the apps/desktop/ directory:

YAML

extraResources:
  - from: "../dashboard/dist"    # relative to apps/desktop/
    to: "dashboard"
  - from: "../gateway/dist"
    to: "gateway"
  - from: "../gateway/node_modules"
    to: "gateway/node_modules"
⚠️ gateway/node_modules bundling is important but can produce very large installers. Consider using electron-builder's asar carefully — gateway's node_modules must NOT be in an asar archive since it contains native .node files. Verify electron-builder.yml has:

YAML

asarUnpack:
  - "**/*.node"
  - "gateway/**/*"
20.7 Verify vite.main.config.ts externalizes all node_modules
The regex /^[a-z@][^/]*/ in rollupOptions.external externalizes all node_modules from the main process bundle. Verify this is present — without it, keytar, better-sqlite3 etc. get bundled and their native modules break.

20.8 Build desktop (after dashboard and gateway are built)
Bash

# Ensure dashboard and gateway are built first
pnpm --filter @locoworker/dashboard build
pnpm --filter @locoworker/gateway build

# Copy dashboard to desktop dist for local testing
bash apps/desktop/scripts/copy-dashboard.sh

# Build desktop main + preload
pnpm --filter @locoworker/desktop build:vite

# Verify:
ls apps/desktop/dist/main/
# Should contain: index.js

ls apps/desktop/dist/preload/
# Should contain: preload.js
20.9 Test desktop in development mode
Bash

# Terminal 1 — Dashboard dev server
pnpm --filter @locoworker/dashboard dev

# Terminal 2 — Desktop (with skip flag to avoid spawning second gateway)
LOCOWORKER_SKIP_EMBEDDED_GATEWAY=true pnpm --filter @locoworker/desktop dev
Expected: Electron window opens, loads dashboard at http://127.0.0.1:5173, ConnectionPanel shows gateway URL.

End of Part 4
Before proceeding to Part 5, confirm:

 apps/gateway builds to CJS dist/main.js
 Gateway /v1/health returns 200
 Gateway SSE stream endpoint streams AgentEvent JSON lines
 Gateway permission resolution endpoints exist
 Gateway CORS allows both dashboard origins (dev + Electron prod)
 All tool packages registered at gateway startup
 apps/cli builds — Pass 21 version with direct + gateway modes
 CLI bin entry and shebang present
 apps/dashboard builds — base: "./" confirmed in dist index.html
 window.locoworker TypeScript declaration exists
 apps/desktop/build/entitlements.mac.plist exists
 Desktop loadURL uses ?gatewayUrl= query param
 CORS origin passed as origin not full URL
 electron-builder.yml has asarUnpack for native modules and gateway
Part 5 of 5 — Integration Verification, Build Order, Testing & Final Checklist
Section 21 — Integration Verification
These tests verify the full system works end-to-end before packaging.

21.1 Full build in dependency order
Bash

# From repo root, run the full build in correct order:
pnpm turbo run build

# Turbo will resolve the dependency graph automatically.
# Watch the output — any red line means a package failed.
# Common failures at this stage:
#   - TypeScript errors (type mismatches across packages)
#   - Missing exports (package A exports X but package B imports Y)
#   - Native module compile errors (better-sqlite3, keytar, tree-sitter)
21.2 TypeScript strict-mode check across all packages
Bash

# From repo root:
pnpm turbo run typecheck
# OR if no typecheck script exists yet:
tsc --build tsconfig.json --dry
Fix all TypeScript errors before proceeding. Common issues:

Error	Fix
Cannot find module '@locoworker/X'	Package not in pnpm-workspace.yaml or not built
Module '"@locoworker/core"' has no exported member 'X'	Export missing from packages/core/src/index.ts
Property 'promptId' does not exist on type 'PermissionPromptEvent'	AgentEvent union not updated to Pass 20 version
Type 'AsyncGenerator' is not assignable to type 'Promise'	Pass 19 queryLoop signature not applied
Property 'locoworker' does not exist on type 'Window'	apps/dashboard/src/electron.d.ts missing
21.3 Gateway + CLI smoke test (direct mode)
Bash

# Start gateway:
cp .env.example .env
# Fill in ANTHROPIC_API_KEY (or OPENAI_API_KEY) in .env
source .env

pnpm --filter @locoworker/gateway dev &
GATEWAY_PID=$!

# Wait for gateway to be ready:
sleep 3
curl http://127.0.0.1:8787/v1/health

# Run a minimal CLI query in gateway mode:
node apps/cli/dist/index.js run \
  --gateway \
  --gateway-url http://127.0.0.1:8787 \
  "Hello, what tools do you have available?"

# Expected: streaming output in terminal with tool list

kill $GATEWAY_PID
21.4 Dashboard standalone smoke test
Bash

# Start gateway:
pnpm --filter @locoworker/gateway dev &

# Start dashboard dev server:
pnpm --filter @locoworker/dashboard dev &

# Open http://127.0.0.1:5173 in browser
# Expected:
#   - ConnectionPanel shows gateway URL http://127.0.0.1:8787
#   - Clicking "Connect" calls /v1/health and shows "Connected"
#   - Creating a session and starting a run streams events in the timeline
21.5 End-to-end permission flow smoke test
Bash

# In a query that triggers a permission prompt (e.g., write a file):
node apps/cli/dist/index.js run \
  --gateway-url http://127.0.0.1:8787 \
  "Write 'hello world' to a file called test.txt"

# Expected flow:
# 1. queryLoop emits permission_prompt event with promptId
# 2. CLI receives event and prompts: "[permission] Allow write to test.txt? [y/N]"
# 3. User types 'y'
# 4. CLI calls POST /v1/permissions/:promptId/resolve with {granted: true}
# 5. queryLoop resumes and writes the file
# 6. run_end event received
21.6 Kairos scheduler smoke test
Bash

# Via gateway API:
curl -X POST http://127.0.0.1:8787/v1/kairos/tasks \
  -H "Content-Type: application/json" \
  -d '{"name":"test","cron":"* * * * *","enabled":true}'

curl http://127.0.0.1:8787/v1/kairos/tasks
# Expected: array containing the new task
21.7 Orchestrator smoke test
Bash

curl -X POST http://127.0.0.1:8787/v1/orchestrator/jobs \
  -H "Content-Type: application/json" \
  -d '{"goal":"List all TypeScript files in the workspace"}'

# Wait a moment, then:
curl http://127.0.0.1:8787/v1/orchestrator/jobs
# Expected: job with status "running" or "complete"
21.8 Desktop integration test
Bash

# Build everything:
pnpm turbo run build

# Run desktop in production-like mode (with embedded gateway):
pnpm --filter @locoworker/desktop dev
# (Without LOCOWORKER_SKIP_EMBEDDED_GATEWAY — let it spawn its own gateway)

# Expected:
# 1. Electron window opens (hidden initially)
# 2. Tray icon appears showing "starting"
# 3. After gateway health check passes:
#    - Tray shows "ready"
#    - Window becomes visible
#    - Dashboard loads at ?gatewayUrl=http://127.0.0.1:3001
# 4. ConnectionPanel in dashboard shows http://127.0.0.1:3001 (the embedded port)
Section 22 — Known Gaps to Document Before Building
These are features described in the spec that are explicitly not yet implemented in Passes 1–24. Document them so you don't spend time debugging missing features that were never built:

22.1 packages/keychain — not wired into CLI
CLI in Pass 21 reads API keys from environment variables. It does NOT read from the OS keychain (that's desktop-only). This is intentional but should be noted: CLI users must always set ANTHROPIC_API_KEY etc. as env vars.

22.2 packages/telemetry — not implemented
missingspecelements.md lists this as missing. No telemetry package exists in any pass. The env var LOCOWORKER_TELEMETRY_ENABLED=false is in .env.example but nothing reads it. Safe to ignore for now.

22.3 packages/plugins — not implemented
Plugin system described in spec but not built in any pass. Ignore for now.

22.4 packages/context — not implemented
Additional context management layer described in spec but not present in any pass. The memory package handles context for now.

22.5 Desktop auto-updater — not testable locally
Pass 24's updater.ts is wired but requires a published GitHub release to test. In local development, autoUpdater.checkForUpdates() will log a warning and no-op — this is expected.

22.6 packages/mirofish Docker scenarios — not testable without Docker
Docker-based scenario orchestration requires Docker Desktop. The non-Docker persona/evaluation harness still works without it.

22.7 Web search tool in packages/tools-web — requires external API
The web adapter in packages/autoresearch and packages/tools-web likely calls an external search API (Tavily, Brave Search, SerpAPI, or similar). This requires its own API key not listed in .env.example. Add when implementing:

Bash

# Add to .env.example:
TAVILY_API_KEY=
# or
BRAVE_SEARCH_API_KEY=
Section 23 — Final Pre-Build Checklist (Canonical Summary)
Work through this list top-to-bottom before running pnpm turbo run build for the first time.

Environment
 Node.js v20 LTS installed and active (node --version)
 pnpm 9.x installed (pnpm --version)
 Turborepo installed globally (turbo --version)
 Platform system deps installed (libsecret, build-essential on Linux; Xcode CLI on macOS; VS Build Tools on Windows)
Repository Root
 pnpm-workspace.yaml uses packages/* and apps/* globs
 Root package.json has all workspace scripts
 tsconfig.base.json uses "moduleResolution": "node16"
 Root tsconfig.json references all 18 packages (including packages/keychain)
 turbo.json has @locoworker/desktop#build depending on dashboard + gateway builds
 .env created from .env.example with API keys filled in
 LOCOWORKER_SKIP_EMBEDDED_GATEWAY=false in .env.example
Pass Application Order
 Pass 1 applied — root scaffold
 Pass 2 applied — packages/core (verify Pass 19 queryLoop overwrites Pass 2 version)
 Pass 3 applied — packages/memory
 Pass 4 applied — packages/graphify
 Pass 5 applied — packages/wiki
 Pass 6 applied — packages/kairos (verify Pass 22 engine overwrites Pass 6 stub)
 Pass 7 applied — packages/orchestrator (verify Pass 22 engine overwrites Pass 7 stub)
 Pass 8 applied — apps/gateway (verify Pass 20 GatewayRuntime overwrites Pass 8 stubs)
 Pass 9 applied — packages/autoresearch
 Pass 10 applied — packages/mirofish
 Pass 11 applied — security / shared
 Pass 12 applied — packages/tools-*
 Pass 13 applied — apps/desktop initial (will be overwritten by Pass 24)
 Pass 14 applied — apps/cli initial (will be overwritten by Pass 21)
 Pass 15 applied — apps/dashboard initial (will be overwritten by Pass 23)
 Pass 16 applied — desktop updates
 Pass 17 applied — tests
 Pass 18 applied — CI/CD workflows
 Pass 19 applied — queryLoop async generator spine (overwrites Pass 2 loop)
 Pass 20 applied — gateway runtime + SSE + permission broker (overwrites Pass 8)
 Pass 21 applied — runnable CLI (overwrites Pass 14)
 Pass 22 applied — Kairos + Orchestrator real engines (overwrites Pass 6 + 7)
 Pass 23 applied — dashboard real client (overwrites Pass 15)
 Pass 24 applied — desktop embedded gateway + keychain (overwrites Pass 13)
Critical File Patches (from pre-build analysis)
 apps/gateway/tsconfig.json — "module": "CommonJS", outDir: "dist"
 apps/dashboard/vite.config.ts — base: "./" present
 apps/dashboard/src/store/gatewaySlice.ts — resolveInitialGatewayUrl uses ?gatewayUrl= query param
 apps/dashboard/src/App.tsx — useEffect applies ?gatewayUrl= to store on mount
 apps/dashboard/src/electron.d.ts — window.locoworker?: LocoWorkerBridge declaration
 apps/desktop/src/main/index.ts — loadURL uses ?gatewayUrl=... encoding
 apps/desktop/src/main/index.ts — LOCOWORKER_CORS_ORIGINS is origin not full URL
 apps/desktop/src/main/index.ts — LOCOWORKER_SKIP_EMBEDDED_GATEWAY dev guard present
 apps/desktop/build/entitlements.mac.plist — file exists
 apps/desktop/electron-builder.yml — asarUnpack: ["**/*.node", "gateway/**/*"] present
 apps/desktop/package.json — copy:dashboard script wired into build
 .github/workflows/ci.yml — libsecret-1-dev installed before pnpm install on Linux
 packages/core/src/PermissionGate.ts — promptId: crypto.randomUUID() in emitted event
 packages/tools-fs/src/ — workspace boundary assertWithinWorkspace present
 packages/tools-shell/src/ — command denylist present
Native Modules
 better-sqlite3 compiled for current Node version
 keytar compiled for current Node version
 tree-sitter and language parsers compiled
 All native modules verified: node -e "require('better-sqlite3')" etc.
Build Verification
 pnpm --filter @locoworker/shared build — ✓
 pnpm --filter @locoworker/core build — ✓
 pnpm --filter @locoworker/memory build — ✓
 pnpm --filter @locoworker/graphify build — ✓
 pnpm --filter @locoworker/wiki build — ✓
 pnpm --filter @locoworker/kairos build — ✓
 pnpm --filter @locoworker/orchestrator build — ✓
 pnpm --filter @locoworker/autoresearch build — ✓
 pnpm --filter @locoworker/mirofish build — ✓
 pnpm --filter @locoworker/keychain build — ✓
 pnpm --filter "@locoworker/tools-*" build — ✓
 pnpm --filter @locoworker/gateway build — ✓ (CJS output at dist/main.js)
 pnpm --filter @locoworker/cli build — ✓
 pnpm --filter @locoworker/dashboard build — ✓ (dist/index.html uses ./assets/)
 pnpm --filter @locoworker/desktop build:vite — ✓
Integration Smoke Tests
 Gateway /v1/health returns 200
 CLI --help prints without errors
 Dashboard loads in browser and connects to gateway
 Permission flow works end-to-end (prompt → approve → continue)
 Kairos API responds correctly
 Orchestrator API responds correctly
 Desktop launches, spawns gateway, loads dashboard at injected gateway URL
Known Gaps (not blocking — document and skip)
 packages/telemetry — not built (env var present but unused)
 packages/plugins — not built
 packages/context — not built
 packages/tools-web — may need external search API key
 Desktop auto-updater — not testable without published GitHub release
 Mirofish Docker scenarios — not testable without Docker Desktop
Quick Reference: "Later Pass Wins" File Override Map
File	Canonical Pass
packages/core/src/queryLoop.ts	Pass 19
packages/core/src/ToolExecutor.ts	Pass 19
packages/core/src/PermissionGate.ts	Pass 20 (promptId addition)
packages/kairos/src/engine/KairosEngine.ts	Pass 22
packages/orchestrator/src/engine/OrchestratorEngine.ts	Pass 22
apps/gateway/src/runtime/GatewayRuntime.ts	Pass 20
apps/gateway/src/routes/	Pass 20 (+ Pass 22 adapters)
apps/cli/src/commands/	Pass 21
apps/dashboard/src/store/	Pass 23
apps/dashboard/vite.config.ts	Pass 24 (base: "./")
apps/desktop/src/main/index.ts	Pass 24
apps/desktop/src/main/ipcHandlers.ts	Pass 24
apps/desktop/src/main/tray.ts	Pass 24
apps/desktop/src/main/config.ts	Pass 24
apps/desktop/src/preload/index.ts	Pass 24
apps/desktop/src/shared/ipcTypes.ts	Pass 24
apps/desktop/electron-builder.yml	Pass 24
apps/desktop/package.json	Pass 24
turbo.json	Pass 24 (desktop build dependency)
