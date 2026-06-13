# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **source tree of Claude Code itself** — the interactive CLI agent (TypeScript + React/Ink TUI). It is an *extracted source-only* tree: there is **no `package.json`, no `tsconfig.json`, no build config, and no tests** checked in. Treat it as a reference/reading codebase. Do not invent `npm run build`/`npm test` commands — none exist here.

Conventions to know before editing:
- **ESM with `.js` import specifiers** on TypeScript files (`import { x } from './foo.js'` resolves `foo.ts`). Keep the `.js` suffix.
- **Bun bundler feature gates**: `import { feature } from 'bun:bundle'` plus `process.env.*` checks are used for compile-time dead-code elimination. Code guarded by gates like `USER_TYPE === 'ant'`, `COORDINATOR_MODE`, `CONTEXT_COLLAPSE`, `WORKFLOW_SCRIPTS`, `KAIROS`, `AGENT_TRIGGERS` is stripped from external builds. When a feature seems to appear and disappear, it is gated.
- **Lazy imports for startup latency**: heavy modules (React/Ink, OpenTelemetry) are deferred. See the comments at the top of `src/main.tsx`.

## Startup flow (entry → REPL)

1. **`src/main.tsx`** — process entry. Fires parallel I/O prefetch (MDM read, keychain) *before* heavy imports, then calls `init()`, populates bootstrap state, builds the tool/skill/plugin registries, and calls `launchRepl()`.
2. **`src/entrypoints/cli.tsx`** — Commander CLI definition and fast-path commands that bypass full module load (`--version`, `--dump-system-prompt`, `--daemon-worker=<kind>`, specialized MCP servers).
3. **`src/entrypoints/init.ts`** — memoized deferred init: `enableConfigs()`, auth/OAuth, trust dialog, telemetry, remote-managed settings + policy limits (kicked off async).
4. **`src/bootstrap/state.ts`** — **global singleton** for cross-module session config that must be set before/around module load: `originalCwd`, `sessionId`, model overrides, telemetry providers, cumulative `modelUsage`, `invokedSkills`, permission-mode transitions. Distinct from the reactive UI state (below).
5. **`src/replLauncher.tsx`** — async lazy-loader that renders `<App><REPL/></App>` into the Ink loop. App lives in `src/components/App.tsx`, the REPL screen in `src/screens/REPL.tsx`.

Two run modes share the core loop: interactive **REPL mode** and headless **SDK mode** (`src/QueryEngine.ts`, holds `mutableMessages` and drives `query()`). Entry points for SDK/MCP are under `src/entrypoints/sdk/` and `src/entrypoints/mcp.ts`.

## Core agent loop

**`src/query.ts`** is the heart: an async generator (`query()` → inner `queryLoop()`) that runs one turn per iteration. Per iteration:

1. Prepare messages: apply compaction (snip → microcompact → context-collapse → autocompact), assemble system prompt (`prependUserContext` / `appendSystemContext`).
2. Stream from the model via the injected `deps.callModel`; yield assistant messages to the UI, collect `tool_use` blocks. Streaming tool execution can overlap with the stream when gated on.
3. Execute tools (`runTools` in `src/services/tools/toolOrchestration.ts`), gated by the `canUseTool` permission check; wrap `tool_result`s into the next user message.
4. Run **stop hooks** (`src/query/stopHooks.ts`) — including memory extraction — and a **token-budget** check (`src/query/tokenBudget.ts`) that can auto-continue.
5. Rebuild state and loop, or return a `Terminal` reason (`completed`, `aborted_*`, `max_turns_reached`, …).

Supporting files: `src/query/config.ts` (immutable per-query snapshot), `src/query/deps.ts` (injected I/O: `callModel`, `microcompact`, `autocompact`, `uuid`). Recovery paths handle `prompt_too_long` (413), `max_output_tokens`, and oversized media.

## Tools system

- **`src/Tool.ts`** — the `Tool` interface and `buildTool()` factory (applies safe defaults). Tools carry `name`, `description`, `inputSchema`, `call()`, render methods, and capability predicates (`isReadOnly`, `isConcurrencySafe`, `checkPermissions`, …). Also defines `ToolUseContext` (the per-call context: `agentId`, `toolPermissionContext`, file-state cache, app-state setter).
- **`src/tools.ts`** — registry assembly: `getAllBaseTools()` (feature-gated full list) → `getTools(permissionContext)` (filters by permission/deny rules, plan mode, REPL mode) → `assembleToolPool()` (merges built-ins + MCP tools, dedupes, sorts by name for prompt-cache stability).
- Tools live one-per-directory under `src/tools/` (e.g. `BashTool/`, `FileEditTool/`, `AgentTool/`, `SkillTool/`, `EnterPlanModeTool/`). MCP tools are surfaced through `MCPTool`. `src/Task.ts` + `src/tasks.ts` define the background-task abstraction (types: `local_agent`, `remote_agent`, `in_process_teammate`, `local_bash`, `local_workflow`, `dream`, …), with implementations under `src/tasks/`.

## Permission & policy system

The authorization layer that decides whether each tool call may run. It is the `canUseTool` gate invoked from the query loop before every tool execution.

**Permission context & modes.** `ToolPermissionContext` (`src/Tool.ts`, shape in `src/types/permissions.ts`) carries the current `mode`, the rule sets `alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules` (each keyed by source), `additionalWorkingDirectories`, `prePlanMode` (saved when entering plan mode), and flags like `isBypassPermissionsModeAvailable`. Modes (`src/utils/permissions/PermissionMode.ts`): **`default`** (ask on new tool uses), **`plan`** (read-only; defer execution), **`acceptEdits`** (auto-allow safe edits/reads in the working dir), **`bypassPermissions`** (allow all except safety-immune paths), plus internal **`auto`** (classifier-decided, gated) and **`dontAsk`** (turn asks into denies).

**Decision flow.** `useCanUseTool` (`src/hooks/useCanUseTool.tsx`) → `hasPermissionsToUseTool` in `src/utils/permissions/permissions.ts`. Evaluation order:
1. tool-wide **deny** rule → deny; 2. tool-wide **ask** rule → ask;
3. the tool's own **`checkPermissions(input, context)`** (e.g. `BashTool` prefix/AST analysis in `src/tools/BashTool/bashPermissions.ts`; path safety in `src/utils/permissions/filesystem.ts` + `pathValidation.ts`);
4. content-specific **ask** rules (these even override bypass); 5. **safety-immune paths** (`.git/`, `.claude/`, shell configs) always prompt even under bypass;
6. **mode bypass** → allow; 7. tool-wide **allow** rule → allow; 8. `passthrough` → ask; 9. post-process for `auto`/`dontAsk` (classifiers in `bashClassifier.ts`, `yoloClassifier.ts`, `classifierDecision.ts`).

**Rules.** String form `Tool` (tool-wide), `Tool(content)` (exact), `Tool(prefix:*)` / `Tool(pattern *)` (wildcard), e.g. `Bash(git commit:*)`, `Read(/etc/**)`, `WebFetch(domain:example.com)`. Parsed/serialized by `permissionRuleParser.ts`; matched by `shellRuleMatching.ts`. **Sources** (precedence/scoping): `policySettings` (org, read-only) > `flagSettings`/`cliArg` > `localSettings` (`.claude/settings.local.json`) > `projectSettings` (`.claude/settings.json`) > `userSettings` (`~/.claude/settings.json`) > `session` (ephemeral). Loaded by `permissionsLoader.ts`.

**Policy layer.** `src/services/policyLimits/` and `src/services/remoteManagedSettings/` fetch org-managed restrictions (ETag-cached, fail-open, hourly refresh) that block features/tools globally and can disable `bypassPermissions` regardless of user settings (`bypassPermissionsKillswitch.ts`).

**UI & persistence.** Tool-specific prompt UIs live in `src/components/permissions/` (`BashPermissionRequest/`, `FileEditPermissionRequest/`, …) dispatched from `PermissionRequest.tsx`; the request/response queue is managed in `src/hooks/toolPermission/PermissionContext.ts`. Choosing "always allow" calls `src/utils/permissions/PermissionUpdate.ts` (`addRules`/`setMode`/`addDirectories`), which persists the rule back to the chosen settings scope and updates the live `ToolPermissionContext`.

## Key subsystems (the modules you asked about)

### Subagent — `src/tools/AgentTool/`
The `Agent` tool spawns subagents. `loadAgentsDir.ts` discovers definitions: **built-in** (`built-in/` → general-purpose, Explore, Plan, verification, claude-code-guide) and **custom** from `.claude/agents/*.md` (frontmatter: `whenToUse`, `tools`, `disallowedTools`, `model`, `hooks`, `mcpServers`, `isolation`, …). `runAgent.ts` creates an **isolated `ToolUseContext`** (own `agentId`, scoped permission mode, cloned file-state) and recursively calls `query()`. Subagents run sync (blocking the parent turn) or as background tasks (`LocalAgentTask`/`RemoteAgentTask`). `isolation: "worktree"` runs them in a fresh git worktree.

### Plan mode — `src/tools/EnterPlanModeTool/`, `ExitPlanModeTool/`
`EnterPlanModeTool` sets the permission mode to `plan` (saving `prePlanMode`), which makes `getTools()` expose only read-only tools. `ExitPlanModeTool` (`ExitPlanModeV2Tool`) stores the plan markdown, optionally requests semantic permissions for the work ahead, restores the prior mode, and flags a plan-review attachment for the next turn. Config/experiments in `src/utils/planModeV2.ts`; mode state lives in `ToolPermissionContext` (`src/Tool.ts`) and `src/bootstrap/state.ts`.

### Skill — `src/tools/SkillTool/` + `src/skills/`
Skills are markdown prompts (frontmatter: `description`, `whenToUse`, `arguments`, `model`, `effort`, `hooks`). `src/skills/loadSkillsDir.ts` discovers them from `.claude/skills/` (project), `~/.claude/skills/` (user), bundled skills (`src/skills/bundled*`), plugins, and MCP servers, deduping by realpath. `SkillTool.call()` substitutes args into the skill prompt and runs it via `runAgent()` (a forked subagent). Skills not in the active pool are discoverable through `ToolSearchTool`.

### Compact — `src/services/compact/`
Three coexisting context-management strategies, applied in `query.ts` in order:
- **Snip** (`HISTORY_SNIP`, gated) — rule-based message removal; reports `tokensFreed` so autocompact doesn't double-count.
- **Microcompact** (`microCompact.ts`) — surgical, time-based clearing of *old tool-result blocks* (file reads, shell output, grep/glob, web). No summarization round; uses placeholders or cache_edits for prompt caching.
- **Autocompact** (`autoCompact.ts` + `compact.ts`) — proactive full-conversation summarization when tokens exceed `effectiveContextWindow − AUTOCOMPACT_BUFFER_TOKENS` (~13K). Calls the model to produce a dense summary, inserts a `SystemCompactBoundaryMessage`, and has a 3-failure circuit breaker. **Context-collapse** (`CONTEXT_COLLAPSE`, gated) replaces autocompact when on.

The **compact boundary** is a `system`/`compact_boundary` message; `getMessagesAfterCompactBoundary()` (`src/utils/messages.ts`) returns everything from the last boundary onward. `postCompactCleanup.ts` re-injects truncated skill docs / file attachments / MCP instructions.

### Automemory — `src/memdir/` + `src/services/extractMemories/`, `SessionMemory/`, `teamMemorySync/`
File-based persistent memory at `~/.claude/projects/<cwd-hash>/memory/*.md` (and `~/.claude/team-memory/` for team scope). Each memory is one `.md` file with frontmatter (`type: user|feedback|project|reference`, `description`) plus a `MEMORY.md` index (capped ~200 lines / 25KB).
- **Extraction**: fired from the **stop hook** (`extractMemories.ts`, `EXTRACT_MEMORIES` gate), main-thread only, throttled by token/tool-call thresholds (init ~10K tokens; update ~5K tokens or 3 tool calls). Runs a forked agent that writes/updates memory files.
- **Recall**: `startRelevantMemoryPrefetch()` (`src/query.ts` → `src/utils/attachments.ts`) fires at query start, asks a model to select up to ~5 relevant memories from the manifest, and injects them as a `relevant_memories` system attachment (avoiding re-surfacing).
- **Session memory** (`src/services/SessionMemory/`) is a separate, within-session running summary, distinct from durable memories.

### Hooks — `src/utils/hooks.ts` (engine), `src/types/hooks.ts` (types)
The **user-configurable hook system** (settings.json), not React hooks. Events include `PreToolUse`, `PostToolUse`, `SessionStart/End`, `Stop`, `UserPromptSubmit`, `PreCompact/PostCompact`, `SubagentStart/Stop`, `PermissionRequest`, `TeammateIdle`, etc. Hooks are shell commands, HTTP, or in-process callbacks; they run in parallel with per-hook timeouts and can block, approve/deny, or mutate tool input/output. Loaded per scope by `src/utils/hooks/hooksConfigManager.ts`; tool hooks executed via `src/services/tools/toolHooks.ts` inside the query loop. Plugin-contributed hooks are converted by `src/utils/plugins/loadPluginHooks.ts`. (Note: `src/hooks/` is mostly **React** `useXxx` hooks — e.g. `useCanUseTool.ts` for the permission gate — a different thing from the hook *system*.)

### Plugins — `src/plugins/` + `src/services/plugins/` + `src/utils/plugins/`
Plugins contribute **commands, agents, skills, hooks, MCP servers, LSP servers, output styles** via a `plugin.json` manifest. Built-in plugins are registered at startup (`src/plugins/builtinPlugins.ts`, `initBuiltinPlugins()`); installable plugins resolve through marketplaces (`{name}@{marketplace}` ids) and are enabled per scope via `enabledPlugins` in settings. Loading/hot-reload in `src/utils/plugins/pluginLoader.ts`; policy allow/deny via `pluginPolicy.ts`.

### Agent team / coordinator — `src/coordinator/`, `src/tools/TeamCreateTool/`, `SendMessageTool/`, `src/tasks/InProcessTeammateTask/`
- **Teams**: multiple agents ("teammates") run in-process (via `AsyncLocalStorage`, `src/utils/teammates/`) or as tmux sessions, coordinated through a **file-based mailbox** (`src/utils/teammateMailbox.ts`, `~/.claude/teams/<team>/inboxes/<agent>.json`). `SendMessageTool` delivers messages/broadcasts and structured shutdown requests; teammate state is tracked as `TaskState` in AppState. `src/services/teamMemorySync/` syncs team memory files with the server.
- **Coordinator mode** (`src/coordinator/coordinatorMode.ts`, `COORDINATOR_MODE` gate): the main agent becomes a coordinator that delegates to worker agents (via `AgentTool`) and synthesizes results, using a specialized system prompt + user context. Workers report back as `<task-notification>` messages.

## State management

- **`src/state/store.ts`** — generic immutable store (`getState`/`setState`/`subscribe`).
- **`src/state/AppStateStore.ts`** — the `AppState` shape (settings, model, `toolPermissionContext`, `mcp` clients/tools, `tasks`, `messages`, fileHistory, teammates, …) and defaults.
- **`src/state/AppState.tsx`** — React context provider + `useAppState(selector)` / `useSetAppState()` hooks.
- Keep **bootstrap state** (`src/bootstrap/state.ts`, global singleton) and **AppState** (reactive UI store) mentally separate — different lifetimes and purposes.

## UI / terminal rendering (TUI)

The interface is a React app rendered to the terminal via a **vendored fork of Ink** (React-for-terminal) under `src/ink/` — not the npm `ink` package.

**Rendering foundation.** `src/ink/ink.tsx` is the engine: a React reconciler driving a render loop with TTY I/O, keyboard/mouse parsing (`events/`, `hit-test.ts`), focus management (`focus.ts`), flexbox layout (Yoga, `layout/` + `src/native-ts/`), a cell-grid output buffer (`output.ts`, `screen.ts`), and incremental diff output (`log-update.ts`) that emits only changed rows, frame-throttled. `src/ink.ts` is the thin public wrapper exporting `Box`/`Text`/`render`/`createRoot` wrapped in a `ThemeProvider`.

**Composition.** `src/components/App.tsx` nests the top-level providers (FPS metrics → `StatsProvider` → `AppStateProvider`). The main interactive screen is `src/screens/REPL.tsx` (large, stateful: messages, input, loading, queues, query orchestration). Layout is handled by `src/components/FullscreenLayout.tsx`, which exposes slots: **scrollable** (transcript), **overlay** (permission/elicitation dialogs floating over the transcript), **modal** (slash-command dialogs), **bottom** (spinner + input footer), and **bottomFloat**.
- **Transcript**: `src/components/Messages.tsx` (virtualized list) → `Message.tsx` per row, rendering streaming markdown, thinking blocks, and tool-use/result blocks.
- **Input**: `src/components/PromptInput/` — a multi-mode editor (`prompt`/`search`/`vim`/`bash`/`file`, see `inputModes.ts`) with multi-line editing, paste handling (`inputPaste.ts`), and footer completions (`PromptInputFooterSuggestions.tsx`).
- **Footer/status**: `src/components/StatusLine.tsx` (tokens, cost, model, effort, branch); `Spinner` while streaming; `expandedView` (`tasks`/`teammates`, from AppState) swaps the transcript for task/teammate panels.

**Contexts** (`src/context/`): `modalContext` (modal-relative sizing), `overlayContext` (Escape-key/overlay coordination), `promptOverlayContext` (queued prompt dialogs), `notifications` (toasts), `mailbox` (teammate messages), `QueuedMessageContext` (input drained mid-turn), `stats` (cost/tokens), `voice`.

**Input handling.** Keybindings resolve through `src/keybindings/` (`KeybindingProviderSetup.tsx` provider; `defaultBindings.ts` merged with `~/.claude/keybindings.json` via `loadUserBindings.ts`; `resolver.ts`/`match.ts` including chords). Vim editing is a state machine in `src/vim/` (`motions.ts`, `operators.ts`, `textObjects.ts`, `transitions.ts`) wired into the prompt input. Early keystrokes are captured pre-render (`src/utils/earlyInput.ts`).

**Dialogs & one-off screens.** `src/interactiveHelpers.tsx` provides `showDialog`/`showSetupDialog`/`renderAndRun`/`exitWithMessage` (render an element to an Ink root, await result, unmount). `src/dialogLaunchers.tsx` holds async-imported launchers (resume chooser, settings-error dialog, snapshot merge, …) wrapped with the App + keybinding providers so they have full context.

## Directory map (top level of `src/`)

Loose files at `src/` root are the spine: `main.tsx` (entry), `query.ts` / `QueryEngine.ts` (loop), `Tool.ts` / `tools.ts` (tool registry), `Task.ts` / `tasks.ts` (task registry), `commands.ts` (slash-command registry), `context.ts` (system/user prompt context), `history.ts`, `cost-tracker.ts` / `costHook.ts`, `setup.ts`, `interactiveHelpers.tsx` / `dialogLaunchers.tsx` (Ink render helpers), `replLauncher.tsx`.

The seven large directories below are expanded to their second level (file counts are approximate scale indicators). Read the named core files first when entering a subsystem.

### `tools/` (~184 files) — one directory per tool
Each tool dir holds the `*Tool.ts(x)` definition + permission/render helpers. Grouped by purpose:
- **File ops**: `FileReadTool/`, `FileWriteTool/`, `FileEditTool/`, `NotebookEditTool/`, `GlobTool/`, `GrepTool/`
- **Execution**: `BashTool/` (+ `bashPermissions.ts`), `PowerShellTool/`, `REPLTool/`, `SleepTool/`
- **Agents/tasks**: `AgentTool/` (subagent engine: `runAgent.ts`, `loadAgentsDir.ts`, `built-in/`), `TaskCreateTool/` … `TaskStopTool/`, `SkillTool/`
- **Plan/worktree**: `EnterPlanModeTool/`, `ExitPlanModeTool/`, `EnterWorktreeTool/`, `ExitWorktreeTool/`
- **Teams/coordination**: `TeamCreateTool/`, `TeamDeleteTool/`, `SendMessageTool/`, `AskUserQuestionTool/`, `SyntheticOutputTool/`
- **MCP/web/search**: `MCPTool/`, `ListMcpResourcesTool/`, `ReadMcpResourceTool/`, `McpAuthTool/`, `WebFetchTool/`, `WebSearchTool/`, `LSPTool/`, `ToolSearchTool/`
- **Scheduling/remote**: `ScheduleCronTool/`, `RemoteTriggerTool/`, plus `ConfigTool/`, `BriefTool/`, `TodoWriteTool/`. `shared/` = cross-tool helpers; `testing/` = test scaffolding.

### `services/` (~130 files) — non-UI subsystems
- **Context mgmt**: `compact/` (autocompact/microcompact/boundary), `extractMemories/`, `SessionMemory/`, `toolUseSummary/`, `AgentSummary/`, `awaySummary.ts`
- **Model/API**: `api/` (`client.ts`, `claude.ts`, `bootstrap.ts`, `withRetry.ts`, `errors.ts`, usage/quota), `tokenEstimation.ts`
- **Integrations**: `mcp/` (server lifecycle), `lsp/`, `oauth/`, `plugins/` (install/enable ops), `teamMemorySync/`, `settingsSync/`, `remoteManagedSettings/`, `policyLimits/`
- **Misc**: `analytics/`, `autoDream/`, `MagicDocs/`, `PromptSuggestion/`, `tips/`, `tools/` (tool orchestration: `toolOrchestration.ts`, `toolHooks.ts`), `notifier.ts`, `diagnosticTracking.ts`

### `utils/` (~564 files) — the shared-logic layer where most subsystems actually live
- **Agent loop support**: `messages/` (`mappers.ts`, `systemInit.ts`), `model/` (model resolution), `processUserInput/` (slash-command/@mention expansion pipeline), `background/`, `task/`, `todo/`, `suggestions/`, `attachments.ts`, `hooks.ts` (the **hook-system engine**)
- **Permissions**: `permissions/` (rule engine — `permissions.ts`, `permissionRuleParser.ts`, `shellRuleMatching.ts`, `PermissionMode.ts`, classifiers; see *Permission & policy system*)
- **Execution & shell**: `bash/` (AST/security parse), `shell/` (snapshot, env), `powershell/`, `sandbox/` (seatbelt), `git/`, `github/`
- **Multi-agent**: `swarm/` (team files, backends, in-process spawn), `teammate*`
- **Config & platform**: `settings/` (settings.json scopes, MDM), `secureStorage/` (keychain), `plugins/` (loader, marketplace, policy), `skills/`, `mcp/`, `memory/`, `telemetry/`, `nativeInstaller/`, `filePersistence/` (file history/rewind), `teleport/`, `deepLink/`, `dxt/`, `computerUse/`, `claudeInChrome/`, `ultraplan/` (plan mode v2)

### `commands/` (~207 files) — slash commands, one file/dir per command
Each exports a command definition consumed by `commands.ts`. Spans session (`clear`, `compact/`, `resume`, `rewind`, `export`), config (`config/`, `model`, `effort`, `permissions/`, `hooks/`, `theme`, `vim`), git/PR (`commit.ts`, `commit-push-pr.ts`, `review.ts`, `pr_comments/`, `branch/`), agents/teams (`agents/`, `tasks/`, `teleport/`), plugins/skills/mcp (`plugin/`, `skills/`, `mcp/`), and auth/account (`login/`, `logout/`, `usage/`, `cost/`).

### `components/` (~389 files) — Ink TUI components (see *UI / terminal rendering*)
- **Core REPL shell**: `App.tsx`, `FullscreenLayout.tsx`, `Messages.tsx`, `StatusLine.tsx`, `Spinner/`, `PromptInput/` (multi-mode editor)
- **Rendering**: `messages/`, `diff/` + `StructuredDiff/`, `HighlightedCode/`, `design-system/` + `ui/` (primitives), `CustomSelect/`, `LogoV2/`
- **Domain panels/dialogs**: `permissions/` (per-tool prompt UIs), `agents/`, `teams/`, `tasks/`, `skills/`, `mcp/`, `memory/`, `Settings/`, `TrustDialog/`, `sandbox/`, `shell/`, `hooks/`, `wizard/`, `HelpV2/`

### `hooks/` (~104 files) — React hooks (distinct from the hook *system* in `utils/hooks.ts`)
Mostly loose `useXxx.ts` hooks consumed by components. `useCanUseTool.tsx` + `toolPermission/` = the permission gate; `notifs/` = notification hooks.

### `ink/` (~96 files) — vendored Ink renderer (React-for-terminal)
`ink.tsx` = the reconciler/render-loop engine. `components/` (Box/Text/ScrollBox…), `layout/` (Yoga flexbox), `events/` (keyboard/mouse parsing), `termio/` (terminal I/O), `hooks/`. The thin public wrapper is `src/ink.ts`.

### Smaller top-level directories
- `entrypoints/` — `cli.tsx`, `init.ts`, `mcp.ts`, `sdk/` (headless SDK types/schemas), `sandboxTypes.ts`.
- `query/` — loop helpers: `config.ts`, `deps.ts`, `stopHooks.ts`, `tokenBudget.ts`.
- `tasks/` — task runners: `LocalAgentTask/`, `RemoteAgentTask/`, `InProcessTeammateTask/`, `LocalShellTask/`, `DreamTask/`.
- `state/`, `bootstrap/` — reactive store (`AppState`) and global session singleton (`bootstrap/state.ts`).
- `context/` — React contexts (`modalContext`, `overlayContext`, `notifications`, `mailbox`, `stats`, …).
- `skills/`, `plugins/`, `coordinator/`, `memdir/`, `keybindings/`, `vim/`, `voice/`, `outputStyles/` — named subsystems documented above / self-describing.
- `bridge/`, `remote/`, `server/`, `upstreamproxy/` — remote/bridged session transport and proxying.
- `screens/` — top-level screens (`REPL.tsx`, `Doctor.tsx`, `ResumeConversation.tsx`).
- `cli/`, `buddy/`, `moreright/`, `assistant/` — CLI plumbing and assorted helpers.
- `migrations/` — one-off config/model migrations run at startup.
- `schemas/`, `types/`, `constants/`, `native-ts/` — shared schemas, types, constants, and native (Yoga) bindings.
- `migrations/` — one-off config/model migrations run at startup.
- `schemas/`, `types/`, `constants/` — shared schemas, types, constants.

## Working in this codebase

- To understand a feature, start from its **tool** (`src/tools/<X>Tool/`) or **command** (`src/commands/<x>`), then follow into `src/services/` and `src/utils/` where the logic usually lives.
- The query loop (`src/query.ts`) is the spine: most subsystems (compaction, memory, hooks, tool execution, token budget) are invoked from there in a fixed order — read it first to see how a turn is orchestrated.
- When something is feature-gated, check `process.env` gate names and `bun:bundle` `feature(...)` calls rather than assuming code is dead.
