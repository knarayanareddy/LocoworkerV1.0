HANDOFF/README.md
Markdown

# LocoWorker — LLM Agent Handoff Package (for Pass 1 → Pass 24 replay)

## What you are building
You are NOT compiling an existing monorepo.
This repository contains a set of Markdown "passes" that each describe files to generate.
Your job is to:
1) Materialize the monorepo filesystem from Pass 1 → Pass 24
2) Apply the post-pass patchset (HANDOFF/04-POSTPASS-PATCHSET.md)
3) Install deps and produce a successful build + smoke tests

## Inputs (source docs)
- Pass 1: `Pass1.md`
- Passes 2–23: `pass2.md` ... `pass23.md`
- Pass 24: `pass24.md` (provided separately — this repo does not currently contain it)
- Design/spec context: `Completeproject.md`
- Normalization/override guidance: `pass16.md`
- Providers/MCP/OpenClaw scaffolding: `pass17.md`
- ADR decisions + env example update + pass map update: `pass18.md`

## Critical repo facts (as of main branch)
- The repo root currently contains pass/spec Markdown files only; there is no generated `apps/` or `packages/` tree yet.
- `FullScaffoldGeneration.md` in the repo is outdated and still mentions "apps/desktop (React + Tauri)". Later passes and ADRs supersede this.
- Desktop stack is Electron (ADR-002), not Tauri.

## Golden rules
1) Later pass wins:
   If two passes write the same path, keep ONLY the highest-numbered pass's version.
   (Pass 16 explicitly documents this rule and provides a conflict table.)
2) "Complete final files":
   Many passes expect full replacement, not partial edits.
3) After replay, apply HANDOFF/04-POSTPASS-PATCHSET.md (there are known cross-pass inconsistencies).

## Minimal success criteria
- `pnpm install` succeeds
- `pnpm build` succeeds (turbo pipeline OK)
- Gateway starts and responds to `/v1/health`
- Dashboard can connect to Gateway `/v1/health` and subscribe to SSE stream
- CLI runs at least one prompt in direct mode and gateway mode
- Desktop launches and loads the dashboard UI (Pass 24 behavior)

Proceed in this order:
1) Read HANDOFF/01-DECISIONS-AND-SCOPE.md
2) Follow HANDOFF/02-PASS-REPLAY-ALGORITHM.md
3) Apply HANDOFF/03-CANONICAL-PASS-MAP.md
4) Apply HANDOFF/04-POSTPASS-PATCHSET.md
5) Execute HANDOFF/05-BUILD-RUNBOOK.md
File: HANDOFF/01-DECISIONS-AND-SCOPE.md
Markdown

# Decisions & Scope (Authoritative for replay)

## Desktop stack: Electron, not Tauri
There is an intentional mismatch in the repository:
- An early pass map file mentions desktop as "React + Tauri".
- Later passes implement Electron Desktop, and Pass 18 contains ADR-002:
  "Desktop Surface: Electron + Vite instead of Tauri" (Accepted).

Treat Electron as canonical for Pass 13+ and Pass 24.

Evidence:
- Outdated map mentions "apps/desktop (React + Tauri)". (Superseded.) 
- Pass 13 scaffolds Electron desktop and uses Electron concepts (ipc types, Electron build scripts).
- Pass 18 ADR-002 explicitly accepts Electron over Tauri, and lists it as an intentional deviation from spec.

## CLI surface: Node (readline/commander) rather than Ink
Pass 18 includes ADR-003 regarding CLI surface choices.
Pass 21 rewrites CLI in apps/cli with commander + node:readline/promises.

## "missingspecelements.md" is non-authoritative
It currently appears empty in repo. Ignore it and rely on Pass 16/17/18/19–23 content.

## Event schema: `AgentEvent.kind`
Dashboard SSE client and CLI renderer in later passes refer to `event.kind` (not `event.type`).
Do not invent a different schema. Keep `kind` consistent across:
- packages/core event types
- packages/gateway SSE formatting
- apps/dashboard event list rendering
- apps/cli renderer

## Scope: buildable dev monorepo first, packaging second
Pass 24 introduces embedded gateway + desktop integration.
A working dev build is required; electron-builder packaging is secondary and may require extra work with pnpm symlinks.
Do not block the build on installer generation unless explicitly requested.
File: HANDOFF/02-PASS-REPLAY-ALGORITHM.md
Markdown

# Pass replay algorithm (how to turn the Markdown passes into real files)

This is the most important operational doc. Follow it exactly.

## Terminology
- "pass file" = `Pass1.md` or `passN.md`
- "emit a file" = a pass provides full contents for a given path (usually via a heading like `## path` or `### \`path\`` + a code block)
- "overwrite" = later pass emits same path; later content replaces earlier entirely

## Rule 1 — Later pass wins
If path P is emitted by multiple passes:
- Keep ONLY the content from the highest pass number that emits P
- Do not attempt line merges unless a pass explicitly says "patch this line"

Pass 16 explicitly lists several multi-written root files and says the highest pass number is authoritative.

## Rule 2 — Only treat explicit path+content as file emission
Many passes include:
- high-level file trees
- commentary
- checklists
These are NOT files.

A file is emitted only when BOTH are true:
- A concrete file path is shown (commonly in a markdown heading like `## \`path\`` or `### \`path\``)
- A fenced code block follows that contains the entire file content

## Rule 3 — Normalize paths
Canonical root is the repo root (e.g., `locoworker/` in diagrams = `./`).

Example:
- If a pass shows `locoworker/packages/core/src/index.ts`, write to `./packages/core/src/index.ts`.

## Rule 4 — Preserve exact file bytes
- Keep JSON ordering and punctuation as shown.
- Preserve shebangs at top of CLI entrypoint.
- Preserve `type: "module"` vs not in package.json as given.
- Preserve `.js` extension on internal TS imports where the pass includes them (NodeNext expectation).

## Rule 5 — Apply Pass 16/17/18 root overwrites faithfully
Passes 16–18 rewrite key repo root files (workspace, root tsconfig refs, env example, etc.).
Do not keep Pass 1 versions of these files.

## Output: canonical monorepo tree
After emitting all files:
- you should have `packages/*`, `apps/*`, `tests/*`, `.github/workflows/*`, `scripts/*`, `.vscode/*`
- then apply post-pass patchset (HANDOFF/04)

## Suggested implementation strategy (for an LLM agent)
Maintain an in-memory map:

`files: Map<string, { pass: number, content: string }>`
For each pass in [1..24]:
- parse file emissions
- set `files[path] = {pass, content}` if (pass >= existing.pass)
After pass loop:
- write map to disk
Then run patchset.
File: HANDOFF/03-CANONICAL-PASS-MAP.md
Markdown

# Canonical pass map + key overrides

This map is derived from later-pass "state of play" documentation and pass internals.

## Files that are overwritten across passes (known hotspots)
Pass 16 explicitly provides a conflict table for root-level files:
- root `package.json`, `pnpm-workspace.yaml`, `turbo.json`, `tsconfig.base.json`, `tsconfig.json`, `.env.example`, `.gitignore`, biome/prettier config
- workflows and scripts
Treat Pass 16 as canonical for these root files unless a higher pass rewrites them.

## Major components by pass (as documented in Pass 16)
Pass 16 includes a "spec alignment status" table mapping:
- packages/core = Pass 2
- memory = Pass 3
- graphify = Pass 4
- wiki = Pass 5
- kairos = Pass 6
- orchestrator = Pass 7
- packages/gateway = Pass 8
- autoresearch = Pass 9
- mirofish = Pass 10
- tools (fs/bash/git/search/web) + shared = Pass 11
- security + apps/cli + apps/gateway = Pass 12
- apps/desktop (Electron) + apps/dashboard = Pass 13
- tests/integration + tests/e2e = Pass 14
- Docker + docs = Pass 15

## Providers / MCP / OpenClaw additions
Pass 16 says these were missing and targeted for Pass 17.
Pass 17 creates:
- packages/providers
- packages/tools-mcp
- openclaw/
and updates root workspace + root tsconfig references accordingly.

## ADRs and deviations from spec
Pass 18 includes ADRs, including:
- ADR-002: Electron desktop instead of Tauri
- ADR-003: Node CLI instead of Ink
Treat ADR-002 as the explicit resolution of the Tauri/Electron discrepancy.

## "Make it real" sequence (Pass 19–23)
- Pass 19: deep core loop wiring (`queryLoop` async generator, permissions, budget, context injection)
- Pass 20: Gateway SSE + permission broker + `/v1/health`, sessions/runs routes, adapters
- Pass 21: CLI rewrite (commander, direct + gateway modes, non-tty handling)
- Pass 22: (as referenced by Pass 19/23) autonomy engines / integration improvements
- Pass 23: Dashboard rewrite (GatewayClient + EventSource SSE, permissions UI, kairos/orchestrator panels)

## Pass 24
Not present in the repo; you must supply it.
Pass 23 explicitly frames Pass 24 as: embed gateway + reuse same REST/SSE contract + desktop IPC surface.
File: HANDOFF/04-POSTPASS-PATCHSET.md
This file is the “gap-remediation layer” that makes the replay buildable and consistent.

Markdown

# Post-pass patchset (must apply after replay)

This patchset remedies cross-pass inconsistencies that will otherwise cause build or runtime failures.

Apply in the order listed.

---

## Patch A — Add missing `@locoworker/tsconfig` package OR remove references to it

### Why
Pass 19/20/21/23 repeatedly reference extending `@locoworker/tsconfig/base.json` and include `@locoworker/tsconfig` as a devDependency, but the workspace list in Pass 16 does not include a `packages/tsconfig` entry, and no pass emits it.
Result: `tsc` fails resolving extends + pnpm fails resolving workspace dependency.

### Option A1 (recommended): CREATE `packages/tsconfig`
Emit these files:

`packages/tsconfig/package.json`
```json
{
  "name": "@locoworker/tsconfig",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "files": ["base.json"],
  "exports": {
    "./base.json": "./base.json"
  }
}
packages/tsconfig/base.json

Copy the EXACT tsconfig.base.json content from Pass 16 (canonical base config).
Then update:

pnpm-workspace.yaml (final version after Pass 17) to include:
* "packages/tsconfig"
root tsconfig.json references (final version after Pass 17) to include:
{ "path": "packages/tsconfig" } (it can be a non-composite reference; harmless)
root package.json may include no changes.
Option A2: Remove @locoworker/tsconfig usage everywhere
Replace all "extends": "@locoworker/tsconfig/base.json" with "extends": "../../tsconfig.base.json" (or correct relative path). Remove @locoworker/tsconfig from all devDependencies. This is more invasive and easy to miss.

Patch B — Harmonize environment variable names (Pass 16/18 vs Pass 20/21/23)
Why
Pass 16/18 .env.example uses LOCOWORKER_WORKSPACE_ROOT and GATEWAY_PORT/GATEWAY_CORS_ORIGINS. (and Pass 18 explicitly replaces Pass 16’s env example)
Pass 20 gateway config reads LOCOWORKER_WORKSPACE, LOCOWORKER_GATEWAY_PORT, LOCOWORKER_CORS_ORIGINS, LOCOWORKER_MODEL_PROVIDER, LOCOWORKER_MODEL, etc.
Pass 21 CLI reads LOCOWORKER_WORKSPACE, LOCOWORKER_GATEWAY_URL, and model/budget vars.
If not harmonized, your gateway/cli/dashboard will run with defaults unexpectedly.

Required fix
Update packages/gateway/src/config/GatewayConfig.ts (from Pass 20) so env lookup falls back:

working dir:

env.LOCOWORKER_WORKSPACE ?? env.LOCOWORKER_WORKSPACE_ROOT ?? process.cwd()
host/port:

host: env.LOCOWORKER_GATEWAY_HOST ?? env.GATEWAY_HOST ?? "127.0.0.1"
port: env.LOCOWORKER_GATEWAY_PORT ?? env.GATEWAY_PORT ?? 8787/3000 (choose one default; consistent across apps)
cors origins:

env.LOCOWORKER_CORS_ORIGINS ?? env.GATEWAY_CORS_ORIGINS
model defaults:

provider: env.LOCOWORKER_MODEL_PROVIDER ?? env.DEFAULT_PROVIDER ?? "anthropic"
model: env.LOCOWORKER_MODEL ?? env.DEFAULT_MODEL ?? "claude-3-5-sonnet-latest"
apiKey: env.LOCOWORKER_API_KEY ?? env.ANTHROPIC_API_KEY ?? env.OPENAI_API_KEY ?? env.GEMINI_API_KEY ...
Update apps/cli config loader (Pass 21) similarly:

workspace:
env.LOCOWORKER_WORKSPACE ?? env.LOCOWORKER_WORKSPACE_ROOT ?? process.cwd()
gatewayUrl:
env.LOCOWORKER_GATEWAY_URL ?? "http://127.0.0.1:<port>"
Update .env.example final:

Keep Pass 18 structure, but add LOCOWORKER_* variants as aliases at the top (commented as "new naming").
Add LOCOWORKER_GATEWAY_URL and VITE_LOCOWORKER_GATEWAY_URL.
Patch C — Root package.json scripts must match latest app package names
Why
Pass 16 root package.json uses filters that do not match later app renames introduced by Pass 20/21/23. Example: Pass 21 sets apps/cli/package.json name to @locoworker/app-cli and bin targets. Pass 20 sets apps/gateway to @locoworker/app-gateway. Pass 23 sets dashboard to @locoworker/app-dashboard. If root scripts still filter old names, pnpm dev:* and pnpm build:* will fail or target wrong packages.

Required fix
Patch root package.json scripts (final root version from Pass 16) so:

dev:cli filters @locoworker/app-cli
dev:gateway filters @locoworker/app-gateway
dev:dashboard filters @locoworker/app-dashboard
desktop remains @locoworker/desktop
Patch D — Dashboard must support Desktop runtime gateway URL injection
Why
Pass 23 dashboard store defaults to import.meta.env.VITE_LOCOWORKER_GATEWAY_URL ?? "http://127.0.0.1:8787". Desktop embedded gateway likely uses DESKTOP_EMBEDDED_GATEWAY_PORT (Pass 16/18 mention it in env example). Without runtime injection, the dashboard will point at the wrong port.

Required fix (robust)
Desktop loads dashboard with a query param:

...?gatewayUrl=http%3A%2F%2F127.0.0.1%3A<port>
Dashboard store resolves initial gateway URL from query param first:

new URLSearchParams(window.location.search).get("gatewayUrl")
On mount, dashboard applies it to store and client instance.

Patch E — Dashboard production build must set base: "./" (Electron file/protocol loading)
Why
Pass 23’s vite.config.ts as emitted does not set base, so asset links become /assets/.... When loaded via Electron custom protocol or file loading, /assets/... may fail.

Required fix
In apps/dashboard/vite.config.ts, set:

base: "./"
Patch F — Add apps/dashboard/src/electron.d.ts so TS knows window.locoworker
If dashboard references window.locoworker (from desktop preload), TS will fail without a global type declaration. Add a declaration file that makes it optional.

Patch G — Desktop embedded gateway: make entrypoint + env consistent with Pass 20
If Pass 24 spawns a gateway process:

Ensure it runs apps/gateway/dist/main.js (Pass 20 app entry)
Ensure it passes BOTH env naming conventions (old + new), at least:
DESKTOP_EMBEDDED_GATEWAY_PORT (desktop config)
LOCOWORKER_GATEWAY_PORT and/or GATEWAY_PORT
LOCOWORKER_CORS_ORIGINS and/or GATEWAY_CORS_ORIGINS
LOCOWORKER_WORKSPACE and/or LOCOWORKER_WORKSPACE_ROOT
Patch H — Linux CI: install libsecret headers if keytar/keychain is used
If Pass 24 introduces keychain via keytar/keytar-based deps:

Linux CI must install libsecret-1-dev before pnpm install.
Patch .github/workflows/ci.yml accordingly.

Patch I — Workspace + root tsconfig references: include Pass 24 new packages
If Pass 24 adds packages/keychain:

Update pnpm-workspace.yaml to include it
Update root tsconfig.json references to include it
If you implement Patch A1 (packages/tsconfig), include that too.

text


---

## File: `HANDOFF/05-BUILD-RUNBOOK.md`

```markdown
# Build & Runbook (dev-first)

## 0) Preconditions
- Node >= 20 (Pass 16 base expects Node 20; `.nvmrc` is 20.15.0 in Pass 16)
- pnpm >= 9
- OS deps:
  - Linux: libsecret-1-dev if keytar/keychain is present

## 1) Generate filesystem from passes
- Replay Pass 1 → Pass 24 using HANDOFF/02 algorithm.
- Ensure "later pass wins" resolution.

## 2) Apply patchset
- Apply HANDOFF/04-POSTPASS-PATCHSET.md in order.

## 3) Install
pnpm install

## 4) Typecheck + build
pnpm typecheck
pnpm build

If turbo exists:
pnpm turbo run typecheck
pnpm turbo run build

## 5) Smoke tests (manual)

### 5.1 Gateway
Start:
pnpm --filter @locoworker/app-gateway dev
Then:
curl http://127.0.0.1:<port>/v1/health

Dashboard expects `/v1/health` (Pass 23 GatewayClient.health()).

### 5.2 Dashboard
pnpm --filter @locoworker/app-dashboard dev
Open http://127.0.0.1:5173 and connect to gateway URL.

### 5.3 CLI (direct mode)
pnpm --filter @locoworker/app-cli dev -- ask "hello"

### 5.4 CLI (gateway mode)
pnpm --filter @locoworker/app-cli dev -- ask --gateway --gateway-url http://127.0.0.1:<port> "hello"

### 5.5 Desktop
- Start dashboard dev server first (if Pass 24 loads dev URL in dev)
- Run desktop dev command
Confirm it launches and dashboard UI connects to embedded gateway.

## 6) Common failure triage
- "Cannot find module @locoworker/tsconfig":
  Apply Patch A (create packages/tsconfig) or remove references
- "Gateway runs but dashboard can't connect":
  likely env var mismatch (Patch B) or CORS mismatch
- "Dashboard assets 404 in electron":
  set `base: "./"` (Patch E)
- "pnpm filter doesn't find package":
  root scripts mismatch (Patch C)
