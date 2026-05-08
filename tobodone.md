Pass 16 — Normalization / Reconciliation pass (highly recommended)
Purpose: remove ambiguity and make “replay from markdown” deterministic.

This pass should output complete final versions (not snippets) of the files that currently cause drift, typically:

root package.json (scripts: remove stale cowork-cli / tauri refs if your generated reality is apps/cli + Electron)
turbo.json
workspace config (pnpm-workspace.yaml)
root TS configs if there are multiple competing ones
.github/workflows/* (so Pass 14 doesn’t “kinda overwrite” Pass 1—this pass makes the final workflows explicit)
update/replace FullScaffoldGeneration.md with an authoritative pass map that matches what actually exists (1–15 + your new 16+)
Think of Pass 16 as: “make the repo internally consistent and buildable with zero manual merging.”

Pass 17 — Spec-gap scaffolding pass
Purpose: add stubs + interfaces + wiring points for the spec items missing from Pass 1–15, without doing huge rewrites yet.

Examples (based on Completeproject.md expectations you mentioned earlier):

packages/tools-mcp (even if initially only a minimal client + tool registration skeleton)
packages/providers (provider registry abstraction; then core’s ProviderRouter can depend on it cleanly)
openclaw/ placeholder package (even if it’s just a README + package.json + interface contracts at first)
This pass is “create the missing folders/files so imports and planned architecture exist.”

Pass 18 — Spec implementation pass (incremental)
Purpose: actually implement one or more missing spec components end-to-end.

You can keep this pass narrowly scoped and still valuable, for example:

Implement tools-mcp fully + integrate it into ToolRegistry (most contained win)
Or implement providers registry + update ProviderRouter to use it
Or (bigger) do the desktop surface migration to Tauri if you truly want spec-first adherence
