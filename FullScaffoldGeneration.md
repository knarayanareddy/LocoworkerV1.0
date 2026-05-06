Generate the complete file tree with every file, one package at a time:

Pass 1: Root scaffold (package.json, pnpm-workspace, turbo.json, tsconfig.base)
Pass 2: packages/core (all files, fully implemented)
Pass 3: packages/memory
Pass 4: packages/graphify
Pass 5: packages/wiki
Pass 6: packages/kairos
Pass 7: packages/orchestrator
Pass 8: packages/gateway
Pass 9: packages/autoresearch
Pass 10: packages/mirofish
Pass 11: packages/security
Pass 12: packages/tools-* (6 packages)
Pass 13: packages/shared
Pass 14: apps/cowork-cli
Pass 15: apps/dashboard
Pass 16: apps/desktop (React + Tauri)
Pass 17: Test suites
Pass 18: CI/CD + .github/workflows
