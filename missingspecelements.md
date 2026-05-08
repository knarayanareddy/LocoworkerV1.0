Missing Spec Elements — Full Gap Analysis & Pass Plan
Based on the complete deep-dive of Completeproject.md and all 18 passes, here is the authoritative gap map followed by a proposed Pass 19–Pass 24 generation plan.

Part 1 — Gap Analysis
How I derived this
The spec (Completeproject.md) describes LocoWorker as a "21-package ecosystem" with specific platform capabilities. After 18 passes we have 19 library packages + 4 apps + 2 test suites = 25 workspace members. But several capabilities described in the spec are either:

A) A package that exists but is hollow (scaffold only, no real implementation)
B) A package that was never generated at all
C) A cross-cutting platform feature described in the spec that no single pass implemented end-to-end
D) A surface feature that the app shells reference but whose internals were never filled in
Gap Category A — Hollow packages (scaffold only, no real implementation)
Package	What exists	What's missing
packages/kairos	Store, TaskQueue, CronScheduler, TemporalContext types	KairosAgent loop body, TickEngine real scheduling, ObservationLog persistence, AutoDream scheduling hook, background task runner that actually calls queryLoop
packages/orchestrator	Store, AgentRegistry, AgentSpawner types, MessageRouter stubs	DelegationPlanner real logic, FileLockManager actual file locking, ResultAggregator with merge strategies, OrchestratorEngine turn-by-turn supervision loop
packages/autoresearch	Planner types, querier interface, evidence graph types	Real web/code/wiki/graph query dispatchers, evidence de-duplication + scoring, synthesis pipeline, Kairos scheduling hook, AutoResearchRunner full loop
packages/mirofish	Persona factory types, scenario builder	Docker orchestration real implementation, incident detection pattern engine, simulation runner that actually calls queryLoop, MiroFishAgent harness
packages/gateway	Auth manager, WS server, route stubs	Fastify server never fully wired — route handlers call stubs, SSE streaming from queryLoop not connected, admin routes not implemented
apps/desktop	Electron shell, IPC type contracts	IPC handlers in main process never implemented — ipcMain.handle() bodies for every ipcTypes.ts method are missing; embedded gateway startup logic absent
Gap Category B — Packages never generated at all
Package	Spec description	Priority
packages/keychain	OS keychain integration for storing API keys (mentioned in ADR-002 as deferred)	Medium
packages/context	Context assembly subsystem — TurnAssembler (Pass 2 has it but it's thin), prompt cache management, token budget ledger	High
packages/telemetry	Local-only, privacy-first telemetry: session stats, cost tracking export, performance metrics. No external calls	Low
packages/plugins	Plugin system described in spec — load third-party tool packages, custom providers, custom rules	Medium
Gap Category C — Cross-cutting platform features described in spec but never implemented end-to-end
Feature	Spec description	Gaps
Permission confirmation flow	Interactive "confirm before destructive action" prompt that the spec describes in detail — user sees the proposed action, types Y/N	PermissionGate in core has the tier checks but the interactive confirmation path (REQUIRE_CONFIRMATION_FOR_DESTRUCTIVE=true) is never wired to a real prompt in CLI or Gateway
CLAUDE.md / MEMORY.md injection	The spec describes a multi-phase context assembly: system + CLAUDE.md + rolling history + tool results. TurnAssembler (Pass 2) has the type but the actual file-reading + injection is not implemented	File reading + sanitisation + injection into every queryLoop call is missing
Session persistence across restarts	MemoryManager stores sessions, but SessionManager in core never calls into memory to restore a prior session. CLI starts fresh every time	The restore session path in SessionManager + CLI --session flag
AutoDream scheduling	AutoDream in packages/memory has the 5-phase pipeline defined but it's never triggered — Kairos was meant to schedule it via AUTODREAM_CRON	The cron job registration in Kairos that calls AutoDream.run()
Cost cap enforcement end-to-end	SessionCostTracker throws correctly (Pass 17 P1) but the queryLoop never catches it and surfaces it as a graceful "budget exhausted" stop	The catch block in queryLoop that converts CostCapExceededError to a terminal AgentEvent
Workspace boundary enforcement	PermissionGate has workspace boundary rules defined but tools-fs and tools-bash do not call it — they have their own path checks	Unified path validation that goes through PermissionGate rather than duplicate per-tool logic
queryLoop streaming to Gateway SSE	apps/gateway/src/bootstrap.ts (Pass 18 P1) wires everything up but the actual SSE route handler that calls queryLoop and pipes AgentEvents over the wire is never written	The /sessions/:id/stream SSE route handler body
queryLoop streaming to Desktop IPC	apps/desktop/src/ipcTypes.ts declares the contract but ipcMain.handle() bodies are empty	Desktop main process IPC handlers
Gap Category D — App shell internals never filled in
App	What exists	What's missing
apps/cli	Bootstrap + REPL entry point	Individual REPL command implementations: wiki, memory, graph, config, tools, sessions, research commands — all declared but bodies not written
apps/gateway	Bootstrap + Fastify server skeleton	All route handler bodies — /sessions, /sessions/:id/stream, /memory/*, /wiki/*, /graph/*, /kairos/*, /research/*, /admin/*
apps/dashboard	React component stubs	State management (Zustand/Jotai stores), API client (typed fetch wrapper for gateway), real chat rendering with tool-call cards
apps/desktop	Electron shell	ipcMain handler bodies, embedded gateway startup/shutdown, system tray integration, update checker stub
Part 2 — Prioritised Pass Plan (Pass 19 → Pass 24)
Based on the gap analysis, here is the proposed generation order. Each pass is independently buildable — later passes depend on earlier ones but nothing is a dead end.

text

Pass 19 — queryLoop completion + context injection + cost cap wiring
Pass 20 — Gateway route handlers (full SSE streaming + all REST routes)  
Pass 21 — CLI command implementations (all REPL commands complete)
Pass 22 — Kairos real engine + AutoDream scheduling + Orchestrator real engine
Pass 23 — Dashboard state management + API client + real UI components
Pass 24 — Desktop IPC handlers + embedded gateway + system tray
Pass 19 — queryLoop completion, context injection & cost cap wiring
Why first: Everything else (gateway SSE, CLI commands, desktop IPC) depends on a fully working queryLoop. This is the most foundational gap.

Files touched:

text

packages/core/src/
├── loop/
│   ├── queryLoop.ts           ← REPLACED (real async-generator, full event set)
│   ├── events.ts              ← NEW (all AgentEvent discriminated union types)
│   ├── turns.ts               ← NEW (TurnAssembler real implementation)
│   └── compaction.ts          ← NEW (AdaptiveCompactor real implementation)
├── context/
│   ├── ClaudeMdReader.ts      ← NEW (reads + sanitises CLAUDE.md / MEMORY.md)
│   ├── ContextAssembler.ts    ← NEW (4-layer context builder)
│   └── TokenBudget.ts         ← NEW (token estimation + budget ledger)
├── permissions/
│   ├── PermissionGate.ts      ← REPLACED (adds real interactive confirmation)
│   └── WorkspaceBoundary.ts   ← NEW (unified path validation used by all tools)
└── __tests__/
    ├── queryLoop.test.ts      ← NEW
    └── permissions.test.ts    ← NEW
Key deliverables:

queryLoop is a real async generator emitting AgentEvent stream
TurnAssembler reads CLAUDE.md + MEMORY.md from disk and injects them
Cost cap thrown by SessionCostTracker is caught and emitted as { type: 'budget_exhausted' } event (graceful stop, not crash)
PermissionGate calls into CLI/Gateway for interactive confirm when tier requires it
WorkspaceBoundary.assertSafe(path) is the single choke-point — tools-fs and tools-bash both call it
Pass 20 — Gateway route handlers (full SSE streaming + all REST routes)
Why second: The gateway is the primary API surface for Dashboard and Desktop. Once queryLoop works (Pass 19), routing its events over SSE is straightforward.

Files touched:

text

apps/gateway/src/
├── routes/
│   ├── sessions.ts            ← NEW (POST /sessions, GET /sessions/:id, DELETE)
│   ├── stream.ts              ← NEW (GET /sessions/:id/stream — SSE queryLoop)
│   ├── memory.ts              ← NEW (GET/POST /memory/*)
│   ├── wiki.ts                ← NEW (full CRUD /wiki/*)
│   ├── graph.ts               ← NEW (GET /graph/symbols, /graph/search)
│   ├── kairos.ts              ← NEW (GET/POST /kairos/tasks)
│   ├── research.ts            ← NEW (POST /research, GET /research/:id)
│   ├── tools.ts               ← NEW (GET /tools, POST /tools/:name/execute)
│   ├── health.ts              ← NEW (GET /health — uses GatewayStack.healthSummary())
│   └── admin.ts               ← NEW (GET /admin/providers, /admin/mcp)
├── server.ts                  ← REPLACED (wires all routes, middleware, WS)
└── __tests__/
    └── routes.test.ts         ← NEW (integration tests per route)
Key deliverables:

GET /sessions/:id/stream opens an SSE connection, calls queryLoop, and pipes every AgentEvent as data: <json>\n\n
All CRUD routes for memory/wiki/graph/kairos/research are complete
/health returns the GatewayHealthSummary from bootstrap
WebSocket session for real-time tool progress (mirrors SSE but bidirectional)
Pass 21 — CLI command implementations
Why third: The CLI REPL was bootstrapped in Pass 12/18 but every command body is // TODO. With queryLoop working (Pass 19) and the stack fully wired, commands are straightforward.

Files touched:

text

apps/cli/src/
├── commands/
│   ├── query.ts               ← NEW (main "send a message" command — streams queryLoop to terminal)
│   ├── wiki.ts                ← NEW (wiki list/search/show/create/edit)
│   ├── memory.ts              ← NEW (memory show/search/clear)
│   ├── graph.ts               ← NEW (graph scan/search/stats)
│   ├── config.ts              ← NEW (config get/set/list)
│   ├── tools.ts               ← NEW (tools list/call/mcp-status)
│   ├── sessions.ts            ← NEW (sessions list/resume/delete)
│   ├── research.ts            ← NEW (research start/status/list)
│   └── index.ts               ← NEW (command registry + dispatch)
├── output/
│   ├── Renderer.ts            ← NEW (ANSI-formatted AgentEvent stream renderer)
│   ├── ToolCallCard.ts        ← NEW (tool call progress display)
│   └── CostMeter.ts           ← NEW (inline budget display)
├── repl.ts                    ← REPLACED (wires commands + renderer + history)
└── __tests__/
    └── commands.test.ts       ← NEW
Key deliverables:

locoworker REPL fully interactive: type a message, see streamed response with tool calls rendered inline
All slash commands (/wiki, /memory, /graph, /config, /tools, /sessions) work
Cost meter shown after each turn
--session <id> flag to resume a prior session
Pass 22 — Kairos real engine + AutoDream scheduling + Orchestrator real engine
Why fourth: These are the "autonomy" packages. They depend on queryLoop (Pass 19) being solid. Kairos schedules AutoDream and Orchestrator coordinates multi-agent runs.

Files touched:

text

packages/kairos/src/
├── engine/
│   ├── TickEngine.ts          ← REPLACED (real setInterval loop)
│   ├── TaskRunner.ts          ← NEW (picks tasks from queue, calls queryLoop)
│   ├── ObservationLog.ts      ← REPLACED (real SQLite write + query)
│   └── AutoDreamScheduler.ts  ← NEW (registers AutoDream cron via CronScheduler)
└── __tests__/
    └── engine.test.ts         ← NEW

packages/memory/src/
└── autodream/
    ├── AutoDream.ts           ← REPLACED (5-phase pipeline actually implemented)
    └── __tests__/
        └── autodream.test.ts  ← NEW

packages/orchestrator/src/
├── engine/
│   ├── OrchestratorEngine.ts  ← REPLACED (real supervision loop)
│   ├── DelegationPlanner.ts   ← REPLACED (real task decomposition)
│   ├── FileLockManager.ts     ← REPLACED (real file locking via SQLite)
│   └── ResultAggregator.ts    ← REPLACED (real merge strategies)
└── __tests__/
    └── engine.test.ts         ← NEW
Key deliverables:

KairosAgent calls queryLoop for scheduled background tasks
AutoDream is registered as a nightly cron job in Kairos (respects AUTODREAM_CRON env)
OrchestratorEngine can spawn sub-agents (each runs its own queryLoop) and collect results
FileLockManager uses SQLite WAL advisory locks so parallel agents don't conflict on files
Pass 23 — Dashboard state management + API client + real UI components
Why fifth: Dashboard is a frontend — it can be developed once the gateway routes exist (Pass 20). It's isolated from backend packages.

Files touched:

text

apps/dashboard/src/
├── api/
│   ├── client.ts              ← NEW (typed fetch wrapper for all gateway endpoints)
│   ├── sse.ts                 ← NEW (SSE hook — parses AgentEvent stream)
│   └── types.ts               ← NEW (shared request/response types)
├── stores/
│   ├── session.store.ts       ← NEW (Zustand session + message store)
│   ├── provider.store.ts      ← NEW (provider health + cost summary)
│   └── tools.store.ts         ← NEW (tools list + MCP server status)
├── components/
│   ├── Chat/
│   │   ├── ChatWindow.tsx     ← REPLACED (real streaming render)
│   │   ├── MessageBubble.tsx  ← REPLACED (user + assistant variants)
│   │   ├── ToolCallCard.tsx   ← REPLACED (real tool call progress)
│   │   └── CostBadge.tsx      ← NEW (per-turn cost display)
│   ├── Sidebar/
│   │   ├── SessionList.tsx    ← NEW
│   │   ├── ProviderStatus.tsx ← NEW
│   │   └── McpStatus.tsx      ← NEW
│   └── Settings/
│       ├── ProviderSettings.tsx ← NEW
│       └── WorkspaceSettings.tsx ← NEW
├── pages/
│   ├── ChatPage.tsx           ← REPLACED (wired to real stores + API)
│   ├── SettingsPage.tsx       ← NEW
│   └── ResearchPage.tsx       ← NEW
└── __tests__/
    └── components.test.tsx    ← NEW
Key deliverables:

Dashboard connects to the running gateway via SSE and renders streamed AgentEvents live
Tool call cards show tool name, args, spinner while running, output when done
Cost badge updates in real time from the SSE stream
Session list persists across page reloads (localStorage + gateway sync)
Pass 24 — Desktop IPC handlers + embedded gateway + system tray
Why last: Desktop builds on top of everything — it embeds the gateway (Pass 20), renders the dashboard (Pass 23), and adds OS-native features.

Files touched:

text

apps/desktop/src/
├── main/
│   ├── index.ts               ← REPLACED (real BrowserWindow + lifecycle)
│   ├── ipcHandlers.ts         ← NEW (all ipcMain.handle() bodies for ipcTypes.ts)
│   ├── EmbeddedGateway.ts     ← NEW (spawns apps/gateway as child process)
│   ├── SystemTray.ts          ← NEW (tray icon + context menu)
│   └── AutoUpdater.ts         ← NEW (electron-updater stub)
├── preload/
│   └── index.ts               ← REPLACED (exposes all IPC methods from ipcTypes.ts)
└── __tests__/
    └── ipc.test.ts            ← NEW

packages/keychain/                ← NEW package
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts
    ├── KeychainManager.ts     ← NEW (OS keychain via keytar with in-memory fallback)
    └── __tests__/
        └── keychain.test.ts   ← NEW
Key deliverables:

Every method in ipcTypes.ts has a real ipcMain.handle() implementation
The desktop app starts an embedded gateway process on DESKTOP_EMBEDDED_GATEWAY_PORT and shuts it down cleanly on app quit
System tray shows status (idle / running / error) and quick actions
packages/keychain provides secure API key storage — desktop uses it; CLI and gateway use env vars
Part 3 — Summary table (what to generate next)
Pass	Title	Blocker	Est. new files	New packages
19	queryLoop completion + context injection + cost cap	None (builds on Pass 18)	~12	None
20	Gateway route handlers + SSE streaming	Pass 19	~12	None
21	CLI command implementations	Pass 19	~14	None
22	Kairos engine + AutoDream + Orchestrator engine	Pass 19	~12	None
23	Dashboard state + API client + real components	Pass 20	~18	None
24	Desktop IPC + embedded gateway + packages/keychain	Pass 20 + 23	~12	packages/keychain
Total: ~80 new/replaced files across 6 passes, 1 new package.

Recommended order to generate
text

Pass 19 first — it unblocks everything else
Pass 20 + Pass 21 can be generated in parallel (no dependency between them)
Pass 22 also unblocked after Pass 19
Pass 23 depends on Pass 20 (needs gateway routes)
Pass 24 depends on Pass 20 + Pass 23 (needs gateway + dashboard)
