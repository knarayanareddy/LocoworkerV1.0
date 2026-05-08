# Build Orchestration Guide for LLM Agents

## Purpose
This document provides the exact execution order and rules for building the LocoWorker project from Passes 1-24.

## Critical rules
1. **"Later pass wins"** — if multiple passes emit the same file path, only the **highest-numbered pass's version** is canonical.
2. **Desktop stack = Electron** — ignore Tauri references in `Completeproject.md`. See `DESKTOP_STACK_DECISION.md`.
3. **Apply patches AFTER passes** — the pre-build checklist has 16+ patches that must be applied to pass-generated files.

## Pass application order

### Phase 1: Root scaffold (Pass 1 only)
Apply Pass 1 to create root config files. Then:
- **Patch 1.1**: Verify `pnpm-workspace.yaml` uses `packages/*` glob (not explicit list)
- **Patch 1.2**: Add `packages/keychain` to root `tsconfig.json` project references (it's added in Pass 24 but Pass 1 won't know about it)

### Phase 2: Packages (Passes 2-12, with Pass 19-22 overwrites)
Apply in this order. Where noted, a later pass **replaces** an earlier one's files entirely.

1. **Pass 2** → `packages/core` (initial scaffold)
2. **Pass 19** → `packages/core` (**replaces** Pass 2's `queryLoop.ts`, `ToolExecutor.ts`)
   - Files to overwrite: `src/queryLoop.ts`, `src/ToolExecutor.ts`
   - Files to keep from Pass 2: `src/types/*`, `src/ToolRegistry.ts`, `src/SessionManager.ts`

3. **Pass 3** → `packages/memory`

4. **Pass 4** → `packages/graphify`

5. **Pass 5** → `packages/wiki`

6. **Pass 6** → `packages/kairos` (initial stub)
7. **Pass 22** → `packages/kairos` (**replaces** Pass 6's `src/engine/*`, `src/scheduler/*`)
   - Files to overwrite: ALL under `src/engine/`, `src/scheduler/`
   - Files to add: `src/gateway/KairosGatewayAdapter.ts`

8. **Pass 7** → `packages/orchestrator` (initial stub)
9. **Pass 22** → `packages/orchestrator` (**replaces** Pass 7's `src/engine/*`)
   - Files to overwrite: `src/engine/OrchestratorEngine.ts`, etc.
   - Files to add: `src/FileLockManager.ts`, `src/AgentSupervisor.ts`, `src/gateway/OrchestratorGatewayAdapter.ts`

10. **Pass 9** → `packages/autoresearch`

11. **Pass 10** → `packages/mirofish`

12. **Pass 11** → `packages/security` or `packages/shared` (check exact package name in Pass 11)

13. **Pass 12** → `packages/tools-fs`, `packages/tools-shell`, `packages/tools-web`, `packages/tools-code`

### Phase 3: Apps (Passes 8, 13-16, with Pass 20-21-23-24 overwrites)

1. **Pass 8** → `apps/gateway` (initial scaffold)
2. **Pass 20** → `apps/gateway` (**replaces** Pass 8's `src/runtime/*`, `src/routes/*`)
   - **Patch 2.1**: Set `"module": "CommonJS"` in `apps/gateway/tsconfig.json`

3. **Pass 14** → `apps/cli` (initial stub)
4. **Pass 21** → `apps/cli` (**replaces** Pass 14's `src/commands/*`)

5. **Pass 15** → `apps/dashboard` (initial scaffold)
6. **Pass 23** → `apps/dashboard` (**replaces** Pass 15's `src/store/*`, `src/components/*`)
   - **Patch 3.1**: Add `base: "./"` to `vite.config.ts`
   - **Patch 3.2**: Create `src/electron.d.ts` with `window.locoworker` declaration
   - **Patch 3.3**: Update `src/store/gatewaySlice.ts` — replace `__LOCOWORKER_GATEWAY_URL__` with `?gatewayUrl=` query param approach

7. **Pass 13** → `apps/desktop` (initial Electron scaffold)
8. **Pass 16** → `apps/desktop` (updates)
9. **Pass 24** → `apps/desktop` + `packages/keychain` (**replaces** Pass 13/16's ALL main process files)
   - **Patch 4.1**: Create `apps/desktop/build/entitlements.mac.plist`
   - **Patch 4.2**: Update `apps/desktop/src/main/index.ts` — use `?gatewayUrl=` in `loadURL`, NOT `executeJavaScript`
   - **Patch 4.3**: Ensure `LOCOWORKER_CORS_ORIGINS` is origin not full URL
   - **Patch 4.4**: Add `LOCOWORKER_SKIP_EMBEDDED_GATEWAY` dev guard

### Phase 4: Infrastructure (Passes 17-18)

1. **Pass 17** → test scaffolds
2. **Pass 18** → CI/CD workflows
   - **Patch 5.1**: Add `libsecret-1-dev` install step for Linux in `.github/workflows/ci.yml`

### Phase 5: Root patches (from pre-build checklist)

Apply these to root `turbo.json`:
- **Patch 6.1**: Add `@locoworker/desktop#build` override depending on `@locoworker/dashboard#build` + `@locoworker/gateway#build`

Apply these to root `tsconfig.json`:
- **Patch 6.2**: Add `{ "path": "packages/keychain" }` to references array

## Execution order for build commands

After all passes + patches applied:

```bash
# 1. Install dependencies
pnpm install

# 2. Rebuild native modules
pnpm --filter @locoworker/desktop rebuild keytar
pnpm --filter @locoworker/graphify rebuild tree-sitter
pnpm --filter @locoworker/memory rebuild better-sqlite3

# 3. Build in dependency order (turbo handles this automatically)
pnpm turbo run build

# 4. Verify
pnpm turbo run typecheck
