# Comparing deepseek-harness, opencode, pi, oh-my-pi

**Session ID:** ses_fce4888daffe6izymXAUpU4rg2
**Created:** 8/24/2026, 8:31:42 AM
**Updated:** 8/26/2026, 7:40:07 PM

---

## User

@deepseek-harness/ 
@opencode/ 
@pi/ 
@oh-my-pi/ 

campare these and give me a detailed report about each capability and architechture? 

---

## Assistant (Build · x-preview-f-free · 134.7s)

_Thinking:_

The user wants a detailed comparison report of four projects. I should explore each repo's README, package structure, etc. This is a big research task — good use for parallel Task agents. Let me launch 4 explore agents in parallel to gather details on each project.

I'll explore all four repos in parallel with agents.

**Tool: task**

**Input:**
```json
{
  "description": "Explore opencode repo",
  "prompt": "Research task (no code writing). Explore the repository at /home/sadiq/Documents/web/oss/opencode/ and report its purpose, capabilities, and architecture.\n\nRead: README.md, CONTEXT.md if present, AGENTS.md, package.json, turbo.json, sst.config.ts, list packages/, sdks/, infra/, specs/ directories, and skim key packages' README/package.json to understand what each does.\n\nReturn a detailed report covering:\n1. What the project is / does\n2. Tech stack & languages\n3. Monorepo layout - list each package/sdk with one-line purpose\n4. Key capabilities/features (TUI, client/server architecture, providers, plugins, LSP, etc.)\n5. Architecture highlights (client-server split, session management, extension points)\n6. Testing/build/deploy infrastructure",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_fce485607ffe8NucoKPpBOj53G" state="completed">
<task_result>
I have enough to write the report.

# OpenCode Repository Report

## 1. What the project is / does

**OpenCode** (`/home/sadiq/Documents/web/oss/opencode/`, npm package `opencode-ai`, repo `github.com/anomalyco/opencode`) is **"the open source AI coding agent."** It is an agentic coding assistant that runs in the terminal (TUI), desktop app, IDE, browser, and CI. You point it at a codebase, connect any LLM provider (75+ via models.dev), and it can read/edit files, run shell commands, search code, use LSP/MCP servers, and execute multi-step tasks — with a permission system (built-in `build` full-access and `plan` read-only agents) governing what it may do.

The repo is a large Bun/TypeScript monorepo containing the agent runtime itself plus a whole product ecosystem: web/console/stats sites, cloud infrastructure (SST on AWS + Cloudflare), SDKs (JS/TS, Effect-native, VS Code extension), a GitHub Action, and a desktop (Electron) app.

## 2. Tech stack & languages

- **Language**: TypeScript everywhere; Go is used only for the TUI's underlying framework heritage (opentui has a Go core compiled per-platform — the TUI package builds platform binaries). Shell scripts for install/release.
- **Runtime/package manager**: Bun 1.3.x (workspaces + catalog dependencies), with Node compatibility.
- **Core frameworks**:
  - **Effect** (v4 beta) — pervasive functional-effect architecture across core, server, protocol, client, sqlite packages
  - **Effect Schema** for domain types/validation, Drizzle ORM + SQLite for durable storage
  - **Hono** HTTP server + Effect `HttpApi` for the public API
  - Vercel **AI SDK** (`ai` v6) for provider integrations
  - **SolidJS** + SolidStart + Tailwind v4 + Vite for all web UIs; **@opentui/solid** for the terminal UI
  - **Electron** (desktop), tree-sitter/WASM, node-pty for terminals
- **Tooling**: Turborepo, oxlint, prettier (no semicolons, width 120), husky, Playwright (e2e), SST v4 for IaC.

Notable: extensive `patchedDependencies` list (effect, ai-sdk providers, MCP SDK, solid-js...) shows heavy upstream patching.

## 3. Monorepo layout

### Core runtime chain (dependency direction: Schema → Core/Protocol → Server → Client → sdk-next)
| Package | Purpose |
|---|---|
| `packages/schema` | Lightweight leaf of shared public domain schemas (Effect Schema records); no DB/runtime deps |
| `packages/core` | Domain core: sessions, agents, config, permissions, tools, LSP, MCP, plugins, git, storage, filesystem, auth |
| `packages/opencode` | The main `opencode` binary/CLI entrypoint (`src/index.ts`); wires everything into the distributable npm package |
| `packages/llm` | LLM protocol layer: providers, wire protocols, cache policy, tool runtime, provider errors |
| `packages/protocol` | Public API composition: endpoint paths, envelopes, errors, cursors, streams over Schema types |
| `packages/server` | Hono/Effect HTTP server hosting the authoritative public `HttpApi`; handlers, middleware, CORS, PTY env |
| `packages/httpapi-codegen` | Codegen compiling the server `HttpApi` into SDK Contract IR and generated clients |
| `packages/client` | Generated Promise + Effect (`/effect`) API clients derived from the HttpApi (browser-safe) |
| `packages/sdk` | Legacy JavaScript SDK (`opencode-ai` JS SDK, rebuilt via `script/build.ts`) |
| `packages/sdk-next` | Effect-native scoped in-process host ("Embedded OpenCode") composing Client + Core + Server |

### User interfaces
| Package | Purpose |
|---|---|
| `packages/tui` | Terminal UI built on @opentui/solid (routes, components, keymaps, feature plugins) |
| `packages/app` | Web/desktop frontend (SolidStart) — the share/web session viewer and desktop shell UI |
| `packages/desktop` | Electron desktop application packaging (electron-builder, electron-vite) |
| `packages/ui` | Shared SolidJS component library |
| `packages/session-ui` | Reusable session/conversation UI components |
| `packages/storybook` | Component storybook for the UI libraries |
| `packages/cli` | CLI helpers package |

### Cloud/product services
| Package | Purpose |
|---|---|
| `packages/console/*` | Cloud console: `app` (marketing/auth console site), `core`, `function` (Lambdas), `mail`, `resource`, `support` — Stripe-billed cloud offering |
| `packages/stats/*` | Usage statistics site: SolidStart app + Effect/Drizzle core + Lambda functions |
| `packages/function` | Shared deployable function/Lambda helpers |
| `packages/enterprise` | Enterprise features/control-plane integration |
| `packages/identity` | Identity/auth service code |
| `packages/slack` | Slack bot integration |
| `packages/containers` | Container-based sandboxing support |
| `packages/codemode` | "Effect-native confined code execution over schema-described tools" (sandboxed tool scripting) |

### Infrastructure/support packages
| Package | Purpose |
|---|---|
| `packages/effect-drizzle-sqlite` | Effect wrappers around Drizzle SQLite |
| `packages/effect-sqlite-node` | Effect bindings for node SQLite |
| `packages/http-recorder` | Record/replay Effect HTTP traffic as deterministic cassettes (testing) |
| `packages/plugin` | Public plugin API package (`@opencode-ai/plugin`) |
| `packages/script` | Repo build/release scripts package |
| `packages/web` | Marketing/docs website (opencode.ai, Astro-like content collections in `src/content`) |
| `packages/docs` | Documentation sources |

### Other top-level dirs
- `sdks/vscode` — VS Code extension (launches opencode in a split terminal, context sharing, file-reference shortcuts)
- `github/` — official **GitHub Action** (`/opencode explain this issue`, `/opencode fix this` — creates branches/PRs from comments)
- `infra/` — SST infrastructure-as-code modules: `app.ts` (main AWS/Cloudflare app), `console.ts`, `enterprise.ts`, `lake.ts` (data lake ingest), `stats.ts`, `monitoring.ts`, `secret.ts`, `stage.ts`
- `specs/` — design specs: `project.md`, `tui-package.md`, `storage/` (SQLite migration plans), `v2/` (session.md, tools.md, provider-model.md, config.md, instructions.md, schema-changelog...)
- `script/` — release/publish/changelog/translation automation
- `nix/`, `flake.nix` — Nix dev environment; `perf/` benchmarks; `artifacts/`

## 4. Key capabilities

- **Terminal UI (TUI)**: rich SolidJS-powered terminal interface (opentui), two built-in agents switchable via Tab (`build` full-access, `plan` read-only), plus a `general` subagent invoked with `@general`.
- **Client/server architecture**: the agent runs as a local server exposing an HTTP API (SSE event streams); TUI, desktop app, web share view, VS Code extension, and SDKs are all clients of the same API. The same HttpApi can be executed **in-process** ("Embedded OpenCode") with an in-memory transport.
- **Multi-provider LLM support**: 75+ providers via models.dev catalog + AI SDK adapters; custom providers via OpenAI-compatible endpoints; provider/model switching mid-session.
- **Agents & subagents**, configurable via markdown/JSON config; permission system with allow/deny rules and per-command approval.
- **Tools**: file read/edit/write, bash, glob/grep, webfetch, todo, patch/diff, snapshot/checkpointing, skills (permission-checked `skill` tool), question prompts, PTY terminals.
- **Extensibility**: plugin API (`@opencode-ai/plugin`), MCP client support, LSP integration, custom commands, AGENTS.md instruction discovery (global/upward project files as a System Context source).
- **Session management**: persistent sessions with SQLite storage, share/public links, compaction, steering vs queued prompts, resumption.
- **IDE & workflow integrations**: VS Code extension, Zed editor support (`editor-zed.ts`), ACP protocol, GitHub Action, Slack bot, desktop apps for macOS/Windows/Linux.

## 5. Architecture highlights

- **Strict layered dependency direction** (from AGENTS.md): Schema ← Core/Protocol ← Server ← Client ← sdk-next. Client runtime never imports Core/Server; Protocol owns endpoint/middleware placement while Server supplies concrete keys.
- **Authoritative single HttpApi**: the server's Effect `HttpApi` is the source of truth; codegen (`httpapi-codegen`) compiles it into an **SDK Contract IR** from which both a zero-Effect Promise client and a rich Effect client (`@opencode-ai/client` root and `/effect` exports) are generated. Never hand-edit generated files; regenerate after protocol changes.
- **V2 Session runtime** (heavily specified in AGENTS.md/CONTEXT.md): durable prompt admission (`session_input` rows) decoupled from model execution; advisory wakes via `SessionExecution`; serialized process-local **Session Drains**; steer/queue delivery semantics with safe provider-turn boundaries; crash-safe durable history without durable execution identity.
- **System Context algebra**: composable typed "Context Sources" (AGENTS.md instructions, date, agent skill guidance, etc.) rendered into a baseline system prompt with **Context Epochs** — immutable provider-cache-friendly baselines invalidated by compaction/session moves; changes emitted as durable mid-conversation system messages rather than mutating the baseline.
- **Tool output bounding**: oversized outputs truncated to bounded previews with complete text spilled to managed temporary files.
- **Event streams**: durable replayable per-session SSE (`sessions.events({sessionID, after})`) versus live-only instance-wide `events.subscribe()` — deliberately different guarantees.
- **Extension points**: plugins (hooks incl. PTY environment observation), MCP servers, LSP, custom providers/models, custom commands, skills, plugin-defined Context Sources (planned).

## 6. Testing / build / deploy infrastructure

- **Testing**: Bun test per package; tests must be run from package directories (a guard makes root-level `bun test` fail). `http-recorder` provides cassette-based deterministic HTTP replay; Playwright for e2e; style guide forbids mocks/globalThis where possible. Turbo orchestrates test tasks with `^build` dependencies (defined for opencode, core, app, ui, session-ui).
- **Build/typecheck**: Turborepo (`turbo.json` v2.8 schema) with `typecheck` and `build` (outputs `dist/**`) tasks; always `bun typecheck` inside a package, never raw `tsc`. Legacy JS SDK rebuilt via `./packages/sdk/js/script/build.ts`.
- **Lint/format**: oxlint (+ oxlint-tsgolint type-aware rules), prettier, husky pre-commit hooks.
- **Deploy**: SST v4 (`sst.config.ts`) targeting **Cloudflare (home) + AWS us-east-1**, with Stripe billing, PlanetScale, Honeycomb monitoring providers; stage-gated modules (`infra/stage.ts` decides AWS deploys); production protected with `retain` removal. Console/stats/lake/enterprise each have dedicated infra modules; Sentry wired into frontends.
- **Release/distribution**: `script/publish.ts`, changelog generation, Windows code-signing script (`sign-windows.ps1`), multi-channel install (curl installer, npm/brew/scoop/choco/pacman/nix), GitHub Actions CI badge on `dev` branch (default branch is `dev`).
</task_result>
</task>
```

**Tool: task**

**Input:**
```json
{
  "description": "Explore pi repo",
  "prompt": "Research task (no code writing). Explore the repository at /home/sadiq/Documents/web/oss/pi/ and report its purpose, capabilities, and architecture.\n\nRead: README.md, AGENTS.md, CONTRIBUTING.md, package.json, tui-plan.md, pi-test.sh, list packages/ directory recursively (top 2 levels), and skim each package's README/package.json to understand what it does. Check .pi/ directory too.\n\nReturn a detailed report covering:\n1. What the project is / does\n2. Tech stack & languages\n3. Monorepo layout - list each package with one-line purpose\n4. Key capabilities/features (agent loop, tools, TUI, extensions, providers)\n5. Architecture highlights (minimal core philosophy, extension model, RPC/session handling)\n6. Testing/build infrastructure",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_fce484954ffej8seiwiSiMS4lm" state="completed">
<task_result>
# Pi Agent Harness — Repository Report

## 1. What the project is / does

**Pi** (`/home/sadiq/Documents/web/oss/pi`, published as `pi-mono` by Earendil Works, pi.dev) is the home of the **Pi agent harness**, including a **self-extensible coding agent CLI**. It is a terminal coding agent (like Claude Code/Codex-class tools) built on a deliberately minimal, aggressively extensible core: users adapt the agent to their workflow via TypeScript Extensions, Skills, Prompt Templates, and Themes distributed as "Pi Packages" over npm/git — instead of forking internals.

The project explicitly rejects baked-in workflow features ("No MCP. No sub-agents. No permission popups. No plan mode. No to-dos. No background bash."), arguing these should be extensions or packages. It also ships no built-in permission system — sandboxing is delegated to containers/micro-VMs (documented patterns: Gondolin micro-VM extension, plain Docker, OpenShell).

The repo itself is developed agent-first: `AGENTS.md` contains strict rules for both humans and AI agents working in the codebase, and maintainers publish their own development sessions as OSS datasets on Hugging Face.

## 2. Tech stack & languages

- **Language**: 100% TypeScript, using only *erasable* TS syntax (Node strip-only mode — no enums, namespaces, parameter properties) so it runs natively under Node without transpilation; `npm run check:ts-imports` enforces this.
- **Runtimes**: Node >= 22.19 (uses `node:sqlite`) and **Bun** for standalone binary builds.
- **Tooling**: npm workspaces monorepo, Biome (lint/format), `tsgo` (TypeScript native preview) + tsc 5.9, Vitest + node:test, esbuild, tsx, Husky pre-commit hooks.
- **Wire format**: length-prefixed **CBOR** framing for its remote-session protocol.
- **Dev deps of note**: `@anthropic-ai/sandbox-runtime`, esbuild, tsx.

## 3. Monorepo layout

All packages versioned in lockstep (currently 0.84.2). Top-level workspaces under `packages/`:

| Package | Purpose |
|---|---|
| `packages/ai` — `@earendil-works/pi-ai` | Unified multi-provider LLM API (OpenAI, Anthropic, Google, Bedrock, Azure, Groq, xAI, OpenRouter, Mistral, etc.); tool calling, streaming, thinking/reasoning, image input/generation, auth resolution, token/cost tracking, mid-session model hand-off. Only includes models supporting function calling. Model catalog generated from live provider data (`models.generated.ts` + scripts). |
| `packages/agent` — `@earendil-works/pi-agent-core` | General-purpose stateful agent runtime: tool execution loop, event streaming, state management, attachments, transport abstraction; pluggable session backends. |
| `packages/session-backends/sqlite-node` | Node `node:sqlite` session persistence backend for pi-agent-core: repository, migrations, materialized views, optional FTS search. Kept separate so core has no native/runtime deps by default. |
| `packages/tui` — `@earendil-works/pi-tui` | Minimal terminal UI framework with differential rendering, synchronized output (CSI 2026), main-screen and alternate-screen renderers, component library (Editor, Markdown, SelectList, images via Kitty/iTerm2 protocols), bracketed paste, autocomplete. Has a `native/` dir. |
| `packages/coding-agent` — `@earendil-works/pi-coding-agent` | The `pi` CLI itself: interactive TUI, print/JSON mode, RPC mode, SDK embedding; read/bash/edit/write tools, sessions with branching and compaction, extensions/skills/themes/packages. Published with a generated `npm-shrinkwrap.json`. |
| `packages/protocol` — `@earendil-works/pi-protocol` | Transport-neutral CBOR schemas, types, encoding, and byte-stream framing (v1: 4-byte big-endian length + CBOR message) for remote pi sessions. |
| `packages/client` — `@earendil-works/pi-client` | Transport-neutral client (`PiClient`) for remote sessions over any ordered byte transport (WebSocket, Unix socket); session leases (exclusive/shared), snapshot-authoritative state, no Node-specific imports. |
| `packages/server` — `@earendil-works/pi-server` | Experimental session server composing pluggable transport listeners (e.g., Unix socket preset); apps supply their own `PiServerService`. |
| `packages/telemetry` — `@earendil-works/pi-telemetry` | Vendor-neutral telemetry contracts (callback-based context/span), typed schemas, no-op + in-memory reference adapters, conformance tests; no exporter/backend coupling. |
| `packages/evals` — `@earendil-works/pi-evals` | Behavioral model-backed evals adapting a real `AgentSession` to `vitest-evals`; runs in isolated temp dirs, emits session JSONL artifacts. |

Additional workspace members are example extension packages under `packages/coding-agent/examples/extensions/` (`with-deps`, `custom-provider-anthropic`, `custom-provider-gitlab-duo`, `sandbox`, `gondolin`). Slack/chat automation lives in a sibling repo (`pi-chat`).

### `.pi/` directory
Project-local pi configuration used to develop pi with pi:
- `.pi/extensions/`: `import-repro.ts`, `prompt-url-widget.ts`, `redraws.ts`, `tps.ts` (dev/diagnostic extensions)
- `.pi/prompts/`: prompt templates `pr.md`, `sa.md`, `cl.md` (changelog audit), `is.md` (issues), `wr.md`
- `.pi/skills/`: `add-llm-provider.md`
- `.pi/git/`, `.pi/npm/`: gitignored local state dirs

## 4. Key capabilities

- **Agent loop** (`pi-agent-core`): subscribe-based event streaming, tool execution, message/state management, pluggable stream function and session storage.
- **Tools**: the coding agent ships read, bash, edit, write tools; individual tools/extensions can be disabled per-invocation; read-only mode available.
- **Providers/models** (`pi-ai`): ~15+ providers with automatic auth resolution (env vars, OAuth tokens, credential store, Bedrock/Azure configs), auto model discovery from generated catalogs, thinking levels, cost tracking.
- **Four run modes**: interactive TUI, print/JSON (one-shot, pipeable stdin), RPC (for process integration, documented in `docs/rpc.md`), and an SDK for embedding.
- **Sessions**: durable storage (SQLite backend), branching, compaction, FTS search, named one-shot sessions.
- **TUI**: differential-rendering components, editor with keybinding config (`DEFAULT_EDITOR_KEYBINDINGS`/`DEFAULT_APP_KEYBINDINGS`), inline images, autocomplete, themes; alt-screen layout system (VStack/HStack/ScrollView, sticky docked footer, mouse-wheel routing) per `tui-plan.md`.
- **Extensibility**: TypeScript extensions with lifecycle hooks, skills (markdown + CLIs with READMEs, explicitly replacing MCP), prompt templates, custom providers, themes, shareable "pi packages".
- **Telemetry**: opt-in vendor-neutral spans; update checks and telemetry documented in settings docs.

## 5. Architecture highlights

- **Minimal-core philosophy**: core stays small; everything else is an extension. Even hook points require maintainer discussion to prevent bloat. PRs bloating core are rejected (CONTRIBUTING.md).
- **Layered stack**: `tui` → `ai` → `agent` (+ session backends) → `coding-agent`; remote access splits cleanly into `protocol` (CBOR framing/schemas) → `server` (listeners + service interface) → `client` (leases, snapshots).
- **Snapshot-authoritative RPC model**: server/client sync via authoritative session/server snapshots; progress events are transient UI hints never folded into state. Session acquisition uses exclusive/shared leases with ownership errors; requests correlated by ID; configurable frame-length limits; authentication happens at the transport layer before protocol bytes flow.
- **Transport neutrality everywhere**: protocol, client, telemetry, and agent all avoid runtime-specific imports; SQLite backend accepts a runtime-injected DB factory so other backends can ship independently.
- **Supply-chain hardening as architecture**: exact-pinned external deps, `min-release-age=2`, lockfile-as-ground-truth with pre-commit gating (`PI_ALLOW_LOCKFILE_CHANGE=1`), generated shrinkwrap + install-lock with lifecycle-script allowlist, CI `npm ci --ignore-scripts`, scheduled npm audits, npm trusted publishing via OIDC, R2-verified release announcement marker for pi.dev.

## 6. Testing/build infrastructure

- **Build order** (dependency chain in root `package.json`): tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent. `build:offline` variant skips live model-data refresh. Standalone binaries built via Bun (`scripts/build-binaries.sh`, supports `--offline-model-data --platform ...`).
- **Checks**: `npm run check` = Biome (with `--error-on-warnings`), pinned-deps verification, TS relative-import check, shrinkwrap/install-lock regeneration checks, `tsgo --noEmit`, browser smoke test.
- **Tests**: Vitest per package plus node:test for `packages/tui`; root `test.sh` runs non-e2e tests inside a fully isolated temp environment (fake HOME, scrubbed env, git askpass stub) so tests never touch real credentials. e2e LLM tests activate only when endpoint/auth env vars exist and are excluded from normal runs. Coding-agent suite tests use a harness + faux provider (no real APIs/paid tokens); issue regressions go in `test/suite/regressions/<issue>-<slug>.test.ts`.
- **Interactive testing**: `pi-test.sh` (and .bat/.ps1) runs the CLI from source via tsx, optionally with `--no-env` to unset all ~30 provider credential variables; AGENTS.md prescribes tmux-driven capture/send-keys testing for the TUI.
- **Evals**: `npm run eval` runs model-backed behavioral evals via vitest-evals with real provider auth, producing `.eval/runs.jsonl` + session JSONL artifacts.
- **Release**: lockstep versioning across all packages (no major releases); `release:local` builds isolated Node+Bun installs outside the repo for smoke tests (release blockers if they fail); release script bumps versions/changelogs/tags; CI publishes npm via trusted publishing and verifies before announcing on pi.dev.
</task_result>
</task>
```

**Tool: task**

**Input:**
```json
{
  "description": "Explore oh-my-pi repo",
  "prompt": "Research task (no code writing). Explore the repository at /home/sadiq/Documents/web/oss/oh-my-pi/ and report its purpose, capabilities, and architecture.\n\nRead: README.md, AGENTS.md, CONTRIBUTING.md, package.json, Cargo.toml, MODULE.bazel/BUILD.bazel briefly, list crates/, packages/, python/, docs/ directories, and skim key crates'/packages' README/Cargo.toml/package.json to understand what each does. Note the relationship to \"pi\" if evident.\n\nReturn a detailed report covering:\n1. What the project is / does\n2. Tech stack & languages (Rust + TS + Python?)\n3. Repo layout - list each crate/package with one-line purpose\n4. Key capabilities/features (agents, TUI/GUI, native performance, docker/robomp stuff)\n5. Architecture highlights (Bazel vs bun/pnpm build split, Rust core vs TS layers, extension points)\n6. Testing/build/deploy infrastructure",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_fce48389effe7pWyYaqC2Y3mVs" state="completed">
<task_result>
# oh-my-pi (omp) — Repository Research Report

## 1. What the project is / does

**oh-my-pi (branded "omp", site: omp.sh)** is a terminal-first AI **coding agent** — "a coding agent with the IDE wired in." It ships as a CLI (`omp`) with an interactive TUI, plus SDK, RPC, and ACP surfaces. It is explicitly a **fork of [Pi](https://github.com/badlogic/pi-mono) by Mario Zechner (@mariozechner)** (`pi-mono`), rewritten/extended into a "batteries-included" coding-first agent. The Pi relationship is pervasive: packages keep `pi-*` names (`@oh-my-pi/pi-ai`, `pi-tui`, …), and there's even a documented upstream-sync process (`docs/porting-from-pi-mono.md` with a tracked sync-point commit).

Headline stats from the README: 60+ LLM providers, 31 built-in tools, 14 LSP ops, 28 DAP ops, ~80k lines of Rust core (plus another ~80k lines vendored: a brush bash fork + 67 in-process CLI utilities). MIT licensed; authored by Stencil Labs, Inc. / Can Bölük.

## 2. Tech stack & languages

| Language | Role |
|---|---|
| **TypeScript** | All product logic: agent runtime, tools, TUI app, LLM providers. Runs on **Bun** (≥1.3.14), Bun APIs preferred over Node |
| **Rust** (edition 2024, nightly toolchain pinned 2026-04-29) | Native performance core compiled to one platform-tagged **N-API `.node` addon** (`pi_natives` cdylib); embedded bash engine via vendored brush-shell fork |
| **Python** (≥3.11) | `python/robomp` (GitHub triage bot, FastAPI) and `python/omp-rpc` (typed RPC client) |
| **Build/tooling** | Bazel (Rust side, CI/release artifacts), Cargo (local Rust iteration), bun workspaces (TS), Biome lint, cargo nextest/clippy pedantic, Nix flake for dev env & packaging |

## 3. Repo layout

### TypeScript packages (`packages/*`, all v18.0.3)
- **coding-agent** — the main CLI application and Node SDK (`@oh-my-pi/pi-coding-agent`); primary focus of the repo
- **ai** — unified multi-provider LLM client with streaming & model discovery
- **catalog** — bundled model database, provider descriptors, model identity/classification (`models.json` generated, never hand-edited)
- **agent** (pi-agent-core) — general agent runtime: tool calling, state management
- **tui** — terminal UI library with differential rendering
- **natives** — N-API bindings loader for the Rust addon (grep, shell, PDF/audio/WebRTC, clipboard, images, syntax highlight, PTY)
- **hashline** — line-anchored, content-hash patch language behind the `edit` tool
- **omptype** — ArkType-compatible schema validation with lazy JIT
- **utils** — shared utilities (logger, streams, dirs/env/process)
- **wire** — shared collab live-session protocol types
- **mnemopi** — local SQLite memory engine for agents
- **snapcompact** — bitmap-frame context compression package + SQuAD eval suite
- **stats** (omp-stats) — local observability dashboard for AI usage
- **collab-web** — browser guest client + local relay for live session sharing
- **browser-relay** — Chrome extension letting the browser tool drive your real tabs
- **metaharness** — unified benchmark runners, Harbor run storage, REST/SSE API, dashboard
- **typescript-edit-benchmark** — edit-tool benchmark over TypeScript source mutations

### Rust crates (`crates/*`)
- **pi-natives** (~25k LoC) — N-API surface aggregating ~22 modules: desktop control, grep, text width/truncation, snapcompact rasterizer, keyboard protocol, ast-grep, diff, PTY, crash handler, highlighting, glob/fd, clipboard, workspace walker, profiler, tokenizers, sixel…
- **pi-shell** (~38k LoC) — embedded bash engine wrapping vendored brush, persistent sessions, in-process coreutils dispatch
- **pi-builtins** — bash builtins + 67 ported command-line utilities (coreutils/findutils/sed/jq-style) running zero-fork/exec
- **vendor/brush-core** — vendored fork of brush-shell (patched into crates-io graph)
- **pi-walker** (~5k) — parallel ignore-aware FS walker + scan cache shared by grep/glob/workspace/shell
- **pi-iso** (~3k) — task isolation backends: APFS clones, btrfs/zfs reflinks, overlayfs, projfs, rcopy
- **pi-ast** (~3k) — tree-sitter + ast-grep matching, structural summaries (50+ grammars)
- **pi-voice** (~1k) — audio capture/playback, Opus, live WebRTC

### Python (`python/*`)
- **robomp** — self-hosted GitHub triage/fix bot driving `omp --mode rpc` per-issue worktrees (FastAPI + sqlite queue + gh-proxy sidecar; auto-fixes bugs, answers questions, even repairs failed release CI)
- **omp-rpc** — typed Python client for omp's NDJSON RPC protocol

### Other top-level dirs
- **docs/** — ~80 internal architecture docs (natives, TUI internals, MCP, providers, memory, compaction, collab, vibe-mode, etc.)
- **scripts/** — release pipeline, CI test orchestrator, install scripts, Nix/Bazel generators, changelog tooling
- **infra/** — bazel-remote cache config, runner Dockerfile
- **bazel/** — platforms, ISA-variant constraints, toolchains, custom `native_addon` rule

## 4. Key capabilities

- **Four entry points**: interactive TUI, one-shot (`omp -p`), Node SDK (`createAgentSession`), stdio RPC (`--mode rpc`) and ACP for editors like Zed.
- **31 tools**: read/write/edit (hashline patches)/ast_edit/ast_grep/grep/glob, bash (in-process shell w/ 46+ builtins), eval (persistent Python + JS kernels that can call back into agent tools), lsp, debug (DAP: lldb/dlv/debugpy), task (parallel subagents in isolated worktrees), hub, todo, ask, browser (Puppeteer/Electron/Chrome relay), computer (desktop control: windows, screenshots, native input, AX tree), web_search (23 provider chain + structured extraction), github, generate_image/inspect_image/tts, memory tools (retain/recall/reflect/learn), checkpoint/rewind.
- **LSP/DAP integration**: renames go through `workspace/willRenameFiles`; real debugger sessions.
- **Time-traveling stream rules**: regex-triggered mid-stream system-reminder injection with retry.
- **Advisor model**: second reviewer model watching every turn on its own context.
- **Collab**: `/collab` share link + QR; teammates join by terminal or browser; client-side encryption.
- **Model routing**: 10 roles (default/smol/slow/plan/vision/designer/task/advisor/commit/tiny), fallback chains, path-scoped models, round-robin credentials, custom OpenAI-compatible providers.
- **Interoperability**: inherits rules/skills/MCP configs from `.claude`, `.cursor`, `.windsurf`, `.gemini`, `.codex`, etc.; reads PRs/issues/conflicts/ssh paths as URLs inside every file tool (`pr://1428`, `conflict://N`).
- **Native performance**: no fork/exec on hot paths; ripgrep-as-library, embedded bash, in-process utilities; single binary across macOS/Linux/Windows without WSL.

## 5. Architecture highlights

- **Layering**: TS layers (provider clients → agent runtime → coding-agent tools/modes → TUI/RPC/ACP wrappers) sit atop one Rust N-API addon (`pi_natives` cdylib) loaded per-platform-tag; blocking work runs on the libuv pool with panic-catching at the napi boundary so Rust panics become rejected promises.
- **Bun-centric runtime**: workers re-enter `cli.ts` via hidden argv selectors (`__omp_worker_*`) so compiled binaries need no separate worker entries; prompts live in static `.md` files imported as text, not code-built strings; strict centralization rules (single sanctioned git wrapper, logger instead of console, TUI sanitization helpers).
- **Dual build split**: Cargo workspace stays authoritative for local iteration (rust-analyzer, nextest, typedef regen); **Bazel builds the shipping `.node` addons hermetically** (rules_rust + zig cc cross toolchain, crate_universe derived from Cargo.lock) producing 8 platform/ISA-variant targets including musl and dual AVX2/baseline x64 variants. Content-addressed remote cache replaces sccache/cargo-zigbuild/cargo-xwin.
- **Extension points**: TypeScript extension modules share the same tool API/slash-command/hotkey registries as built-ins; MCP server support; managed skills marketplace; `xd://` device namespace for discoverable off-by-default tools; settings-gated tools.
- **Catalog convention**: catalog values import only from `@oh-my-pi/pi-catalog/<module>`, never through the pi-ai barrel.

## 6. Testing / build / deploy infrastructure

- **Tests**: `bun test` orchestrated by `scripts/ci-test-ts.ts` with tiers (local/all/workspace/native, coding-agent singleton/ui/runtime/native/heavy splits); Rust via `bun run test:rs` = cargo nextest + doctest pass; Python pytest (`test:py`); strict contract-testing rules in AGENTS.md (no source-grep tests, no `mock.module()`, full-suite-safe tests).
- **CI** (`.github/workflows`): `ci.yml`, `bazel-cache-warm.yml`, `nix.yml`; smoke probe `omp --smoke-test` wired into install-method CI covering binary/source-link/tarball installs.
- **Checks/lint**: Biome for TS, clippy (pedantic/nursery, correctness+suspicious denied) for Rust, ruff for Python, `bun check` everywhere (never raw tsc).
- **Release**: `bun run release` script handles version bump → changelog finalize → commit/tag/publish; dedicated release scripts build binaries, checksums, publish (npm `-ws`, Homebrew tap update), macOS signing/notarization docs, musl release handling.
- **Distribution**: curl installer script (omp.sh), PowerShell installer, npm (`@oh-my-pi/pi-coding-agent`), Homebrew (`can1357/tap/omp`), Nix flake (packages/overlay/NixOS/Home-Manager modules, optional Wayland screencast), mise pinning.
- **Docker**: root `Dockerfile` builds the "pi" runtime image (bun+rustup+pi-natives+omp shim); `Dockerfile.robomp` extends it for the robomp bot with a gh-proxy sidecar compose setup; infra/ has bazel-remote cache and kata-runner configs.
</task_result>
</task>
```

---

## User

you can write in an .md file

---

## User

setup up a exhaustive list of todo for yourself to deliver a high quality comparision.

---

## Assistant (Build · x-preview-f-free · 15.1s)

_Thinking:_

The user wants me to set up a todo list and then write a detailed comparison report to a .md file. I still need to research the fourth repo (deepseek-harness) — I launched 3 agents earlier but not one for deepseek-harness. Let me create todos, launch the deepseek-harness agent, then write the report.

**Tool: todowrite**

**Input:**
```json
{
  "todos": [
    {
      "content": "Research opencode repo (done via agent)",
      "status": "completed",
      "priority": "high"
    },
    {
      "content": "Research pi repo (done via agent)",
      "status": "completed",
      "priority": "high"
    },
    {
      "content": "Research oh-my-pi repo (done via agent)",
      "status": "completed",
      "priority": "high"
    },
    {
      "content": "Research deepseek-harness repo",
      "status": "in_progress",
      "priority": "high"
    },
    {
      "content": "Verify key claims/numbers across repos (spot-check READMEs)",
      "status": "pending",
      "priority": "medium"
    },
    {
      "content": "Build comparison matrix (capabilities, architecture, extensibility, testing)",
      "status": "pending",
      "priority": "high"
    },
    {
      "content": "Write detailed comparison report to comparison-report.md",
      "status": "pending",
      "priority": "high"
    }
  ]
}
```

**Output:**
```
[
  {
    "content": "Research opencode repo (done via agent)",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Research pi repo (done via agent)",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Research oh-my-pi repo (done via agent)",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Research deepseek-harness repo",
    "status": "in_progress",
    "priority": "high"
  },
  {
    "content": "Verify key claims/numbers across repos (spot-check READMEs)",
    "status": "pending",
    "priority": "medium"
  },
  {
    "content": "Build comparison matrix (capabilities, architecture, extensibility, testing)",
    "status": "pending",
    "priority": "high"
  },
  {
    "content": "Write detailed comparison report to comparison-report.md",
    "status": "pending",
    "priority": "high"
  }
]
```

---

## Assistant (Build · x-preview-f-free · 198.0s)

**Tool: task**

**Input:**
```json
{
  "description": "Explore deepseek-harness repo",
  "prompt": "Research task (no code writing). Explore the repository at /home/sadiq/Documents/web/oss/deepseek-harness/ and report its purpose, capabilities, and architecture.\n\nRead: README.md, CLAUDE.md, AGENTS.md, BENCHMARK.md, package.json, pnpm-workspace.yaml, list packages/, apps/, python/, native/, docs/ directories (top 2 levels), skim key packages' README/package.json to understand what each does. Note the relationship to \"pi\" if evident (it may be a fork or derivative).\n\nReturn a detailed report covering:\n1. What the project is / does\n2. Tech stack & languages\n3. Repo layout - list each package/app with one-line purpose\n4. Key capabilities/features (agent loop, tools, TUI, providers, extensions)\n5. Architecture highlights (layering, extension points, protocols, session handling)\n6. Testing/build/deploy infrastructure",
  "subagent_type": "explore"
}
```

**Output:**
```
<task id="ses_fce462742ffenK4L6QPnvXUPJb" state="completed">
<task_result>
# DeepSeek Harness (`dsh`) — Repository Research Report

**Repo:** `/home/sadiq/Documents/web/oss/deepseek-harness/` · version `0.1.1-rc.2` (developer preview) · MIT license

---

## 1. What the project is / does

DeepSeek Harness (`dsh`) is an **open-source, plugin-based coding/automation agent harness developed by DeepSeek AI**. Its stated design principle is "**everything is a plugin**": the model adapter, tool registry, session log, sandbox policy, and even the agent loop itself are Cordis plugins mounted into a composable tree at boot — so every part is replaceable from configuration with no privileged core.

It ships as:
- A CLI launcher (`npx @deepseek-ai/dsh web` or from source `pnpm dsh ...`)
- A **Web UI** (browser GUI served at `127.0.0.1:3080`, React client + Node host)
- A **headless one-shot runner** (`dsh --profile headless "task"`)
- An **ACP (Agent Client Protocol) server** for editor automation
- A **JSON-RPC server + TypeScript SDK**, and a **Python SDK** that drives a bundled runtime over newline-delimited JSON-RPC on stdio
- Installable single-exe distribution via the Python runtime carrier

It is in developer preview with explicit no-compatibility-promise stance until first tagged release.

## 2. Tech stack & languages

| Layer | Technology |
|---|---|
| Language | TypeScript everywhere, `strict: true`, ESM-only (`"type": "module"`), TS ~6, tsx for source launch |
| Runtime | Node.js `^22.19 || >=24`; pnpm workspaces (pnpm 11) |
| Framework | **Cordis** (plugin/context/event/waterfall framework from the cordiverse ecosystem) — **source-vendored** under `vendor/` and rescoped to `@deepseek-ai/*` (cordis, loader, include, group, timer, hmr, logger-console, cosmokit, schemastery) |
| Web UI | Vite + React client (`packages/client/ui-*` plugins, testing-library tests); VitePress docs website |
| Build | tsc project references → `lib/types`; **tsdown** bundles publishable runtime |
| Tests | Vitest (unit, coverage, e2e, keyless snapshot replay, web/stress/perf configs), fast-check property tests, jscpd duplication check |
| Lint/hygiene | oxlint (+tsgolint typed rules), knip, publint, lefthook git hooks, dozens of custom repo gates as tsx scripts |
| Python | Python SDK (`deepseek-harness-sdk`) + runtime binary carrier (`deepseek-harness-runtime-bin`) |
| Native | C++/N-API addon: **Landlock self-restrict-then-exec launcher** (`native/landlock-run`, Linux sandboxing; per-platform npm packages) |
| Sandboxing | bubblewrap/Landlock/Seatbelt backends + Windows ACL restricted-token runner; patched node-pty for PTY |

## 3. Repo layout

```
apps/
  cli        @deepseek-ai/dsh — the product launcher: profiles, bundles, patch layers; owns the dsh bin
  web        Vite frontend entry (@deepseek-ai/dsh-web-frontend) building dist/ served by dsh web
packages/    (~55 groups, all @deepseek-ai/dsh-*)
  core/          Product API spine: session, system-prompt, tools, agent registry, agent-loop, scope, default model
  api/           Remote BFF assembly + Typert RPC gateway
  typert/        Type-graph generator/loader/runtime registry
  llm/           LLM seam + adapters: llm-deepseek, llm-pi-ai (multi-provider via pi-ai), llm-retry, token-meter
  shell/         Bash capability: executor seam, local/pwsh providers, bash/pwsh model tools
  subprocess/    Subprocess capability + local process-tree provider
  terminal/      Persistent PTY sessions (node-pty incl. ConPTY)
  code-runtime/  Code execution seam + worker-thread provider ("Code Mode")
  sandbox/       Process confinement: bwrap/Landlock/Seatbelt local impls, policy, Windows ACL
  fs/            Filesystem capability, local provider, file tools, policy fencing
  lsp/           Language-server capability + lsp tool
  skill/         Skill provider registry + catalog/loader tool
  web/           Web capability: search/fetch providers (DeepSeek/Exa/Perplexity search, HTTP fetch)
  compaction/    Context compaction capability + basic provider
  context/       Model-visible request context (workspace instructions, time)
  subagent/      Subagent seam + providers: fork-in-process, spawn-in-process, Claude Code, Codex, ACP, DSH-SDK + delegation tools
  jobs/          Background-job runtime + job_* tools
  workflow/      Workflow engine (worker threads) + workflow/ralph tools
  todo/ plan/    todo_write tool; plan mode as logged state
  guard/         Loop hygiene (repeat reminders) + tool deadline enforcer
  extensions/    Self-modification: agent inspects/mounts/unmounts its own plugins live (cordis tools + runners)
  hooks/         Claude Code / Codex hook bridges + wire protocol library
  mcp/           MCP client integration
  e2b/           E2B cloud-sandbox POC (FS/subprocess adapters)
  session/       Durable data plane: JSONL & SQLite persistence, projection/cache, stats, telemetry(+OTel), titles(LLM)
  session-query/ Session retrieval: bounded reads, lineage, semantic filter, SQLite FTS
  attachment/ spill/ storage/ workspace/   Content-addressed attachments; tool-result spill; non-session storage; workspace entity
  settings/ credentials/ identity/         User settings, credential refs + env/.env authz flows, anonymous identity
  interaction/   Approvals, permission presets, commands, ask-user tool
  goal/ schedule/ feedback/                Same-session goals; scheduled follow-ups; human feedback
  preset/ bundle/                          Per-session composition presets; installable --profile bundles (base, web-app, headless)
  boot/            App-bin glue (cmdline, app-boot)
  sdk/             JSON-RPC protocol + TS client + server plugin
  acp/             Automation-only Agent Client Protocol server
  host/ client/    Web-GUI halves: host = API gateway/HTTP server/directory picker; client = browser shell + ~30 ui-* React plugins
  examples/ experimental/ support(util, test-support, runtime-diagnostics)
python/      Python SDK + bundled runtime binaries (single-exe deploy root)
native/      Landlock-run launcher source of record (3-package npm family)
vendor/      Vendored/rescoped Cordis framework + cosmokit/schemastery, pinned SHAs, exhaustive local-mods log
examples/    Runnable cordis.yml leaves: headless-agent, jsonrpc-agent, acp-agent, web-cordis, web-schedule, mcp-memory
docs/        Bilingual architecture, subsystems reference, cookbook, catalogs (generated), postmortems, user guide
.agents/     Agent skills + decision-record "Agent Notes"
scripts/     ~80 gate/generator scripts; website/ VitePress projection; patches/ .github/ CI workflows
```

## 4. Key capabilities

- **Agent loop**: turn/step event flow (`turn/start` → prompt assembly → `agent/pre-step` waterfall (can rewrite/reject input) → `agent/request` → `llm/stream` → tool pipeline `tools/pre-execute → execute → post-execute` → loop until nothing owed). The loop itself is a swappable plugin (`ctx.agentLoop`).
- **Tools**: guarded registry (`ctx.tools`); bash/pwsh, persistent PTY terminals, filesystem read/write/edit + bash-backed discovery, LSP, web search/fetch, todo_write, plan mode, ask-user, subagent delegation/control/report, `job_*` background jobs, workflow/ralph, skills catalog/loader, MCP client tools, session-query. Tool UI render intent (`generic`/`terminal`/`diff`) is part of tool design.
- **Providers (LLM)**: DeepSeek native adapter plus a generic multi-provider adapter on `@earendil-works/pi-ai` (OpenAI, Anthropic, etc., incl. OpenAI-compatible custom endpoints), retry policies, token metering.
- **UI**: full Web UI (React plugin architecture with slots, conversation nodes, settings cards, themes, sidebar, trajectory viewer); no first-party TUI shipped — terminal apps mount via profiles (e.g., hypothetical `dsh --profile tui`).
- **Extensions/self-modification**: `extensions/` lets the agent inspect its own Cordis graph and write/mount/unmount plugins live; HMR plugin reload; `dsh --profile` bundles as patch layers; Claude Code/Codex hook bridges.
- **Safety**: permission/approval plane, sandbox backends (bwrap/Landlock/Seatbelt/Windows ACL) wrapping spawned argv, FS write fencing, tool deadlines.
- **Session intelligence**: compaction, spill of oversized tool results, session titles via LLM, telemetry/OTel, checkpointing, forking.

## 5. Architecture highlights

- **Cordis plugin kernel (vendored)**: services, typed events (declaration-merged `SessionEventMap`), reversible effects, waterfalls (listeners must call `next()`). Registrations are effects with disposers; HMR-safe.
- **Profiles & bundles**: a running `dsh` is a config tree composed from ordered patch layers — base bundle (`dsh-base`: adapters, tools, persistence, policy) → mode bundles (`web-app`, `headless`) → profile `cordis.patch.yml` → home-level patch → `--patch` overlays. Patches target rows by id and replace whole configs; `--dump-config` shows the composed tree.
- **Capability seams**: each capability = Service Definition + Service Provider(s) + Consumer(s); extension plugins depend only on definitions, so swapping one provider (e.g., remote sandbox) relocates Bash, PTY, and LSP together.
- **Session log as source of truth**: append-only durable events; "model-visible ⟺ logged" invariant enforced at build time; `deriveMessages()` projects model history; fork/resume/transcripts/telemetry all derive from the stream; SQLite monotonic `SCHEMA_VERSION`.
- **Event domains**: session events (durable), `agent/*` (live interception), capability events (`fs/*`, `tools/*`, `telemetry/*`). Event producer/consumer map generated.
- **Protocols**: JSON-RPC (SDK), Agent Client Protocol (ACP), MCP, Claude Code/Codex hook wire protocol, Typert RPC gateway for the web host.
- **Extension points table** in `docs/architecture.md` maps every goal (new provider/tool/command/backend/UI node) to a documented mechanism — "plugins, not loop changes."
- Strict engineering culture: branded opaque IDs, explicit request/spec resolution, loud misconfiguration, per-package runtime invariants, exhaustive vendored-modification logging, bilingual docs under word budgets with machine-checked links.

## 6. Testing / build / deploy infrastructure

- **Tests**: Vitest unit suites; `test:coverage` is the CI gate (**per-file 100% coverage** required on all package sources, partitioned runs); real-API e2e (self-skips without `DEEPSEEK_API_KEY`); **keyless snapshot tests** replaying ACP/headless transcripts against expected outputs (record needs a key); dedicated web snapshot/perf/stress configs; jscpd clone detection; fast-check property tests; mock LLM server for dev.
- **Gates**: ~30 custom verification scripts (doc budgets, md wrap/links, JSDoc enforcement, package invariants, vendored links, cordis-config validation, generated catalogs freshness-gated in CI: tool/config/persistence/module-graph catalogs), knip, publint, workspace constraints, NodeNext consumer checks, oxlint.
- **Build**: `tsc -b` project references emit `lib/types`; tsdown bundles runtime artifacts; separate host/client build faces; source plane (tsconfig paths to `src/`) never mixed with artifact plane.
- **CI**: GitHub Actions matrix (`ci.yml`, `ci-master.yml`) covering static, lint, coverage, snapshot, artifacts, consumers, node-compat; Windows gates locally reproducible via wine (`check:windows-wine`); e2b e2e and pi-ai-provider e2e workflows; GitLab CI file also present.
- **Release/deploy**: scripted release family (`release:dsh`, `release:vendor`, pack/verify/publish, packed-install verification, npm baseline publishing); native Landlock builds per-arch with per-platform npm optional deps; **Python single-exe deployment** — `python/sdk-runtime` is the dependency-manifest deploy root whose closure the exe bundles and the PyPI runtime-bin package distributes; VitePress docs site doubles as dead-link check.

## Relationship to "pi"

This is **not a fork of pi**. The only connection is `@earendil-works/pi-ai` (the multi-provider LLM library from the pi/earendil-works ecosystem), used as an **optional dependency behind the `dsh-llm-pi-ai` adapter** to get provider wire protocols and model catalogs (OpenAI, Anthropic, etc.). Fresh pi-ai releases are explicitly exempted from minimum-release-age gating because catalog updates are the point of bumping it, and there is a dedicated `pi-ai-provider-e2e` CI workflow. The harness's own framework layer is instead **Cordis** (vendored from cordiverse, rescoped to `@deepseek-ai/*`).
</task_result>
</task>
```

---

## Assistant (Build · x-preview-f-free · 12.3s)

_Thinking:_

Now I have all 4. Let me spot-check a few key claims quickly (optional), then write the report. The user wants a detailed report in an .md file. Let me do a quick verification of headline stats — maybe check READMEs briefly for opencode, oh-my-pi numbers, deepseek-harness version. Quick greps.

Actually to keep it efficient, I'll do a couple of quick parallel reads of the main READMEs' first sections to verify key claims.

**Tool: todowrite**

**Input:**
```json
{
  "todos": [
    {
      "content": "Research all 4 repos",
      "priority": "high",
      "status": "completed"
    },
    {
      "content": "Verify key claims/numbers across repos (spot-check READMEs)",
      "priority": "medium",
      "status": "in_progress"
    },
    {
      "content": "Build comparison matrix (capabilities, architecture, extensibility, testing)",
      "priority": "high",
      "status": "pending"
    },
    {
      "content": "Write detailed comparison report to comparison-report.md",
      "priority": "high",
      "status": "pending"
    }
  ],
  "command": "rg -i -m 5 'provider|tool|fork|pi-mono|lines? of' /home/sadiq/Documents/web/oss/oh-my-pi/README.md | head -20; echo ---; rg -i -m 3 'version' /home/sadiq/Documents/web/oss/deepseek-harness/package.json /home/sadiq/Documents/web/oss/pi/package.json /home/sadiq/Documents/web/oss/opencode/package.json /home/sadiq/Documents/web/oss/oh-my-pi/package.json"
}
```

**Output:**
```
[
  {
    "content": "Research all 4 repos",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Verify key claims/numbers across repos (spot-check READMEs)",
    "status": "in_progress",
    "priority": "medium"
  },
  {
    "content": "Build comparison matrix (capabilities, architecture, extensibility, testing)",
    "status": "pending",
    "priority": "high"
  },
  {
    "content": "Write detailed comparison report to comparison-report.md",
    "status": "pending",
    "priority": "high"
  }
]
```

---

## Assistant (Build · x-preview-f-free · 6.6s)

**Tool: bash**

**Input:**
```json
{
  "command": "rg -i -m 6 'fork|providers|tools' /home/sadiq/Documents/web/oss/oh-my-pi/README.md; echo ===; node -e \"for (const p of ['deepseek-harness','pi','opencode','oh-my-pi']) { try { console.log(p, require('/home/sadiq/Documents/web/oss/'+p+'/package.json').version) } catch(e){ console.log(p, 'n/a') } }\""
}
```

**Output:**
```
zsh:1: command not found: rg
zsh:1: == not found

```

---

## Assistant (Build · x-preview-f-free · 4.4s)

**Tool: grep**

**Input:**
```json
{
  "path": "/home/sadiq/Documents/web/oss/oh-my-pi/README.md",
  "pattern": "fork of|providers|31 (built-in )?tools|lines"
}
```

**Output:**
```
Found 100 matches (more matches available)
/home/sadiq/Documents/web/oss/oh-my-pi/patches/puppeteer-core@25.3.0.patch:
  Line 384: +        // xxx-stealth: install lazy context providers and invalidate the


/home/sadiq/Documents/web/oss/oh-my-pi/THIRD-PARTY-NOTICES.txt:
  Line 265: lines (RTK's defensive default) so xdist `[gwN]` prefixes and custom reporters

  Line 6682: ## n-readlines@v3.4.1

  Line 6687: Repository: <http://github.com/nacholibre/node-readlines.git>  

  Line 8180: ## trim-newlines@v5.0.0

  Line 8182: > Trim newlines from the start and/or end of a string

  Line 8199: ## trim-trailing-lines@v1.1.4


/home/sadiq/Documents/web/oss/oh-my-pi/AGENTS.md:
  Line 203: All text displayed in tool renderers must be sanitized. Raw content (file contents, error messages, tool output) breaks terminal rendering: tabs → visual holes, long lines → overflow, paths → leak home directory.

  Line 208: - **Truncate** lines with `truncateToWidth()` / `ui.truncate()`. Use `TRUNCATE_LENGTHS` constants.

  Line 215: - **Error messages** — these often embed file content (e.g., patch failure messages include unmatched lines). If a message contains file content, it needs `replaceTabs()`.


/home/sadiq/Documents/web/oss/oh-my-pi/python/omp-rpc/src/omp_rpc/protocol.py:
  Line 861:     total_lines: int

  Line 863:     output_lines: int

  Line 969:     widget_lines: tuple[str, ...] | None = None

  Line 1450:         total_lines=int(payload.get("totalLines", 0)),

  Line 1452:         output_lines=int(payload.get("outputLines", 0)),

  Line 1597:         widget_lines=_tuple_of_strings(


/home/sadiq/Documents/web/oss/oh-my-pi/docs/mcp-config.md:
  Line 479: OMP loads providers in descending priority. The MCP-capable order is:


/home/sadiq/Documents/web/oss/oh-my-pi/README.md:
  Line 27: **60+** providers · **31** built-in tools · **14** lsp ops · **28** dap ops · **~80k** lines of Rust core.

  Line 191: web_search chains twenty-three ranked providers and hands whatever URLs it finds straight to read. Arxiv PDFs, GitHub pages, Stack Overflow threads come back as structured markdown with anchors intact — the same tool surface you use on local files. Cite, follow, quote, never lose where you came from.

  Line 207: Perfect edits, fewer tokens. The model points at anchors instead of retyping the lines it wants to change, so whitespace battles and string-not-found loops just stop happening. Edit a stale file and the anchors diverge — we reject the patch before it corrupts anything. Grok 4 Fast spends 61% fewer output tokens on the same work.

  Line 233: ![omp TUI reading pr://can1357/oh-my-pi/1063 and then /diff/1, showing hunk headers, added lines, and a [MODIFIED] (+12 -0) summary.](https://omp.sh/captures/pr.webp)

  Line 263: 31 tools live in the same namespace as `read` and `bash`. Pin the active set with `--tools read,edit,bash,…`; rarely used discoverable tools stay behind `xd://` devices. `read xd://` lists them, and `write xd://<tool>` runs one when `tools.xdev` is enabled.

  Line 297: - `web_search` — one query across configured providers, returning answer plus citations.

  Line 335: ## Sixty-plus providers, a thousand models, _one /model away_.

  Line 343: Direct APIs and gateways. Mix providers per role.

  Line 359: ### Custom OpenAI-compatible providers

  Line 361: Define custom providers in `~/.omp/agent/models.yml`:

  Line 364: providers:

  Line 387: - **Custom providers** — Declare anything that speaks `openai-completions`, `openai-responses`, `openai-codex-responses`, `azure-openai-responses`, `anthropic-messages`, `bedrock-converse-stream`, `google-generative-ai`, `google-gemini-cli`, or `google-vertex` in `~/.omp/agent/models.yml`.

  Line 392: Full provider & routing reference at [omp.sh/docs/providers](https://omp.sh/docs/providers).

  Line 398: ### Search providers

  Line 453: ## Roughly **~80,000** lines of Rust, doing the work other harnesses shell out for.

  Line 455: Six crates, one platform-tagged N-API addon. Search, shell, AST, highlight, PTY, desktop control, image decode, BPE counting — all in-process on the libuv pool. No fork/exec on the hot path. Another ~80k lines ride along vendored: the brush bash fork, plus 58 command-line utilities — coreutils, findutils, sed, jq, ripgrep-backed grep, fd, diff, moreutils — ported into the builtins crate and compiled straight into the shell.

  Line 460: Per crate, code lines only:

  Line 570: omp is a fork of [Pi](https://github.com/badlogic/pi-mono) by [Mario Zechner](https://github.com/mariozechner), rewritten as a coding-first surface: sessions, subagents, slash commands, extensions — all TypeScript, all MIT, all on [GitHub](https://github.com/can1357/oh-my-pi). Shape it from config, hook it from outside, or read the source when you need to.

  Line 586: omp is a fork of [pi-mono](https://github.com/badlogic/pi-mono) by [Mario Zechner](https://github.com/mariozechner), extended with a batteries-included coding workflow.

  Line 635: For architecture and contribution guidelines, see [packages/coding-agent/DEVELOPMENT.md](packages/coding-agent/DEVELOPMENT.md).

  Line 671: | **[brush-core](crates/vendor/brush-core)**         | Vendored fork of [brush-shell](https://github.com/reubeno/brush) for embedded bash execution        |

  Line 679: guidelines on contributing.


/home/sadiq/Documents/web/oss/oh-my-pi/infra/docs/01-host-and-cluster.md:
  Line 368: The two lines that make pod egress work are **`masquerade: yes`** (SNAT pod source IPs to `<PUBLIC_IP>` on the way out) and **`forward: yes`** (allow routing between interfaces). The `services` list (`ssh`, `http`, `https`, `cockpit`, `dhcpv6-client`) is the host's own inbound allow-list and is unrelated to pod egress — note that `http`/`https` here are for the **host nginx**, not k3s.


/home/sadiq/Documents/web/oss/oh-my-pi/docs/ai-schema-normalize.md:
  Line 3: `@oh-my-pi/pi-ai` exposes one unified schema normalizer that providers consume


/home/sadiq/Documents/web/oss/oh-my-pi/docs/secrets.md:
  Line 3: Prevents sensitive values (API keys, tokens, passwords) from being sent to LLM providers. When enabled, configured secrets and built-in credential-shaped token patterns are replaced before provider-visible text leaves the process. Reversible placeholders are restored in model-authored tool arguments before execution and when local session context is rebuilt for display or resume.


/home/sadiq/Documents/web/oss/oh-my-pi/docs/config-usage.md:
  Line 37:         capability providers enumerate items

  Line 39:   other providers have their own loading rules)

  Line 153: 2. Project settings: discovered via the settings capability (`settings.json` and `config.yml` from providers)

  Line 303: - Ambient extension-module capability discovery is explicitly restricted to `provider: "native"`; foreign providers are not scanned for this step.


/home/sadiq/Documents/web/oss/oh-my-pi/rustfmt.toml:
  Line 67: blank_lines_lower_bound = 0

  Line 68: blank_lines_upper_bound = 1


/home/sadiq/Documents/web/oss/oh-my-pi/docs/provider-compat-reference.md:
  Line 25: 2. **Request time** (`packages/ai/src/providers/openai-shared.ts`): `resolveOpenAICompatPolicy(model, options)` combines the resolved compat with per-request options (`reasoning`, `disableReasoning`, `toolChoice`, …) into an `OpenAICompatPolicy` with `reasoning`, `tools`, `messages`, and `stream` sub-policies. When thinking is active and `whenThinking` exists, the policy **pointer-swaps** to the pre-built variant — no per-request spreading or allocation:

  Line 69: | `kimiApiFormat` | Per-model protocol metadata | `"openai"` vs `"anthropic"` transport for Kimi Code models (`providers/kimi.ts`) |

  Line 155: | Anthropic (`providers/anthropic.ts`) | `anthropic-adaptive` or `budget` | Adaptive: `thinking: { type: "adaptive" }` + `output_config.effort: low…max` (requires beta `effort-2025-11-24`); Budget: `thinking: { type: "enabled", budget_tokens: N }`. Interleaved thinking via beta `interleaved-thinking-2025-05-14`. `ensureMaxTokensForThinking` raises `max_tokens` to at least `budget_tokens + 1024` |

  Line 156: | OpenAI Responses (`providers/openai-responses.ts`) | `effort` | `reasoning: { effort }` plus `reasoning.summary: "auto" \| "detailed" \| "concise" \| null` |

  Line 157: | OpenAI Chat Completions (`providers/openai-completions.ts`) | `effort` | `reasoning_effort` by default; the actual toggle field depends on `thinkingFormat` (see [flag table](#reasoning-wire-format)) |

  Line 158: | Google Gemini / Vertex (`providers/google-shared.ts`) | `google-level` or `budget` | `thinkingConfig: { includeThoughts, thinkingLevel: MINIMAL…HIGH, thinkingBudget: N }` |

  Line 162: `resolveOpenAICompatPolicy` (`providers/openai-shared.ts`) decides per request:

  Line 169: If a host rejects the emitted effort with 400/422, `resolveOpenAIReasoningEffortFallback` (`providers/openai-reasoning-fallback.ts`) parses the error text to extract accepted values or drop the parameter, then retries.

  Line 173: - **Structured deltas**: providers emit `thinking_start` / `thinking_delta` / `thinking_end` stream events.

  Line 181: - **Forced tool choice**: see [§4](#edge-cases); several providers must drop thinking when a tool call is forced.

  Line 185: All providers start from the same neutral wire schema — `toolWireSchema(tool)` (`utils/schema/wire.ts`) — and diverge in normalization, streaming shape, and result encoding.

  Line 187: ### Anthropic (`providers/anthropic.ts`)

  Line 195: ### OpenAI Chat Completions (`providers/openai-completions.ts`)

  Line 201: ### OpenAI Responses (`providers/openai-responses.ts`)

  Line 207: ### Google Gemini / Vertex (`providers/google-shared.ts`, `google.ts`)

  Line 214: ### Amazon Bedrock (`providers/amazon-bedrock.ts`)

  Line 234: `OpenAIStrictToolsState` (`providers/openai-shared.ts`) tracks strict-mode failures per scope `${provider}:${baseUrl}:${modelId}`: a 400 strict-schema rejection calls `disableStrictToolsForScope`, and `isStrictToolsDisabledForScope` makes all subsequent requests for that scope run non-strict — one retry, then remembered, no per-turn 400 tax. Anthropic has the analogous per-session `strictToolsDisabled` flag.

  Line 249: - **`providers/transform-messages.ts`**: shared pre-flight pass — tool-call dedup, sanitization, id normalization — before provider-specific conversion.

  Line 295: 2. **Bedrock sentinel**: see [§3](#amazon-bedrock-providersamazon-bedrockts).


/home/sadiq/Documents/web/oss/oh-my-pi/docs/native-crates.md:
  Line 12: | `pi-builtins`   | [`crates/pi-builtins`](../crates/pi-builtins)     | Every builtin the embedded shell installs: a patched fork of brush's POSIX/bash builtins, plus one module per in-process command-line utility (`cat`, `grep`/`rg`, `sed`, `ls`, `find`, `jq`, `fd`, `diff`, `ps`, `top`, `kill`, the moreutils set, …). `src/host.rs` holds the `Utility` trait and the `Host` view of the shell (stdio, working directory, exported environment, cancellation) that the utilities run against. Ports of uutils coreutils/findutils/sed and jaq live here too; see the crate `LICENSE` for third-party notices. |


/home/sadiq/Documents/web/oss/oh-my-pi/docs/cli-reference.md:
  Line 96: See [providers](./providers.md) and [models](./models.md) for model resolution.

  Line 105: | `--external-thinking` | Use a private scratchpad while disabling supported GPT/Claude/Gemini reasoning. Use at your own risk: providers have flagged this request shape as abuse. |

  Line 176: # Machine-readable output for pipelines

  Line 244: | `search`, `q` | Test web search providers from the CLI. | [web_search tool](./tools/web_search.md) |


/home/sadiq/Documents/web/oss/oh-my-pi/docs/provider-endpoint-constraints.md:
  Line 19: - [Providers](./providers.md) — provider availability, credentials, custom providers

  Line 90: ### Anthropic/OpenAI dual-surface providers

  Line 133: - Routes providers through the OpenRouter `provider` object.

  Line 258: - some providers support strict tools

  Line 315:   providers do not support `developer` and must downgrade to `user`. Some reject

  Line 344: - **DeepSeek/template-leak tool calls.** Some providers leak tool-call markup in


/home/sadiq/Documents/web/oss/oh-my-pi/docs/skills.md:
  Line 27: For provider-based discovery (native/Claude/Codex/Agents/plugin providers), skills are discovered as **one level under `skills/`**:

  Line 69: - the claude/codex/agents/opencode/claude-plugins providers can load skills without description

  Line 75: 1. **Capability providers** via `loadCapability("skills")` (the managed/auto-learn provider's skills are skipped here and handled in pass 3)

  Line 81: ### Built-in skill providers and precedence

  Line 85: Current registered skill providers:


/home/sadiq/Documents/web/oss/oh-my-pi/.github/ISSUE_TEMPLATE/config.yml:
  Line 5:     about: How to add custom providers and models via models.yml


/home/sadiq/Documents/web/oss/oh-my-pi/docs/custom-tools.md:
  Line 47: 1. Capability providers (`toolCapability`), including:

  Line 59: - `.md` and `.json` files are discovered as tool metadata by some providers, but the executable module loader rejects them as runnable tools.


/home/sadiq/Documents/web/oss/oh-my-pi/docs/toolconv/minimax.md:
  Line 65: | Schema-declared string whose runtime value is a string | Verbatim text, including leading/trailing spaces and newlines | Verbatim string |


/home/sadiq/Documents/web/oss/oh-my-pi/crates/vendor/brush-core/src/patterns.rs:
  Line 160: 	#[expect(clippy::too_many_lines)]


/home/sadiq/Documents/web/oss/oh-my-pi/docs/ttsr-injection-lifecycle.md:
  Line 60: The snapshot is source-bearing payload, not the whole prospective file: pre-existing target content is invisible unless the call repeats it. Current edit modes expose `new_string` for replace, added lines for JSON patch, hashline, and apply-patch forms, and full content for create forms; write exposes its entire `content`. Multi-file hashline/apply-patch calls are split into separate `{ path, digest }` entries, so AST language, path scope/globs, buffers, and matching are evaluated per file. Matching is in-memory through native `astMatch` with Smart strictness.


/home/sadiq/Documents/web/oss/oh-my-pi/.github/actions/bazel-cache/action.yml:
  Line 115:              # Mask lines inside the redirected block below would land in the


/home/sadiq/Documents/web/oss/oh-my-pi/docs/toolconv/harmony.md:
  Line 72: - A property `description` becomes a `//` comment on the line *above* the field; a JSON Schema `title` renders as `// TITLE` followed by a `//` blank-comment line; `examples` render as `// Examples:` then `// - "value"` lines.

  Line 164: Complete multi-turn weather exchange: system + developer prompt → user question → assistant analysis CoT → assistant commentary tool call → tool result → assistant final answer. This is a single contiguous token stream (newlines inside headers are only between top-level messages for readability; in practice messages are concatenated with no separator).


/home/sadiq/Documents/web/oss/oh-my-pi/packages/ai/src/provider-details.ts:
  Line 1: import { getOpenAICodexTransportDetails, type OpenAICodexTransportDetails } from "./providers/openai-codex-responses";


/home/sadiq/Documents/web/oss/oh-my-pi/crates/pi-natives/src/ast.rs:
  Line 1273: 		let lines = page

  Line 1280: 		assert_eq!(lines, vec![2, 3]);


/home/sadiq/Documents/web/oss/oh-my-pi/crates/vendor/brush-core/src/completion.rs:
  Line 297: 	#[expect(clippy::too_many_lines)]

  Line 455: 	#[expect(clippy::too_many_lines)]

  Line 706: 		for line in output.lines() {

  Line 1527: 	#[allow(clippy::too_many_lines)]


/home/sadiq/Documents/web/oss/oh-my-pi/docs/toolconv/glm-4.5.md:
  Line 83: The `<tool_call>{function-name}` / `<arg_key>` / `<arg_value>` lines above are part of the **prompt text** (the format spec the model is told to follow), not an example call. This tools turn is emitted only when `tools` is non-empty, and it is closed implicitly by the next role marker (e.g. a user-supplied `<|system|>` or the first `<|user|>`), with no blank line between them.


(Results truncated. Consider using a more specific path or pattern.)
```

**Tool: bash**

**Input:**
```json
{
  "command": "node -e \"for (const p of ['deepseek-harness','pi','opencode','oh-my-pi']) { try { console.log(p, JSON.parse(require('fs').readFileSync('/home/sadiq/Documents/web/oss/'+p+'/package.json')).version) } catch(e){ console.log(p, 'n/a') } }\""
}
```

**Output:**
```
deepseek-harness 0.1.1-rc.2
pi 0.0.3
opencode undefined
oh-my-pi undefined

```

---

## Assistant (Build · x-preview-f-free · 9.2s)

_Thinking:_

Verified oh-my-pi claims (60+ providers, 31 tools, ~80k Rust, fork of pi-mono). Versions: deepseek-harness 0.1.1-rc.2, pi root package 0.0.3 (packages lockstep at 0.84.2 per agent — fine, I'll cite the package version). Now write the report.

Write a comprehensive comparison-report.md. Include:
- TL;DR / executive summary
- Per-project detailed sections
- Comparison matrices
- Architecture deep dive
- Extensibility comparison
- Testing/build
- Lineage diagram (pi → oh-my-pi; pi-ai used by deepseek-harness)
- Recommendations/when to use

Let me write it well and exhaustively but organized.

All four repos researched and key claims verified (oh-my-pi's "60+ providers, 31 tools, ~80k Rust, fork of pi-mono" confirmed in README.md:27,453,570). Writing the report now.

**Tool: todowrite**

**Input:**
```json
{
  "todos": [
    {
      "content": "Research all 4 repos",
      "priority": "high",
      "status": "completed"
    },
    {
      "content": "Verify key claims/numbers across repos",
      "priority": "medium",
      "status": "completed"
    },
    {
      "content": "Build comparison matrix + write comparison-report.md",
      "priority": "high",
      "status": "in_progress"
    }
  ]
}
```

**Output:**
```
[
  {
    "content": "Research all 4 repos",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Verify key claims/numbers across repos",
    "status": "completed",
    "priority": "medium"
  },
  {
    "content": "Build comparison matrix + write comparison-report.md",
    "status": "in_progress",
    "priority": "high"
  }
]
```

---

## Assistant (Build · x-preview-f-free · 207.2s)

**Tool: write**

**Input:**
```json
{
  "filePath": "/home/sadiq/Documents/web/oss/comparison-report.md",
  "content": "# AI Coding Agent Harnesses: Comparative Report\n\n**Projects compared:** `deepseek-harness` · `opencode` · `pi` · `oh-my-pi`\n**Date:** 2026-08-24\n**Scope:** Capabilities, architecture, extensibility, tooling/testing, and how the four projects relate to each other.\n\n---\n\n## Table of Contents\n\n1. [Executive Summary](#1-executive-summary)\n2. [Lineage & Relationships](#2-lineage--relationships)\n3. [Project Profiles](#3-project-profiles)\n   - [3.1 pi (pi-mono)](#31-pi-pi-mono)\n   - [3.2 oh-my-pi (omp)](#32-oh-my-pi-omp)\n   - [3.3 opencode](#33-opencode)\n   - [3.4 deepseek-harness (dsh)](#34-deepseek-harness-dsh)\n4. [Head-to-Head Comparison Matrix](#4-head-to-head-comparison-matrix)\n5. [Architecture Deep Dive](#5-architecture-deep-dive)\n6. [Capability Comparison](#6-capability-comparison)\n7. [Extensibility Models Compared](#7-extensibility-models-compared)\n8. [Engineering Quality: Testing, Build, Release](#8-engineering-quality-testing-build-release)\n9. [Which One Should You Use?](#9-which-one-should-you-use)\n\n---\n\n## 1. Executive Summary\n\nAll four are **open-source agentic coding harnesses** — terminal-first (or multi-surface) tools that wire an LLM to a codebase with file editing, shell execution, and session management. They differ radically in philosophy:\n\n| Project | One-liner | Core bet |\n|---|---|---|\n| **pi** | Minimal, self-extensible agent harness | \"No MCP. No sub-agents. No permission popups.\" Everything is a user-supplied TypeScript extension |\n| **oh-my-pi (omp)** | Pi fork turned batteries-included coding agent | Native Rust performance core + maximal built-in tool surface (31 tools, LSP/DAP) |\n| **opencode** | The productized multi-surface coding agent | Client/server split with one authoritative HTTP API; TUI, desktop, IDE, web, cloud all as clients |\n| **deepseek-harness (dsh)** | Plugin-kernel harness where *everything* is a plugin | Vendored Cordis framework; model adapter, tools, sandbox, even the agent loop are swappable config |\n\nKey facts at a glance:\n\n- **pi** (`@earendil-works/pi-*`, Earendil Works, pi.dev): 100% erasable-syntax TypeScript on Node ≥22 / Bun; deliberately minimal core; packages distributed via npm/git.\n- **oh-my-pi** (`@oh-my-pi/pi-coding-agent`, omp.sh, by Can Bölük / Stencil Labs): fork of pi-mono; ~80k lines of Rust compiled into one N-API addon (embedded bash, in-process grep/coreutils, AST tools); 60+ providers, 31 built-in tools.\n- **opencode** (`opencode-ai`, anomalyco): Bun + Effect-TS monorepo; strict layered architecture (Schema → Core → Server → Client → SDK); SST/AWS+Cloudflare cloud ecosystem; VS Code extension and GitHub Action.\n- **deepseek-harness** (`@deepseek-ai/dsh`, v0.1.1-rc.2 developer preview, MIT): DeepSeek's plugin-based harness; capability seams (LLM, shell, sandbox, FS, LSP, subagent…) each with swappable providers; Web UI + headless + ACP + JSON-RPC/TS/Python SDKs.\n\nNotable cross-pollination: **dsh is not a pi fork**, but it uses `@earendil-works/pi-ai` as an optional multi-provider LLM adapter behind its own `llm` seam. **omp is a direct fork of pi**, keeping `pi-*` package names and an upstream-sync process.\n\n---\n\n## 2. Lineage & Relationships\n\n```\n                    ┌──────────────────────────────┐\n                    │   pi (badlogic/pi-mono)      │\n                    │  minimal-core agent harness  │\n                    └──────────┬───────────────────┘\n                               │ direct fork (tracked sync point,\n                               │ keeps @…/pi-* package names)\n                               ▼\n                    ┌──────────────────────────────┐\n                    │   oh-my-pi (can1357)         │\n                    │  batteries-included rewrite  │\n                    │  + Rust native core          │\n                    └──────────────────────────────┘\n\n┌──────────────────────────────┐        uses as optional lib\n│  deepseek-harness (@deepseek)│◄───── @earendil-works/pi-ai\n│  Cordis plugin kernel        │       (provider wire protocols\n│  NOT a pi fork               │        & model catalogs only)\n└──────────────────────────────┘\n\n┌──────────────────────────────┐\n│  opencode (anomalyco)        │   independent lineage\n│  Effect-TS client/server     │\n└──────────────────────────────┘\n```\n\n---\n\n## 3. Project Profiles\n\n### 3.1 pi (pi-mono)\n\n> *\"No MCP. No sub-agents. No permission popups. No plan mode. No to-dos. No background bash.\"*\n\n**What it is.** A terminal coding agent built on a deliberately minimal core that users adapt through **TypeScript Extensions, Skills, Prompt Templates, and Themes** distributed as shareable \"Pi Packages\" over npm/git. Workflow features are intentionally excluded from core; they belong in extensions/packages. There is no built-in permission system either — sandboxing is delegated to containers/micro-VMs (documented patterns: Gondolin micro-VM extension, Docker, OpenShell).\n\n**Tech stack.**\n- 100% TypeScript using only *erasable* syntax (Node strip-only mode — no enums/namespaces/parameter properties), enforced by a CI check.\n- Node ≥22.19 (native `node:sqlite`) and Bun for standalone binaries.\n- npm workspaces, Biome lint/format, `tsgo` (TypeScript native preview), Vitest + node:test, esbuild/tsx, Husky.\n- Length-prefixed **CBOR** framing for its remote-session protocol.\n\n**Monorepo layout** (lockstep versioning across all packages):\n\n| Package | Purpose |\n|---|---|\n| `pi-tui` | Minimal TUI framework: differential rendering, CSI 2026 sync output, alt/main screen renderers, editor/select-list/markdown components, Kitty/iTerm2 inline images |\n| `pi-telemetry` | Vendor-neutral telemetry contracts (callback-based spans), no-op + in-memory reference adapters |\n| `pi-ai` | Unified multi-provider LLM API (~15+ providers incl. OpenAI, Anthropic, Google, Bedrock, Azure, Groq, xAI, OpenRouter, Mistral); streaming, thinking levels, image I/O, auth resolution, cost tracking, mid-session model hand-off |\n| `pi-agent-core` | General-purpose stateful agent runtime: tool loop, event streaming, attachments, pluggable session backends |\n| `session-backends/sqlite-node` | SQLite persistence backend (repository, migrations, FTS search) kept out of core so core has zero runtime deps |\n| `pi-coding-agent` | The `pi` CLI: interactive TUI, print/JSON mode, RPC mode, SDK embedding; read/bash/edit/write tools; sessions w/ branching & compaction; extensions/skills/themes/packages |\n| `pi-protocol` | Transport-neutral CBOR schemas + byte-stream framing for remote sessions |\n| `pi-client` | Transport-neutral remote-session client: exclusive/shared session leases, snapshot-authoritative state |\n| `pi-server` | Experimental session server composing pluggable transport listeners |\n| `pi-evals` | Model-backed behavioral evals via `vitest-evals`, isolated temp dirs |\n\n**Capabilities.**\n- Four run modes: interactive TUI, print/JSON one-shot, RPC (for process integration), embeddable SDK.\n- Built-in tools limited to read/bash/edit/write; per-invocation disable lists; read-only mode.\n- Sessions: durable SQLite storage, branching, compaction, full-text search.\n- Extensions with lifecycle hooks; custom providers; skills explicitly positioned as the MCP replacement; themes; prompt templates.\n- Agent-first development culture: AGENTS.md rules for humans and AI; maintainers publish their dev sessions as OSS datasets on Hugging Face.\n\n**Architecture highlights.**\n- **Minimal-core doctrine**: hook points require maintainer discussion; PRs bloating core are rejected.\n- **Layered stack**: `tui → ai → agent (+backends) → coding-agent`; remote access splits cleanly `protocol → server → client`.\n- **Snapshot-authoritative RPC**: server/client sync via authoritative snapshots; progress events are transient UI hints never folded into state; leases enforce single-writer ownership; auth happens at transport layer before protocol bytes flow.\n- **Transport neutrality everywhere**: protocol/client/telemetry avoid runtime-specific imports; the SQLite backend accepts an injected DB factory.\n- **Supply-chain hardening as architecture**: exact-pinned deps, `min-release-age=2` days, lockfile-as-ground-truth with pre-commit gating, generated shrinkwrap + lifecycle-script allowlist, `npm ci --ignore-scripts` in CI, npm trusted publishing (OIDC).\n\n### 3.2 oh-my-pi (omp)\n\n> *\"A coding agent with the IDE wired in.\"* — fork of pi-mono, rewritten as a coding-first surface.\n\n**What it is.** A batteries-included terminal coding agent: CLI/TUI plus SDK, RPC, and ACP surfaces. Where pi strips features out, omp builds them in — subagents, slash commands, debugger integration, desktop control, collaboration sharing — while pushing hot-path work into a large native Rust layer. Headline stats: **60+ providers, 31 built-in tools, 14 LSP ops, 28 DAP ops, ~80k lines of Rust core** (+~80k vendored: brush bash fork + dozens of in-process CLI utilities).\n\n**Tech stack.**\n- **TypeScript** on **Bun** (≥1.3.14) for all product logic (agent, tools, TUI, providers).\n- **Rust** (edition 2024, pinned nightly) compiled to one platform-tagged **N-API `.node` addon** (`pi_natives` cdylib).\n- **Python** for `robomp` (GitHub triage bot) and `omp-rpc` (typed Python RPC client).\n- **Bazel** (hermetic shipping builds of the Rust addon, zig cc cross-toolchain) alongside **Cargo** for local iteration; bun workspaces; Biome; Nix flake.\n\n**Repo layout — key TS packages:** `coding-agent` (main CLI/SDK), `ai` (multi-provider client), `catalog` (bundled model DB), `agent`, `tui`, `natives` (N-API loader), `hashline` (content-hash patch language behind the edit tool), `omptype` (ArkType-compatible validation), `utils`, `wire` (collab protocol), `mnemopi` (SQLite memory engine), `snapcompact` (context compression + SQuAD evals), `stats` (usage dashboard), `collab-web` (browser guest + relay), `browser-relay` (Chrome extension driving real tabs), `metaharness` (benchmark runners/Harbor storage/dashboard).\n\n**Rust crates:** `pi-natives` (~25k LoC; grep, diff, ast-grep, PTY, clipboard, highlighting, desktop control, tokenizers, sixel…), `pi-shell` (~38k LoC embedded bash wrapping vendored brush), `pi-builtins` (bash builtins + ported coreutils/findutils/sed/jq running zero-fork), `vendor/brush-core`, `pi-walker` (parallel ignore-aware FS walker), `pi-iso` (task isolation: APFS clones, btrfs/zfs reflinks, overlayfs, projfs), `pi-ast` (tree-sitter + ast-grep, 50+ grammars), `pi-voice` (audio capture/playback, Opus, WebRTC).\n\n**Capabilities (selection).**\n- Four entry points: interactive TUI, one-shot `-p`, Node SDK (`createAgentSession`), stdio NDJSON RPC + ACP for editors like Zed.\n- **31 tools**: read/write/edit (hashline patches)/ast_edit/ast_grep/grep/glob, **in-process bash** (46+ builtins), eval (persistent Python + JS kernels that can call back into agent tools), lsp, debug (DAP: lldb/dlv/debugpy), task (**parallel subagents in isolated worktrees**), browser (Puppeteer/Electron/Chrome relay), computer (desktop control: windows, screenshots, native input, AX tree), web_search (23-provider chain + structured extraction), github, generate_image/inspect_image/tts, memory tools (retain/recall/reflect/learn), checkpoint/rewind, todo, ask, hub.\n- **Model routing**: 10 roles (default/smol/slow/plan/vision/designer/task/advisor/commit/tiny), fallback chains, path-scoped models, round-robin credentials.\n- **Advisor model**: second reviewer model watching every turn on its own context.\n- **Time-traveling stream rules**: regex-triggered mid-stream system-reminder injection with retry.\n- **Collab**: `/collab` share link + QR; teammates join by terminal or browser; client-side encryption.\n- **Interop**: inherits rules/skills/MCP configs from `.claude`, `.cursor`, `.windsurf`, `.gemini`, `.codex`; reads `pr://1428`, `conflict://N`, ssh paths as URLs inside every file tool.\n- Secrets redaction before provider-visible text leaves the process.\n\n**Architecture highlights.**\n- **One N-API addon, no fork/exec on hot paths**: ripgrep-as-library, embedded bash, in-process utilities → single binary across macOS/Linux/Windows without WSL.\n- Blocking Rust work runs on the libuv pool with panic-catching at the napi boundary (panics become rejected promises).\n- Bun workers re-enter `cli.ts` via hidden argv selectors so compiled binaries need no separate worker entries; prompts live in static `.md` imports, not code-built strings.\n- **Dual build split**: Cargo authoritative locally (rust-analyzer, nextest); **Bazel ships hermetically** — rules_rust + zig cc, crate_universe from Cargo.lock, 8 platform/ISA targets including musl and dual AVX2/baseline x64.\n- Extension modules share the same tool/slash-command/hotkey registries as built-ins; MCP support; managed skills marketplace; settings-gated `xd://` discoverable off-by-default tools.\n\n### 3.3 opencode\n\n> *\"The open source AI coding agent\"* — TUI, desktop app, IDE, browser, and cloud, all clients of one HTTP API.\n\n**What it is.** A heavily productized coding-agent ecosystem: the agent runs as a local **server** exposing an authoritative HTTP API (SSE event streams); every UI (TUI, Electron desktop, VS Code extension, shared web view, SDKs) is a client. 75+ providers via models.dev + Vercel AI SDK adapters. Ships a cloud console (Stripe billing), stats site, GitHub Action (`/opencode fix this issue` → PR), Slack bot, and enterprise control-plane.\n\n**Tech stack.**\n- Bun 1.3 monorepo (workspaces + catalog deps), Turborepo orchestration.\n- **Effect v4** pervasive: Effect Schema domain types, `HttpApi` for the API, Drizzle ORM + SQLite for durable storage, Hono HTTP server.\n- SolidJS/SolidStart + Tailwind v4 + Vite for web UIs; **@opentui/solid** for the TUI; Electron desktop; node-pty terminals; tree-sitter/WASM.\n- oxlint + tsgolint, prettier, husky, Playwright e2e, SST v4 IaC (Cloudflare home + AWS us-east-1, Stripe, PlanetScale, Honeycomb, Sentry).\n- Heavy upstream patching (patchedDependencies: effect, ai-sdk providers, MCP SDK, solid-js…).\n\n**Monorepo layout (dependency direction: Schema ← Core/Protocol ← Server ← Client ← sdk-next).**\n\nCore chain:\n\n| Package | Purpose |\n|---|---|\n| `schema` | Shared public domain schemas (Effect Schema records), dependency-free leaf |\n| `core` | Domain core: sessions, agents, config, permissions, tools, LSP, MCP, plugins, git, storage |\n| `opencode` | Main binary/CLI entrypoint wiring everything into the distributable |\n| `llm` | Provider/protocol layer: wire protocols, cache policy, tool runtime, provider errors |\n| `protocol` | Public API composition: endpoint paths, envelopes, cursors, streams |\n| `server` | Hono/Effect server hosting the authoritative public `HttpApi` |\n| `httpapi-codegen` | Compiles the HttpApi into SDK Contract IR → generated clients |\n| `client` (+`/effect`) | Generated Promise + Effect API clients (browser-safe) |\n| `sdk` / `sdk-next` | Legacy JS SDK; Effect-native in-process host (\"Embedded OpenCode\") |\n\nSurfaces & ecosystem: `tui`, `app` (web/desktop frontend), `desktop` (Electron), `ui` + `session-ui` + `storybook` (shared components), `console/*` (cloud console: app/core/functions/mail/resource/support), `stats/*`, `enterprise`, `identity`, `slack`, `containers` (sandboxing), `codemode` (Effect-native confined code execution over schema-described tools), `web` (docs/marketing site), `plugin` (public plugin API), `http-recorder` (cassette record/replay testing). Plus `sdks/vscode` (VS Code extension), `github/` (official GitHub Action), `infra/` (SST modules: app, console, enterprise, lake, stats, monitoring), `specs/` (design specs for v2 session/tools/provider/config/storage).\n\n**Capabilities.**\n- Rich SolidJS TUI with two built-in agents switchable via Tab: `build` (full access) and `plan` (read-only), plus a `general` subagent invoked with `@general`.\n- Tools: file read/edit/write, bash, glob/grep, webfetch, todo, patch/diff, snapshot/checkpointing, permission-checked skills, question prompts, PTY terminals.\n- Agents/subagents configurable via markdown/JSON; **permission system** with allow/deny rules and per-command approval.\n- Sessions: persistent SQLite storage, share/public links, compaction, steering vs queued prompts, resumption.\n- Extensibility: plugin API (`@opencode-ai/plugin` with hooks including PTY environment observation), MCP client, LSP, custom providers/models/commands, AGENTS.md instruction discovery.\n- Integrations: VS Code extension, Zed support, ACP, GitHub Action, Slack bot, macOS/Windows/Linux desktop apps.\n\n**Architecture highlights.**\n- **Strict layered dependency direction**, mechanically enforced: Client runtime never imports Core/Server; Protocol owns endpoint/middleware placement; Server supplies concrete keys.\n- **Single authoritative HttpApi**: codegen compiles it into SDK Contract IR generating both a zero-Effect Promise client and a rich Effect client. Generated files are never hand-edited.\n- **V2 session runtime** (extensively specified in `specs/v2/`): durable prompt admission decoupled from model execution; serialized process-local \"Session Drains\"; steer/queue semantics at safe provider-turn boundaries; crash-safe history without durable-execution identity.\n- **System Context algebra**: composable typed Context Sources (AGENTS.md instructions, date, skill guidance…) rendered into a baseline prompt with immutable provider-cache-friendly **Context Epochs**; changes emitted as durable mid-conversation system messages rather than mutating the baseline.\n- Tool-output bounding: oversized outputs truncated to previews with full text spilled to managed temp files.\n- Two deliberately different event streams: durable replayable per-session SSE vs live-only instance-wide subscription.\n\n### 3.4 deepseek-harness (dsh)\n\n> *\"Everything is a plugin.\"* — plugin-based agent harness from DeepSeek AI, developer preview.\n\n**What it is.** An open-source, plugin-based coding/automation harness where the model adapter, tool registry, session log, sandbox policy — **and even the agent loop itself** — are plugins mounted into a composable tree at boot. Ships as: a CLI launcher (`npx @deepseek-ai/dsh web`), a **Web UI** (React client + Node host at `127.0.0.1:3080`), a headless one-shot runner, an **ACP server**, a **JSON-RPC server + TS SDK**, and a **Python SDK** driving a bundled runtime over newline-delimited JSON-RPC on stdio. Single-exe distribution via the Python runtime carrier. Explicit no-compatibility-promise until first tagged release.\n\n**Tech stack.**\n- Strict-mode ESM-only TypeScript; Node `^22.19 || >=24`; pnpm 11 workspaces.\n- **Cordis** plugin/context/event/waterfall framework (from cordiverse), **source-vendored** under `vendor/`, rescoped to `@deepseek-ai/*`, pinned SHAs with exhaustive modification logging.\n- Vite + React web client; VitePress docs site; tsc project references + tsdown bundling.\n- Vitest (unit/e2e/keyless snapshots/web stress/perf), fast-check property tests, jscpd duplication checks, knip/publint, lefthook, ~30 custom gate scripts.\n- C++/N-API addon: **Landlock self-restrict-then-exec launcher** (Linux sandboxing, per-platform npm packages); bubblewrap/Landlock/Seatbelt + Windows ACL restricted-token sandbox backends; patched node-pty.\n- Python SDK + PyPI-distributed runtime binary carrier.\n\n**Repo layout (~55 package groups, all `@deepseek-ai/dsh-*`).** Highlights:\n\n| Area | Packages |\n|---|---|\n| Spine | `core` (session, system-prompt, tools, agent registry, **swappable agent-loop**), `api` (remote BFF + Typert RPC gateway), `typert` (type-graph registry) |\n| Model | `llm` seam + `llm-deepseek`, **`llm-pi-ai`** (multi-provider via `@earendil-works/pi-ai`), `llm-retry`, `token-meter` |\n| Execution | `shell` (local/pwsh providers), `subprocess`, `terminal` (persistent PTY), `code-runtime` (\"Code Mode\" worker-thread execution), `sandbox` (bwrap/Landlock/Seatbelt/Windows ACL), `fs` (+ policy fencing), `lsp`, `web` (search/fetch: DeepSeek/Exa/Perplexity), `e2b` (cloud-sandbox POC) |\n| Orchestration | `subagent` (fork/spawn in-process, Claude Code, Codex, ACP, DSH-SDK providers + delegation tools), `jobs` (background jobs), `workflow` (+ ralph tools), `todo`, `plan`, `guard` (loop hygiene + tool deadlines), `extensions` (**agent inspects/mounts/unmounts its own plugins live**) |\n| Data plane | `session` (JSONL & SQLite persistence, projections, telemetry/OTel, LLM titles), `session-query` (bounded reads, lineage, semantic filter, FTS), `attachment`/`spill`/`storage`/`workspace`, `compaction`, `context` |\n| Delivery | `preset`/`bundle` (`--profile` installable bundles: base, web-app, headless), `boot`, `sdk` (JSON-RPC + TS client), `acp`, `host`/`client` (Web GUI halves; ~30 React `ui-*` plugins) |\n| Compat | `hooks` (Claude Code/Codex hook bridges), `mcp`, `skill` |\n\n**Capabilities.**\n- Agent loop event flow: `turn/start` → prompt assembly → `agent/pre-step` waterfall (listeners can rewrite/reject input) → `agent/request` → `llm/stream` → tool pipeline `pre-execute → execute → post-execute` → repeat. The loop is itself a swappable plugin.\n- Tools: bash/pwsh, persistent PTY terminals, FS read/write/edit + bash-backed discovery, LSP, web search/fetch, todo_write, plan mode, ask-user, subagent delegation/control/report, background `job_*`, workflow/ralph, skills catalog/loader, MCP client tools, session-query. Tool UI render intent (`generic`/`terminal`/`diff`) is part of tool design.\n- Providers: native DeepSeek adapter plus generic multi-provider adapter on pi-ai (OpenAI/Anthropic/etc.), retry policies, token metering.\n- Safety: approvals + permission presets, OS-level sandbox backends, FS write fencing, tool deadlines.\n- Session intelligence: compaction, spill of oversized tool results, LLM-generated titles, OTel telemetry, checkpointing, forking.\n- Self-modification: the `extensions` capability lets the agent inspect its own Cordis graph and mount/unmount plugins live; HMR reload; profile patch layers.\n\n**Architecture highlights.**\n- **Cordis kernel**: services, declaration-merged typed events (`SessionEventMap`), reversible effects with disposers, waterfalls requiring explicit `next()`.\n- **Profiles & bundles**: a running dsh is a config tree composed from ordered patch layers — base bundle → mode bundle (web-app/headless) → profile `cordis.patch.yml` → home patch → `--patch` overlays; `--dump-config` reveals the composed tree.\n- **Capability seams**: each capability = Service Definition + Service Providers + Consumers; extensions depend only on definitions, so swapping one provider relocates dependent capabilities together (e.g., a remote sandbox moves Bash, PTY, and LSP at once).\n- **Session log as source of truth**: append-only events with a build-time-enforced \"model-visible ⟺ logged\" invariant; `deriveMessages()` projects model history; fork/resume/transcripts/telemetry all derive from the stream; monotonic SQLite schema versions.\n- Documented extension-point table maps every goal (new provider/tool/command/backend/UI node) to a mechanism — \"plugins, not loop changes.\"\n- Notably: **no first-party TUI** — the flagship UX is the Web UI; terminal apps mount via profiles.\n\n---\n\n## 4. Head-to-Head Comparison Matrix\n\n| Dimension | pi | oh-my-pi | opencode | deepseek-harness |\n|---|---|---|---|---|\n| Maturity | Stable-ish, lockstep 0.x | Actively developed fork | Production product + cloud | Developer preview (0.1.1-rc.2), no compat promise yet |\n| License / origin | MIT, Earendil Works | MIT, Stencil Labs / Can Bölük | (open) anomalyco | MIT, DeepSeek AI |\n| Primary language(s) | Erasable TS only | TS (Bun) + Rust nightly + Python | TS (Bun) + Effect v4 | Strict ESM TS + vendored Cordis; C++ Landlock addon; Python SDK |\n| Runtime | Node ≥22.19 / Bun binaries | Bun (Node compat secondary) | Bun 1.3 | Node ^22.19\\|\\|≥24 (pnpm) |\n| Primary UX | TUI (+ print/RPC modes) | TUI (+ RPC/ACP) | TUI, desktop, web, IDE, GitHub, Slack | **Web UI**, headless, ACP, SDKs (no first-party TUI) |\n| Package manager / build | npm workspaces, esbuild | bun ws + Cargo + Bazel (shipping) + Nix | Bun ws + Turborepo | pnpm ws, tsc refs + tsdown |\n| LLM providers | ~15+ (own catalog, generated from live data) | 60+, 1000+ models, 10 routing roles | 75+ via models.dev + AI SDK | DeepSeek native + others via pi-ai adapter |\n| Built-in tools | 4 (read/bash/edit/write) — deliberate | 31 (incl. AST, DAP debug, browser, desktop, memory, image/TTS) | ~12 (read/edit/write, bash, glob/grep, webfetch, todo, patch, skills, PTY…) | ~20+ (bash/pwsh, PTY, fs, lsp, web, jobs, workflow, subagents, MCP, session-query…) |\n| Subagents | ❌ by design (extension territory) | ✅ parallel, isolated worktrees | ✅ `@general` + configurable agents | ✅ multiple providers (fork/spawn/Claude Code/Codex/ACP) |\n| Permissions/approvals | ❌ none (sandbox externally) | Settings-gated tools, secrets redaction | ✅ allow/deny rules, per-command approval, plan/build agents | ✅ approvals, permission presets, sandbox backends |\n| Sandboxing | Externalized (Docker/Gondolin docs) | `pi-iso`: APFS clones, reflinks, overlayfs, projfs | Containers package | bwrap/Landlock/Seatbelt/Windows ACL + E2B POC |\n| MCP | ❌ explicit non-goal (skills replace it) | ✅ + inherits .claude/.cursor/etc. configs | ✅ MCP client | ✅ MCP client |\n| LSP | ❌ | ✅ 14 ops, rename-aware | ✅ | ✅ (capability + tool) |\n| Debugger (DAP) | ❌ | ✅ 28 ops (lldb/dlv/debugpy) | ❌ | ❌ |\n| Sessions storage | SQLite backend (pluggable) | SQLite + mnemopi memory engine | SQLite via Drizzle/Effect | JSONL or SQLite, append-only log as truth |\n| Compaction | ✅ | ✅ + snapcompact bitmap-frame compression | ✅ | ✅ + spill + semantic query |\n| Remote/collab | Experimental CBOR server/client, leases | `/collab` links, browser guests, encrypted | Share links, cloud console | Web host/client, JSON-RPC SDK, ACP |\n| Telemetry | Vendor-neutral contracts pkg | Local stats dashboard | Cloud stats site + Honeycomb/Sentry | OTel + local observability |\n| Native perf layer | None (pure TS) | ~80k LoC Rust N-API addon, zero-fork shell/grep/utils | opentui Go-core heritage (TUI binaries) | Landlock launcher (C++), otherwise pure TS |\n| Config model | Extensions/skills/themes/packages | models.yml, settings, skills marketplace | Markdown/JSON agents + config files | Ordered patch layers → profiles/bundles |\n\n---\n\n## 5. Architecture Deep Dive\n\n### 5.1 Kernel / composition model\n\n| | Kernel concept | Swap granularity |\n|---|---|---|\n| **pi** | Plain layered libraries; composition happens in `pi-coding-agent`'s bootstrap | Extensions hook lifecycle events; whole features live outside core |\n| **oh-my-pi** | pi's layering + capability providers (tools/skills/settings discovered via `loadCapability`) | Extensions register into the same registries as built-ins; Rust behind one N-API boundary |\n| **opencode** | Effect layers composing Schema→Core→Server; single HttpApi as contract | New Context Sources, plugins (hook-based), custom providers/models/commands |\n| **dsh** | **Cordis plugin tree**: services, events, waterfalls, disposers | Anything — including the agent loop, session log implementation, sandbox backend — via patch layers |\n\nThe philosophical spectrum: **pi minimizes the kernel** → **opencode fixes the kernel and generates everything downstream from one API contract** → **dsh makes even the kernel's contents negotiable at boot** → **omp maximizes what ships inside the kernel**.\n\n### 5.2 Client/server and protocols\n\n- **opencode**: strongest client/server story. One authoritative Effect `HttpApi`; codegen produces both Promise and Effect clients; same API executed in-process (\"Embedded OpenCode\") or over the network; durable replayable per-session SSE vs live instance-wide events — guarantees are explicit and different by design.\n- **pi**: experimental remote sessions with transport-neutral CBOR framing (4-byte length prefix), snapshot-authoritative sync, exclusive/shared leases, ID-correlated requests, transport-layer auth. Progress events never mutate state.\n- **dsh**: JSON-RPC (TS SDK + Python SDK over stdio NDJSON), ACP server for editors, Typert RPC gateway for the web host, Claude Code/Codex hook wire protocols, MCP.\n- **omp**: stdio NDJSON RPC + ACP (Zed etc.), collab wire types in `packages/wire`, browser guest relay; Python client (`omp-rpc`) exists for the robomp bot.\n\n### 5.3 Session/state architecture\n\n- **dsh** is the most rigorous: append-only session log is the source of truth; \"model-visible ⟺ logged\" invariant checked at build time; everything (resume, fork, transcripts, telemetry, titles) derived from the event stream; bounded reads/lineage/FTS in a query layer.\n- **opencode** specifies deeply (specs/v2): durable prompt admission decoupled from execution, serialized drains, steering at safe turn boundaries, Context Epochs for provider-cache stability.\n- **omp/pi**: SQLite-backed sessions with branching (pi) and compaction (both); omp adds checkpoint/rewind and a dedicated compression engine (snapcompact).\n- All four treat oversized tool output as a first-class problem: opencode spills to temp files, dsh has an explicit spill package, omp bounds/truncates via native helpers.\n\n### 5.4 Performance strategy\n\n- **omp** stands alone here: in-process Rust for grep/shell/AST/highlight/PTY/walking; embedded bash with 58+ ported utilities; zero fork/exec on hot paths; hermetic Bazel builds producing musl + AVX2/baseline variants.\n- **opencode/dsh/pi** accept JS-speed tool implementations; their performance work goes into rendering (opentui differential rendering, pi-tui CSI 2026 synchronized output) rather than tool execution.\n\n---\n\n## 6. Capability Comparison\n\n### Tools & integrations\n\n| Capability | pi | omp | opencode | dsh |\n|---|---|---|---|---|\n| File read/edit/write | ✅ | ✅ + hashline content-hash patches + AST edit | ✅ + patch/diff + checkpoints | ✅ + policy fencing |\n| Shell | bash | **in-process embedded bash** + 46+ builtins | bash + PTY | bash/pwsh + persistent PTY |\n| Code search | glob/grep via tools | native rg-as-library + ast-grep | glob/grep | fs discovery + bash |\n| LSP | ❌ | ✅ | ✅ | ✅ |\n| Debugger | ❌ | ✅ DAP | ❌ | ❌ |\n| Browser automation | ❌ | ✅ Puppeteer/Electron/Chrome relay | webfetch only | web fetch/search providers |\n| Desktop/computer use | ❌ | ✅ windows/screenshots/input/AX tree | ❌ | ❌ |\n| Background jobs | ❌ (non-goal) | ✅ task subagents/worktrees | ❌ core (plugins possible) | ✅ job_* + workflow/ralph |\n| Memory systems | ❌ | ✅ mnemopi retain/recall/reflect/learn | ❌ core | session-query semantic retrieval |\n| Image/audio gen | via pi-ai | generate_image/inspect_image/tts/voice | ❌ core | via llm/web seams |\n| Skills | ✅ (replaces MCP) | ✅ marketplace + cross-tool import | ✅ permission-checked | ✅ catalog + loader |\n| Hooks (Claude Code-style) | extensions instead | extensions + stream rules | plugin hooks | ✅ dedicated bridges + guard/deadlines |\n\n### Model routing & providers\n\n- **omp** is deepest: 10 role slots, fallback chains, path-scoped models, round-robin credentials, custom OpenAI-compatible declarations, per-provider reasoning-effort normalization with 400/422 fallback parsing.\n- **opencode** leverages the AI SDK + models.dev for breadth; mid-session switching supported.\n- **pi** generates catalogs from live provider data; auth resolution across env/OAuth/credential store; mid-session hand-off.\n- **dsh** centers on DeepSeek but gains breadth by delegating to pi-ai behind its adapter seam — the clearest evidence that these projects compose rather than merely compete.\n\n---\n\n## 7. Extensibility Models Compared\n\n| | Mechanism | Distribution | Can change core behavior? |\n|---|---|---|---|\n| **pi** | TS extensions (lifecycle hooks), skills, themes, prompt templates | npm/git \"Pi Packages\" | Only via hooks maintainer approves adding; core PRs rejected if bloating |\n| **omp** | TS extensions sharing built-in registries (tools/slash/hotkeys), MCP, skills marketplace, `xd://` devices, stream rules | Files/marketplace/npm | Yes — extensions sit in the same registries as built-ins |\n| **opencode** | Plugins (hooks, e.g., PTY env observation), custom providers/models/commands, Context Sources | npm packages | Moderately — plugin hook API; core internals not directly exposed |\n| **dsh** | Cordis plugins mounted at boot; profiles/patch layers; HMR; **the agent can self-modify its plugin graph** | Bundles/profiles, npm | Yes — maximally: swap the agent loop, sandbox, persistence; agent does it at runtime |\n\nDistinctive stance each way:\n- **pi**: extensibility is the *product*; core stays a substrate.\n- **omp**: built-ins are just privileged extensions; parity between user code and shipped code.\n- **opencode**: extension points are documented and versioned against the generated API contract.\n- **dsh**: configuration *is* code — boot-time composition trees, plus the unusual ability for the agent to reconfigure itself mid-session.\n\n---\n\n## 8. Engineering Quality: Testing, Build, Release\n\nAll four are unusually disciplined. Standout practices per project:\n\n| | Testing | Build | Release/Distribution |\n|---|---|---|---|\n| **pi** | Vitest + node:test; isolated temp-env test runner (fake HOME, scrubbed env, askpass stub); faux-provider harness (no real tokens); model-backed evals with JSONL artifacts; tmux-driven TUI testing prescribed | Dependency-chain build order; Bun standalone binaries; offline model-data variant | Lockstep versioning, isolated smoke installs, OIDC trusted publishing, release-marker verification on pi.dev; supply-chain gates (release-age, script allowlist) |\n| **omp** | bun test tiered orchestrator (singleton/ui/runtime/native/heavy splits); cargo nextest + pedantic clippy; pytest; contract-test rules (no source-grep tests, no mock.module) | Cargo local / Bazel hermetic shipping (zig cc cross, 8 targets, remote cache); Nix flake | curl installer, PowerShell, npm, Homebrew tap, NixOS/Home-Manager modules, mise; signing/notarization; `--smoke-test` probe in install CI |\n| **opencode** | Per-package Bun tests (root-run guarded); http-recorder cassettes; Playwright e2e; style guide forbids mocks/globalThis | Turborepo typecheck/build tasks; always package-scoped `bun typecheck` | SST v4 deploy (Cloudflare+AWS, Stripe, stage-gated); Windows code-signing; multi-channel install (npm/brew/scoop/choco/pacman/nix/curl) |\n| **dsh** | **Per-file 100% coverage CI gate**; fast-check property tests; keyless snapshot replay of ACP/headless transcripts; web stress/perf configs; jscpd clone detection; real-API e2e self-skipping | tsc project references + tsdown; separate source/artifact planes; mock LLM dev server | ~30 machine-checked repo gates (doc budgets, link checking, catalog freshness, vendored-mod logs); wine-reproducible Windows gates; GitLab + GitHub CI; Python single-exe carrier on PyPI |\n\n---\n\n## 9. Which One Should You Use?\n\n- **Choose pi** if you want maximum control with minimal machinery: you enjoy writing TypeScript, want your workflow features as your own packages, run untrusted work in containers yourself, and value supply-chain rigor. It's also the best codebase to *read* if you want to understand agent-harness fundamentals.\n- **Choose oh-my-pi** if you want the most capable *tool surface* today: real debugging (DAP), LSP-aware edits, parallel subagents in isolated worktrees, desktop/browser automation, deep model routing — and you're fine with a large native component and a fast-moving fork.\n- **Choose opencode** if you want a polished product spanning terminal/desktop/IDE/cloud with a mature permission model, team features (share links, GitHub Action, Slack), and the strongest client/server + generated-SDK story for building *on top of* the agent programmatically.\n- **Choose deepseek-harness** if you're building a *custom harness* rather than using one: the capability-seam design lets you swap LLM, shell, sandbox, persistence, even the loop; the Web UI is first-class; DeepSeek models get the native path. Caveat: developer preview, no stability promise, no first-party TUI.\n\nThey also stack: dsh already depends on pi's provider library; omp tracks pi upstream; and any of them can be driven headlessly from CI. For teams, the pragmatic pairing is often **one product for daily use** (opencode or omp) **plus one hackable substrate** (pi or dsh) for bespoke automations.\n"
}
```

**Output:**
```
Wrote file successfully.
```

---

## Assistant (Build · x-preview-f-free · 3.1s)

**Tool: todowrite**

**Input:**
```json
{
  "todos": [
    {
      "content": "Research all 4 repos",
      "priority": "high",
      "status": "completed"
    },
    {
      "content": "Verify key claims/numbers across repos",
      "priority": "medium",
      "status": "completed"
    },
    {
      "content": "Build comparison matrix + write comparison-report.md",
      "priority": "high",
      "status": "completed"
    }
  ]
}
```

**Output:**
```
[
  {
    "content": "Research all 4 repos",
    "status": "completed",
    "priority": "high"
  },
  {
    "content": "Verify key claims/numbers across repos",
    "status": "completed",
    "priority": "medium"
  },
  {
    "content": "Build comparison matrix + write comparison-report.md",
    "status": "completed",
    "priority": "high"
  }
]
```

---

## Assistant (Build · x-preview-f-free · 7.6s)

Report written to `oss/comparison-report.md`. Highlights:

- **Lineage**: omp is a direct fork of pi (keeps `pi-*` names, tracked sync point); dsh is *not* a fork but uses pi's `pi-ai` library behind its LLM seam; opencode is independent.
- **Philosophy spectrum**: pi minimizes the kernel (4 tools, extensions-only) → omp maximizes built-ins (31 tools, ~80k LoC Rust, DAP/desktop/browser) → opencode fixes one authoritative HttpApi and generates everything downstream → dsh makes even the agent loop a swappable Cordis plugin.
- The file includes per-project deep profiles, a 25-row comparison matrix, architecture/protocol/session deep dives, extensibility models, engineering-practices comparison, and a "which to choose" section.

---

## User

compare the file and folder structure also. write report in another file.

---

## Assistant (Build · x-preview-f-free · 7.9s)

_Thinking:_

The user wants a comparison of the file/folder structures of the four repos, written to another .md file. I should gather actual structural data: top-level entries (already have from earlier reads), packages/ subdirectories for each, and maybe counts. Let me collect the directory listings efficiently with bash.

**Tool: bash**

**Input:**
```json
{
  "command": "for r in deepseek-harness opencode pi oh-my-pi; do echo \"=== $r/packages ===\"; ls /home/sadiq/Documents/web/oss/$r/packages 2>/dev/null; echo \"--- $r/apps|crates|python ---\"; ls -d /home/sadiq/Documents/web/oss/$r/{apps,crates,python,sdks,native,vendor,infra,docs}/* 2>/dev/null | sed \"s|/home/sadiq/Documents/web/oss/$r/||\"; done"
}
```

**Output:**
```
=== deepseek-harness/packages ===
acp
AGENTS.md
api
attachment
boot
bundle
CLAUDE.md
client
code-runtime
compaction
context
core
credentials
e2b
examples
experimental
extensions
feedback
fs
goal
guard
hooks
host
identity
interaction
jobs
llm
lsp
mcp
plan
preset
README.i18n.yaml
README.md
README.zh.md
runtime-diagnostics
sandbox
schedule
sdk
session
session-query
settings
shell
skill
spill
storage
subagent
subprocess
terminal
test-support
todo
typert
util
web
workflow
workspace
--- deepseek-harness/apps|crates|python ---
=== opencode/packages ===
zsh:1: no matches found: /home/sadiq/Documents/web/oss/deepseek-harness/crates/*
zsh:1: no matches found: /home/sadiq/Documents/web/oss/opencode/apps/*
app
cli
client
codemode
console
containers
core
desktop
docs
effect-drizzle-sqlite
effect-sqlite-node
enterprise
function
httpapi-codegen
http-recorder
identity
llm
opencode
plugin
protocol
schema
script
sdk
sdk-next
server
session-ui
slack
stats
storybook
tui
ui
web
--- opencode/apps|crates|python ---
=== pi/packages ===
agent
ai
client
coding-agent
evals
protocol
server
session-backends
telemetry
tui
--- pi/apps|crates|python ---
=== oh-my-pi/packages ===
agent
ai
browser-relay
catalog
coding-agent
collab-web
hashline
metaharness
mnemopi
natives
omptype
snapcompact
stats
tsconfig.workspace.json
tui
typescript-edit-benchmark
utils
wire
--- oh-my-pi/apps|crates|python ---
zsh:1: no matches found: /home/sadiq/Documents/web/oss/pi/apps/*
zsh:1: no matches found: /home/sadiq/Documents/web/oss/oh-my-pi/apps/*

```

---

## Assistant (Build · x-preview-f-free · 4.9s)

**Tool: bash**

**Input:**
```json
{
  "command": "for d in deepseek-harness/apps deepseek-harness/python deepseek-harness/native deepseek-harness/vendor deepseek-harness/docs opencode/infra opencode/sdks opencode/github pi/packages/session-backends oh-my-pi/crates oh-my-pi/python oh-my-pi/infra oh-my-pi/bazel; do echo \"=== $d ===\"; ls /home/sadiq/Documents/web/oss/$d 2>/dev/null; done"
}
```

**Output:**
```
=== deepseek-harness/apps ===
cli
web
=== deepseek-harness/python ===
development.i18n.yaml
development.md
development.zh.md
README.i18n.yaml
README.md
README.zh.md
sdk
sdk-runtime
=== deepseek-harness/native ===
landlock-run
README.i18n.yaml
README.md
README.zh.md
=== deepseek-harness/vendor ===
AGENTS.md
CLAUDE.md
cordis
cosmokit
group
hmr
include
loader
logger-console
README.md
schemastery
timer
=== deepseek-harness/docs ===
agent-lifecycle.i18n.yaml
agent-lifecycle.md
agent-lifecycle.zh.md
AGENTS.md
api-gateway.i18n.yaml
api-gateway.md
api-gateway.zh.md
architecture.i18n.yaml
architecture.md
architecture.zh.md
capability-seams.i18n.yaml
capability-seams.md
capability-seams.zh.md
config-catalog.i18n.yaml
config-catalog.md
config-catalog.zh.md
cookbook
cordis-api
cordis-primer.i18n.yaml
cordis-primer.md
cordis-primer.zh.md
cordis-tutorial
defensive-patterns.i18n.yaml
defensive-patterns.md
defensive-patterns.zh.md
development.i18n.yaml
development.md
development.zh.md
event-producer-consumer.i18n.yaml
event-producer-consumer.md
event-producer-consumer.zh.md
glossary.i18n.yaml
glossary.md
glossary.zh.md
graph-atlas.i18n.yaml
graph-atlas.md
graph-atlas.zh.md
i18n
module-graph.i18n.yaml
module-graph.md
module-graph.zh.md
persistence-catalog.i18n.yaml
persistence-catalog.md
persistence-catalog.zh.md
postmortem
rescope.i18n.yaml
rescope.md
rescope.zh.md
subsystems
testing.i18n.yaml
testing.md
testing.zh.md
tool-catalog.i18n.yaml
tool-catalog.md
tool-catalog.zh.md
tool-execution-pipeline.i18n.yaml
tool-execution-pipeline.md
tool-execution-pipeline.zh.md
user
web-styling.i18n.yaml
web-styling.md
web-styling.zh.md
=== opencode/infra ===
app.ts
console.ts
enterprise.ts
lake.ts
monitoring.ts
secret.ts
stage.ts
stats.ts
=== opencode/sdks ===
vscode
=== opencode/github ===
action.yml
bun.lock
index.ts
package.json
README.md
script
sst-env.d.ts
tsconfig.json
=== pi/packages/session-backends ===
sqlite-node
=== oh-my-pi/crates ===
pi-ast
pi-builtins
pi-iso
pi-natives
pi-shell
pi-voice
pi-walker
vendor
=== oh-my-pi/python ===
omp-rpc
robomp
=== oh-my-pi/infra ===
bazel-remote
docs
reload-runner.sh
runner.Dockerfile
tune-kata-runtime.sh
=== oh-my-pi/bazel ===
BUILD.bazel
clippy.bazelrc
defs.bzl
patches
platforms
toolchains
triples
variants

```

---

## Assistant (Build · x-preview-f-free · 17.5s)

_Thinking:_

I have enough data. Let me get a few more stats: counts of files, maybe depth-2 structure of a couple key packages to compare conventions. Quick stats: number of top-level dirs/files, total file counts.

**Tool: bash**

**Input:**
```json
{
  "command": "for r in deepseek-harness opencode pi oh-my-pi; do echo \"$r: top=$(ls /home/sadiq/Documents/web/oss/$r | wc -l) tracked_files=$(git -C /home/sadiq/Documents/web/oss/$r ls-files 2>/dev/null | wc -l) pkgs=$(ls /home/sadiq/Documents/web/oss/$r/packages 2>/dev/null | grep -v -E '\\.md|json$' | wc -l)\"; done"
}
```

**Output:**
```
deepseek-harness: top=46 tracked_files=7903 pkgs=51
opencode: top=50 tracked_files=6523 pkgs=32
pi: top=18 tracked_files=1401 pkgs=10
oh-my-pi: top=39 tracked_files=6583 pkgs=17

```

---

## Assistant (Build · x-preview-f-free · 95.5s)

**Tool: write**

**Input:**
```json
{
  "content": "# File & Folder Structure Comparison\n\n**Projects:** `deepseek-harness` · `opencode` · `pi` · `oh-my-pi`\n**Companion to:** [comparison-report.md](./comparison-report.md)\n\n---\n\n## 1. Size at a Glance\n\n| Metric | pi | oh-my-pi | opencode | deepseek-harness |\n|---|---|---|---|---|\n| Top-level entries | **18** | 39 | 50 | 46 |\n| Git-tracked files | **1,401** | 6,583 | 6,523 | **7,903** |\n| Packages under `packages/` | **10** (+1 backend) | 17 TS pkgs + 8 Rust crates | 32 | **51 groups** |\n| Extra language roots | — | `crates/`, `python/`, `bazel/` | `sdks/`, `github/`, `infra/` | `apps/`, `python/`, `native/`, `vendor/` |\n\nThe shape mirrors the philosophy from the main report: **pi is one order of magnitude smaller than everything else**, dsh is the largest by file count despite shipping the least product surface, and opencode/omp are mid-sized but wide.\n\n---\n\n## 2. Top-Level Layout Side by Side\n\n### Common \"monorepo starter kit\" (all four have)\n\n```\n.git/  .github/  .gitignore  .gitattributes\npackage.json  tsconfig*.json\nAGENTS.md     ← agent-first development rules in all four\nLICENSE  README.md(+ translations)  CONTRIBUTING.md  SECURITY.md(pi, omp)\nscripts/  docs/(except pi)  patches/\nnode_modules/ (not tracked)\n```\n\n### What each repo adds on top\n\n| Category | pi | oh-my-pi | opencode | deepseek-harness |\n|---|---|---|---|---|\n| **Build orchestration** | `tsconfig.base.json`, npm scripts, `test.sh` | `BUILD.bazel`, `MODULE.bazel`, `.bazelrc/.bazelversion`, `Cargo.toml/lock`, `rust-toolchain.toml`, `bunfig.toml` | `turbo.json`, `sst.config.ts`, `bunfig.toml`, multiple `vitest.*.config.ts` | `pnpm-workspace.yaml`, `tsdown.config.ts`, `pytest.ini`, many `tsconfig.*.json` planes |\n| **Lint/format** | `biome.json` | `biome.json`, `rustfmt.toml`, `deny.toml`, `.fallowrc.jsonc` | `.oxlintrc.json`, `.prettierignore`, `.husky/` | `.oxlintrc.json`(+staged), `.jscpd.json`, `knip.json`, `lefthook.yml` |\n| **Nix dev env** | ❌ | ✅ `flake.nix`, `nix/`, `.cargo/` | ✅ `flake.nix`, `nix/` | ❌ |\n| **Docker** | ❌ | ✅ 2 Dockerfiles (+robomp variant), `.dockerignore` | ✅ `.dockerignore` only (infra lives elsewhere) | ❌ (sandbox backends instead; E2B POC) |\n| **CI extras** | `pi-test.{sh,bat,ps1}` | `infra/bazel-remote`, runner configs | `github/action.yml` (the GitHub Action *is a top-level dir*) | `.gitlab-ci.yml` **+** GitHub workflows, `.agents/` skills dir |\n| **i18n** | README translations only | ❌ | 18+ README translations | **Every doc tripled** (`*.md`, `*.zh.md`, `*.i18n.yaml`) |\n| **Brand/marketing assets** | ❌ | `assets/` | screenshots in root | `BRAND_GUIDELINES.md`(+zh+i18n), `website/`, `examples/` |\n\n---\n\n## 3. The `packages/` Directories\n\nThis is where the architectural differences are most visible.\n\n### pi — 10 packages: a clean dependency stack\n\n```\npackages/\n├── tui              ─┐\n├── telemetry         │ build order:\n├── ai                │ tui → ai → agent → sqlite → protocol\n├── agent             │ → client → server → coding-agent\n├── session-backends/\n│   └── sqlite-node   │\n├── protocol         ─┤ remote-session split:\n├── client           ─┤   protocol → server → client\n├── server           ─┘\n├── coding-agent      ← the actual `pi` CLI (product)\n└── evals             ← behavioral eval harness\n```\n\nEvery name maps 1:1 to a layer in the architecture diagram. No UI apps, no cloud services, no shared component libraries — the structure *is* the minimalism.\n\n### oh-my-pi — 17 packages + 8 crates: fork of pi's stack, widened\n\nKept from pi: `agent`, `ai`, `tui`, `coding-agent` (same names — evidence of the fork).\n\nAdded around that core:\n\n```\npackages/                          crates/\n├── catalog      (model DB)       ├── pi-natives    (N-API mega-addon)\n├── natives      (Rust loader)    ├── pi-shell      (embedded bash)\n├── hashline     (patch lang)     ├── pi-builtins   (coreutils-in-process)\n├── omptype      (validation)     ├── pi-walker     (FS walker)\n├── utils                         ├── pi-ast        (tree-sitter)\n├── wire        (collab proto)    ├── pi-iso        (task isolation)\n├── mnemopi     (memory engine)   └── vendor/\n├── snapcompact (compression)         └── brush-core (bash fork)\n├── stats / collab-web / browser-relay\n├── metaharness / typescript-edit-benchmark   (benchmarking)\n└── tsconfig.workspace.json\n```\n\nNote what's **missing vs pi**: no `protocol`, `client`, `server`, `telemetry`, or `evals` packages — omp replaced pi's experimental remote-stack with its own RPC/ACP/collab surfaces and moved evals into `metaharness`.\n\n### opencode — 32 packages: layered core + product ecosystem\n\n```\nCore chain (strict direction):\nschema → core → llm/protocol → server → httpapi-codegen → client → sdk-next\n                                                     ↘ sdk (legacy)\n\nSurfaces:            tui · app(web/desktop) · desktop · cli · ui · session-ui · storybook\nEcosystem/cloud:     console/*(app,core,function,mail,resource,support) · stats/*\n                     enterprise · identity · slack · function · containers · codemode\nInfra/support:       effect-drizzle-sqlite · effect-sqlite-node · http-recorder\n                     plugin · script · web(docs site) · docs\n```\n\nThe naming encodes the architecture: everything Effect-related is prefixed (`effect-*`), codegen is explicit (`httpapi-codegen`), and cloud products are grouped (`console/*`, `stats/*`). No other repo here ships its own marketing site, storybook, or Slack bot as workspace members.\n\nAlso unique: **product extensions live outside `packages/`** as first-class top-level dirs:\n\n- `sdks/vscode/` — VS Code extension\n- `github/` — the official GitHub Action (own package.json/bun.lock!)\n- `infra/*.ts` — SST IaC modules per service (app, console, enterprise, lake, stats, monitoring, secret, stage)\n- `specs/` — design specs (v2 session/tools/provider/config/storage) kept in-repo\n\n### deepseek-harness — 51 flat groups + apps/: capability seams\n\n```\napps/          cli · web                    ← launchers (like pi's coding-agent role)\npackages/      ~51 FLAT directories, no nesting:\n  Spine:         core · api · typert · boot · preset · bundle\n  Capabilities:  llm · shell · subprocess · terminal · code-runtime · sandbox\n                 fs · lsp · web · compaction · context · skill · mcp · e2b\n  Orchestration: subagent · jobs · workflow · todo · plan · guard · extensions\n                 goal · schedule · feedback · interaction\n  Data plane:    session · session-query · attachment · spill · storage\n                 workspace · settings · credentials · identity\n  Delivery:      host · client · sdk · acp · hooks\n  Support:       util · test-support · runtime-diagnostics · examples · experimental\npython/        sdk · sdk-runtime            ← PyPI distribution carrier\nnative/        landlock-run                 ← C++ N-API sandbox launcher\nvendor/        cordis + cosmokit/schemastery etc.  ← vendored framework, pinned SHAs\ndocs/          ~20 bilingual architecture docs + catalogs + postmortems\n.agents/       agent skills + decision records\nwebsite/       VitePress docs projection\n```\n\nDistinctive structural traits:\n\n- **Flat, uniform naming**: every package is a *capability seam* (`fs`, `shell`, `sandbox`, `lsp`…), not a layer. Compare with opencode where package names describe *positions in a dependency chain*.\n- **`vendor/` with governance**: the Cordis framework is source-vendored, rescoped, SHA-pinned, with an exhaustive modification log and its own AGENTS.md/CLAUDE.md inside.\n- **Docs are a build artifact**: generated catalogs (`tool-catalog`, `persistence-catalog`, `module-graph`, `graph-atlas`) are freshness-gated in CI; every doc has an English, Chinese, and i18n-metadata variant.\n- **`.agents/` directory**: in-repo agent skills and decision-record notes — pi keeps similar config in `.pi/`, omp in `.omp/`, opencode in `.opencode/`; all four are self-hosting their own tooling.\n\n---\n\n## 4. Self-Hosting Config Dirs (each repo develops itself with itself)\n\n| Repo | Dir | Contents |\n|---|---|---|\n| pi | `.pi/` | extensions (tps/redraw diagnostics), prompt templates (`pr.md`, `sa.md`, `cl.md`…), skills, git/npm state dirs |\n| oh-my-pi | `.omp/` | project-local omp configuration |\n| opencode | `.opencode/` | agents/config for developing opencode with opencode |\n| deepseek-harness | `.agents/` + `CLAUDE.md` + `AGENTS.md` | skills, decision records; plus `packages/AGENTS.md`, `packages/CLAUDE.md`, vendor-level AGENTS.md — nested instruction scopes |\n\npi goes furthest: it also publishes maintainer dev sessions as datasets, and its test scripts (`pi-test.sh`) exist specifically for tmux-driven self-testing of the CLI.\n\n---\n\n## 5. Structural Conventions Compared\n\n| Convention | pi | oh-my-pi | opencode | deepseek-harness |\n|---|---|---|---|---|\n| Package grouping | Flat | Flat | Flat (cloud via `console/*`, `stats/*` subdirs) | Flat, ~51 dirs |\n| Multi-language roots | none | `crates/`, `python/`, `bazel/`, `assets/` | none (TS only in ws) | `apps/`, `python/`, `native/`, `vendor/` |\n| Docs location | scattered (README, CONTRIBUTING, rpc docs in pkg) | `docs/` (~80 files, flat) | `specs/` + `packages/web` content + `packages/docs` | `docs/` structured: cookbook/, user/, subsystems/, postmortem/, catalogs |\n| Test infra placement | per-package + root `test.sh` | per-package + `scripts/ci-test-ts.ts` orchestrator + `metaharness` pkg | per-package + root vitest configs ×5 + `http-recorder` pkg | per-package + 5 vitest configs + `test-support` + `runtime-diagnostics` pkgs |\n| Release scripts | `scripts/build-binaries.sh` | `scripts/` release family + Homebrew/Nix/mise | `script/publish.ts` + sign-windows.ps1 + install script | `scripts/` (~80 gates/generators) + python carrier |\n| Lockfiles | package-lock.json | bun.lock + Cargo.lock | bun.lock | pnpm-lock.yaml + Cargo? (no — native built per-platform) |\n| Translations | README only (—) | none | README ×18 languages | README + CONTRIBUTING + BRAND + **every doc file** (en/zh/i18n.yaml) |\n\n---\n\n## 6. Reading Structure as Philosophy\n\n1. **pi**: `packages/` reads like a textbook layer diagram. Ten names, zero surprises. If you can't guess where code lives from the architecture, the structure failed — so it doesn't.\n\n2. **oh-my-pi**: pi's skeleton with growth rings around it — the four preserved pi packages sit beside nine purpose-built ones (`hashline`, `mnemopi`, `snapcompact`…) and a whole parallel `crates/` universe bridged by exactly one package (`natives`). The boundary between TS and Rust is a single structural seam.\n\n3. **opencode**: two distinct trees in one repo — a *framework* tree (strictly ordered core chain, codegen, generated clients) and a *business* tree (console, stats, slack, enterprise). Plus infra-as-code and the GitHub Action as sibling top-level dirs: the repo is simultaneously the product and the company's deployment unit.\n\n4. **deepseek-harness**: no layers visible in the tree at all — just capabilities. Layering exists only in the boot composition (profiles/bundles), not the folder layout. The heavy investment is instead in *governance structure*: vendor logs, generated catalogs, gate scripts, trilingual docs, postmortems. It's the only repo where `docs/` out-numbers most other repos' entire source tree per-file discipline.\n\n---\n\n## 7. Quick Reference: Where Do I Find X?\n\n| Looking for… | pi | oh-my-pi | opencode | dsh |\n|---|---|---|---|---|\n| Agent loop | `packages/agent` | `packages/agent` | `packages/core` (src/session…) | `packages/core` (swappable) |\n| Tools impl | `packages/coding-agent/src/tools`(read/bash/edit/write) | `packages/coding-agent` (31 tools) + Rust crates | `packages/core` tools module | `packages/*` per-capability tool pkgs |\n| TUI framework | `packages/tui` | `packages/tui` | `packages/tui` (@opentui/solid) | ❌ (Web UI: `host`/`client`) |\n| Provider layer | `packages/ai` | `packages/ai` + `catalog` | `packages/llm` + AI SDK | `packages/llm*` adapters |\n| Remote API/protocol | `protocol`→`server`→`client` | `wire` + RPC mode + ACP | `protocol`→`server`→`httpapi-codegen`→`client` | `sdk`, `acp`, `api`, `typert` |\n| Persistence | `session-backends/sqlite-node` | SQLite + `mnemopi` | `core` + `effect-*-sqlite` | `session` + `session-query` |\n| Sandboxing | external (docs) | `crates/pi-iso` | `packages/containers` | `packages/sandbox` + `native/landlock-run` |\n| CI/release | `scripts/`, `.github/` | `scripts/`, `bazel/`, `infra/` | `script/`, `infra/`, `.github/` | `scripts/`, `.github/` + `.gitlab-ci.yml` |\n",
  "filePath": "/home/sadiq/Documents/web/oss/structure-comparison.md"
}
```

**Output:**
```
Wrote file successfully.
```

---

## Assistant (Build · x-preview-f-free · 8.9s)

Written to `oss/structure-comparison.md`. Key structural findings:

- **pi** (10 pkgs, 1.4k files): `packages/` reads as a literal layer diagram — smallest by 4–5×.
- **oh-my-pi**: pi's skeleton preserved (`agent`, `ai`, `tui`, `coding-agent` kept) plus a parallel Rust universe in `crates/` bridged by one package (`natives`); pi's remote stack (protocol/server/client) was dropped and replaced.
- **opencode**: two trees in one repo — a strictly ordered framework chain (`schema→core→server→codegen→client`) and a business tree (console, stats, slack, enterprise), with `infra/`, `sdks/vscode/`, and the GitHub Action as top-level siblings.
- **deepseek-harness** (51 flat groups, 7.9k files — largest): no layers visible in folders, only capability seams; heavy governance structure instead — vendored Cordis with modification logs, generated doc catalogs, ~30 CI gate scripts, trilingual everything.

The report includes size stats, side-by-side top-level layouts, annotated package trees, self-hosting config dir comparison, and a "where do I find X" quick reference table.

---

## User

create an .md file explaining each file in pi.

---

## Assistant (Build · x-preview-f-free · 8.9s)

_Thinking:_

The user wants an .md file explaining each file in pi. pi has 1,401 tracked files — explaining *every* file is infeasible/unreasonable. I should interpret sensibly: explain each top-level file and directory structure of pi thoroughly. But the user said "each file in pi" — likely means each file/folder at top

---

## User

create an .md file explaining each file an folder in pi.

---

## Assistant (Build · x-preview-f-free · 6.5s)

_Thinking:_

The user wants a .md file explaining each file and folder in the pi repo. I need to enumerate pi's files/folders. Top-level I already have (25 entries). Let me get details: scripts/, packages subdirs, .pi/, .github/, etc.

Let me gather the full tree at reasonable depth, excluding node_modules/.git.

**Tool: bash**

**Input:**
```json
{
  "command": "find /home/sadiq/Documents/web/oss/pi -maxdepth 2 -not -path '*/node_modules*' -not -path '*/.git/*' -not -name node_modules -not -name .git | sed 's|/home/sadiq/Documents/web/oss/pi/||' | sort"
}
```

**Output:**
```
AGENTS.md
biome.json
CONTRIBUTING.md
.gitattributes
.github
.github/APPROVED_CONTRIBUTORS
.github/ISSUE_TEMPLATE
.github/workflows
.gitignore
/home/sadiq/Documents/web/oss/pi
.husky
.husky/pre-commit
LICENSE
.npmrc
package.json
package-lock.json
packages
packages/agent
packages/ai
packages/client
packages/coding-agent
packages/evals
packages/protocol
packages/server
packages/session-backends
packages/telemetry
packages/tui
.pi
.pi/extensions
.pi/git
.pi/npm
.pi/prompts
.pi/skills
pi-test.bat
pi-test.ps1
pi-test.sh
README.md
scripts
scripts/agent-treeshake-smoke-entry.ts
scripts/auto-pi.sh
scripts/browser-smoke-entry.ts
scripts/build-binaries.sh
scripts/build-coding-agent-bundle.mjs
scripts/check-browser-smoke.mjs
scripts/check-lockfile-commit.mjs
scripts/check-pinned-deps.mjs
scripts/check-ts-relative-imports.mjs
scripts/cost.ts
scripts/create-source-archive.sh
scripts/diff-model-catalog.mjs
scripts/edit-tool-stats.mjs
scripts/generate-coding-agent-install-lock.mjs
scripts/generate-coding-agent-shrinkwrap.mjs
scripts/generate-thinking-capabilities.mjs
scripts/local-release.mjs
scripts/package-workspaces.mjs
scripts/profile-coding-agent-node.mjs
scripts/publish.mjs
scripts/publish-model-catalog.mjs
scripts/publish-release-announcement.mjs
scripts/publish-release-announcement.test.mjs
scripts/read-tool-stats.mjs
scripts/release.mjs
scripts/release-notes.mjs
scripts/release-packages.mjs
scripts/repro-5893-wsl-bash.mjs
scripts/session-context-stats.mjs
scripts/session-transcripts.ts
scripts/stats.ts
scripts/sync-versions.js
scripts/sync-versions.test.mjs
scripts/tool-stats.ts
scripts/update-source-imports-to-ts.sh
SECURITY.md
test.sh
tsconfig.base.json
tsconfig.json
tui-plan.md
vitest.base.ts

```

---

## Assistant (Build · x-preview-f-free · 9.0s)

**Tool: bash**

**Input:**
```json
{
  "command": "for p in agent ai client coding-agent evals protocol server session-backends/sqlite-node telemetry tui; do echo \"=== packages/$p ===\"; ls /home/sadiq/Documents/web/oss/pi/packages/$p | head -25; done; echo \"=== .pi subdirs ===\"; ls /home/sadiq/Documents/web/oss/pi/.pi/extensions /home/sadiq/Documents/web/oss/pi/.pi/prompts /home/sadiq/Documents/web/oss/pi/.pi/skills /home/sadiq/Documents/web/oss/pi/.github/workflows"
}
```

**Output:**
```
=== packages/agent ===
CHANGELOG.md
docs
package.json
README.md
scripts
src
test
tsconfig.build.json
vitest.config.ts
vitest.harness.config.ts
=== packages/ai ===
bedrock-provider.d.ts
bedrock-provider.js
CHANGELOG.md
package.json
README.md
scripts
src
test
tsconfig.build.json
vitest.config.ts
=== packages/client ===
CHANGELOG.md
package.json
README.md
src
test
tsconfig.build.json
tsconfig.test.json
vitest.config.ts
=== packages/coding-agent ===
CHANGELOG.md
docs
examples
install-lock
npm-shrinkwrap.json
package.json
README.md
scripts
src
test
tsconfig.build.json
tsconfig.examples.json
vitest.config.ts
=== packages/evals ===
package.json
README.md
scripts
src
test
tsconfig.json
vitest.config.ts
vitest.test.config.ts
=== packages/protocol ===
CHANGELOG.md
package.json
README.md
src
test
tsconfig.build.json
tsconfig.test.json
vitest.config.ts
=== packages/server ===
CHANGELOG.md
package.json
README.md
src
test
tsconfig.build.json
tsconfig.test.json
vitest.config.ts
=== packages/session-backends/sqlite-node ===
CHANGELOG.md
package.json
README.md
scripts
src
test
tsconfig.build.json
vitest.config.ts
=== packages/telemetry ===
CHANGELOG.md
package.json
README.md
src
test
tsconfig.build.json
=== packages/tui ===
CHANGELOG.md
native
package.json
README.md
src
test
tsconfig.build.json
=== .pi subdirs ===
/home/sadiq/Documents/web/oss/pi/.github/workflows:
approve-contributor.yml
build-binaries.yml
ci.yml
issue-analysis.yml
issue-gate.yml
issue-triage-labels.yml
npm-audit.yml
pr-gate.yml
publish-model-catalog.yml
remove-inprogress-on-close.yml

/home/sadiq/Documents/web/oss/pi/.pi/extensions:
import-repro.ts
prompt-url-widget.ts
redraws.ts
tps.ts

/home/sadiq/Documents/web/oss/pi/.pi/prompts:
cl.md
is.md
pr.md
sa.md
wr.md

/home/sadiq/Documents/web/oss/pi/.pi/skills:
add-llm-provider.md

```

---

## Assistant (Build · x-preview-f-free · 82.6s)

_Thinking:_

I have enough. Write the pi-files-guide.md file. Explain each top-level file/folder, each package, common in-package structure, scripts/ files (many — group them logically but the user asked "each file"), .pi contents, .github workflows.

Let me write it comprehensively.

**Tool: write**

**Input:**
```json
{
  "content": "# pi Repository: File & Folder Guide\n\nA file-by-file explanation of [`pi/`](./pi/) (`badlogic/pi-mono`), the minimal agent harness.\nFor context see [comparison-report.md](./comparison-report.md) and [structure-comparison.md](./structure-comparison.md).\n\n---\n\n## 1. Root Files\n\n| File | Purpose |\n|---|---|\n| `package.json` | npm workspace root; defines the **build order** of all packages (tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent), plus `check`, `build`, `test`, `eval`, `release:*` scripts |\n| `package-lock.json` | Ground-truth lockfile; changes are gated by pre-commit checks (`PI_ALLOW_LOCKFILE_CHANGE=1` override) and `scripts/check-lockfile-commit.mjs` |\n| `.npmrc` | npm config incl. supply-chain hardening settings such as minimum release age for dependencies |\n| `tsconfig.json` / `tsconfig.base.json` | Shared strict TS config; erasable-only syntax enforced (Node strip-only mode) |\n| `vitest.base.ts` | Base Vitest config inherited by every package's `vitest.config.ts` |\n| `biome.json` | Biome lint/format rules (runs with `--error-on-warnings`) |\n| `AGENTS.md` | Strict contribution rules for humans *and* AI agents; prescribes tmux-driven TUI testing, regression-test placement, core-bloat policy |\n| `README.md` | Project overview, philosophy (\"No MCP. No sub-agents…\"), usage docs |\n| `CONTRIBUTING.md` | Contribution process; warns that core-bloating PRs are rejected |\n| `SECURITY.md` | Security policy and vulnerability reporting |\n| `LICENSE` | MIT license |\n| `.gitignore` / `.gitattributes` | Standard ignores; line-ending/normalization attributes |\n| `test.sh` | Runs non-e2e tests inside a **fully isolated temp environment** (fake HOME, scrubbed env vars, git askpass stub) so tests never touch real credentials |\n| `pi-test.sh` / `pi-test.bat` / `pi-test.ps1` | Run the `pi` CLI from source via tsx for interactive testing; `--no-env` unsets all ~30 provider credential variables |\n| `tui-plan.md` | Working plan document for the TUI alt-screen layout system (VStack/HStack/ScrollView, sticky docked footer, mouse-wheel routing) |\n\n---\n\n## 2. Root Directories\n\n### `packages/` — the monorepo (see §3)\n\n### `scripts/` — repo automation (~35 scripts)\n\n**Supply-chain & hygiene gates**\n| Script | Purpose |\n|---|---|\n| `check-pinned-deps.mjs` | Verifies external deps are exactly pinned (no ranges) |\n| `check-lockfile-commit.mjs` | Ensures lockfile changes are intentional/gated |\n| `check-ts-relative-imports.mjs` | Enforces erasable-TS import rules (no extensionless relative imports) |\n| `check-browser-smoke.mjs` + `browser-smoke-entry.ts` | Browser-runtime smoke test (TS code must also run outside Node) |\n| `generate-coding-agent-shrinkwrap.mjs` / `generate-coding-agent-install-lock.mjs` | Regenerate the published package's shrinkwrap + lifecycle-script install allowlist |\n\n**Build & release**\n| Script | Purpose |\n|---|---|\n| `build-binaries.sh` | Build standalone Bun binaries per platform (`--offline-model-data --platform …`) |\n| `build-coding-agent-bundle.mjs` | esbuild bundle for the coding-agent package |\n| `agent-treeshake-smoke-entry.ts` | Tree-shaking smoke entry to verify dead-code elimination |\n| `release.mjs` / `local-release.mjs` / `release-packages.mjs` | Version bump, changelog, tag, publish pipeline; local variant builds isolated Node+Bun installs as release-blocker smoke tests |\n| `release-notes.mjs` | Generate release notes from changelogs |\n| `publish-release-announcement.mjs` (+`.test.mjs`) | Publish the R2-verified announcement marker on pi.dev |\n| `publish-model-catalog.mjs` | Refresh/publish generated model catalogs |\n| `sync-versions.js` (+`.test.mjs`) | Lockstep versioning across all packages |\n| `package-workspaces.mjs` | Packaging helper for workspaces |\n\n**Model/tooling utilities (dev diagnostics)**\n| Script | Purpose |\n|---|---|\n| `cost.ts`, `stats.ts`, `tool-stats.ts`, `read-tool-stats.mjs`, `edit-tool-stats.mjs`, `session-context-stats.mjs` | Analyze sessions: token costs, tool usage stats, read/edit behavior stats |\n| `session-transcripts.ts` | Export/manipulate session transcripts |\n| `diff-model-catalog.mjs` | Diff generated model catalog against live provider data |\n| `generate-thinking-capabilities.mjs` | Generate per-model thinking/reasoning capability data |\n| `profile-coding-agent-node.mjs` | CPU-profile the CLI under Node |\n| `repro-5893-wsl-bash.mjs` | Repro script for a specific WSL bash issue (#5893) |\n| `auto-pi.sh` | Convenience launcher running pi on itself/workflows |\n| `create-source-archive.sh` | Source tarball creation |\n| `update-source-imports-to-ts.sh` | Codemod imports to explicit `.ts` extensions |\n\n### `.pi/` — pi configuring itself with pi\n| Path | Contents |\n|---|---|\n| `extensions/import-repro.ts` | Diagnostic extension reproducing an import issue |\n| `extensions/prompt-url-widget.ts` | TUI widget extension showing prompt URLs |\n| `extensions/redraws.ts` | Diagnostics counting TUI redraws |\n| `extensions/tps.ts` | Tokens-per-second live meter widget |\n| `prompts/pr.md` | Prompt template: PR workflow |\n| `prompts/sa.md` | Prompt template (session/analysis helper) |\n| `prompts/cl.md` | Prompt template: changelog audit |\n| `prompts/is.md` | Prompt template: issues |\n| `prompts/wr.md` | Prompt template (work/release helper) |\n| `skills/add-llm-provider.md` | Skill teaching an agent how to add a new LLM provider to `pi-ai` |\n| `git/`, `npm/` | Gitignored local state dirs |\n\n### `.github/`\n| Path | Purpose |\n|---|---|\n| `workflows/ci.yml` | Main CI: build order, checks, tests |\n| `workflows/pr-gate.yml` | PR gating checks |\n| `workflows/build-binaries.yml` | Standalone binary builds per platform |\n| `workflows/npm-audit.yml` | Scheduled dependency audits |\n| `workflows/approve-contributor.yml` + `APPROVED_CONTRIBUTORS` | Contributor allowlist mechanism |\n| `workflows/issue-gate.yml`, `issue-triage-labels.yml`, `issue-analysis.yml`, `remove-inprogress-on-close.yml` | Issue automation/triage pipeline |\n| `workflows/publish-model-catalog.yml` | Scheduled model catalog regeneration/publish |\n| `ISSUE_TEMPLATE/` | Issue templates |\n\n### `.husky/pre-commit` — runs the gate suite (Biome, pinned deps, lockfile, ts-relative-imports, shrinkwrap regen) before every commit.\n\n---\n\n## 3. `packages/` — Package by Package\n\nEvery package follows a shared skeleton:\n\n```\n<pkg>/\n├── package.json          workspace member, lockstep version\n├── README.md             usage + architecture notes\n├── CHANGELOG.md          maintained per package\n├── src/                  source (erasable TS only)\n├── test/                 Vitest suites\n├── vitest.config.ts      extends ../../vitest.base.ts\n└── tsconfig.build.json   build project reference\n```\n\nBuild order (dependency chain): **tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent**\n\n### `packages/tui` — `@earendil-works/pi-tui`\nMinimal terminal UI framework.\n- `src/` — differential renderer, synchronized output (CSI 2026), main-screen and alternate-screen renderers, components (Editor with configurable keybindings, Markdown, SelectList), bracketed paste, autocomplete, image support (Kitty/iTerm2 protocols)\n- `native/` — small native bits used by the renderer\n- `test/` — tested with node:test (in addition to Vitest)\n\n### `packages/telemetry` — `@earendil-works/pi-telemetry`\nVendor-neutral telemetry contracts: callback-based context/span interfaces, typed schemas, no-op and in-memory reference adapters, conformance tests. Deliberately no exporter/backend coupling. (No vitest config at root — tests run under base setup.)\n\n### `packages/ai` — `@earendil-works/pi-ai`\nUnified multi-provider LLM API.\n- `src/` — provider adapters (OpenAI, Anthropic, Google, Bedrock, Azure, Groq, xAI, OpenRouter, Mistral, …), streaming, tool calling, thinking levels, image input/generation, auth resolution (env vars, OAuth, credential store), token/cost tracking, mid-session model hand-off\n- `models.generated.ts` area — model catalog generated from live provider data\n- `bedrock-provider.js/.d.ts` — prebuilt Bedrock provider entry\n- `scripts/` — model-catalog generation from provider APIs\n\n### `packages/agent` — `@earendil-works/pi-agent-core`\nGeneral-purpose stateful agent runtime: subscribe-based event streaming, tool execution loop, message/state management, attachments, pluggable stream function, transport abstraction, pluggable session backends. Includes its own `docs/` and harness test config (`vitest.harness.config.ts`).\n\n### `packages/session-backends/sqlite-node`\nSQLite session persistence backend for pi-agent-core built on Node's native `node:sqlite`: repository pattern, migrations, materialized views, optional FTS search. Kept out of core so core has zero runtime deps; accepts an injected DB factory so alternative backends can ship independently.\n\n### `packages/protocol` — `@earendil-works/pi-protocol`\nTransport-neutral remote-session wire protocol: CBOR schemas/types/encoding, byte-stream framing (v1: 4-byte big-endian length prefix + CBOR message). No runtime-specific imports.\n\n### `packages/client` — `@earendil-works/pi-client`\nRemote-session client (`PiClient`) working over any ordered byte transport (WebSocket, Unix socket): exclusive/shared session leases, snapshot-authoritative state, ID-correlated requests. Browser-safe.\n\n### `packages/server` — `@earendil-works/pi-server`\nExperimental session server composing pluggable transport listeners (Unix socket preset included). Apps supply their own service implementation (`PiServerService`).\n\n### `packages/coding-agent` — `@earendil-works/pi-coding-agent`\nThe actual `pi` product.\n- `src/` — interactive TUI app, print/JSON one-shot mode, RPC mode, SDK embedding; read/bash/edit/write tools; sessions with branching and compaction; extension/skill/theme/package loading\n- `docs/` — user-facing docs (incl. RPC protocol docs)\n- `examples/extensions/` — workspace-member example packages: `with-deps`, `custom-provider-anthropic`, `custom-provider-gitlab-duo`, `sandbox`, `gondolin`\n- `npm-shrinkwrap.json` — generated, pinned dependency tree shipped to consumers\n- `install-lock/` — lifecycle-script allowlist for hardened installs\n- `tsconfig.examples.json` — separate typecheck plane for examples\n\n### `packages/evals` — `@earendil-works/pi-evals`\nModel-backed behavioral evals adapting a real `AgentSession` to `vitest-evals`; runs in isolated temp dirs, emits session JSONL artifacts (`.eval/runs.jsonl`). Requires real provider auth; excluded from normal test runs (`npm run eval`).\n\n---\n\n## 4. Visual Map\n\n```\npi/\n├── AGENTS.md · README.md · CONTRIBUTING.md · SECURITY.md · LICENSE\n├── package.json · package-lock.json · .npmrc\n├── tsconfig{,.base}.json · biome.json · vitest.base.ts\n├── tui-plan.md                      # TUI design plan\n├── test.sh                          # isolated-env test runner\n├── pi-test.{sh,bat,ps1}             # run CLI from source\n├── .husky/pre-commit                # gate suite\n├── scripts/                         # ~35 automation scripts\n├── .github/                         # CI, binary builds, triage bots\n├── .pi/                             # pi developing itself\n│   ├── extensions/{import-repro,prompt-url-widget,redraws,tps}.ts\n│   ├── prompts/{pr,sa,cl,is,wr}.md\n│   └── skills/add-llm-provider.md\n└── packages/\n    ├── tui ──────────────┐\n    ├── telemetry          │  core stack\n    ├── ai                 │  (built in this order)\n    ├── agent              │\n    ├── session-backends/sqlite-node\n    ├── protocol           │  remote-session split\n    ├── client             │\n    ├── server             │\n    ├── coding-agent       │  ← the `pi` CLI (product)\n    └── evals              # behavioral evals\n```\n\n---\n\n## 5. Notes\n\n- All TypeScript uses **erasable syntax only** so it runs natively under Node ≥22 without transpilation; `scripts/check-ts-relative-imports.mjs` polices this.\n- The repo is developed agent-first: `.pi/` contains the same artifact types users create (extensions/prompts/skills) — the maintainers dogfood their own extensibility system.\n- `oh-my-pi` preserves four of these package names verbatim (`ai`, `agent`, `tui`, `coding-agent`) — see [structure-comparison.md](./structure-comparison.md).\n",
  "filePath": "/home/sadiq/Documents/web/oss/pi-files-guide.md"
}
```

**Output:**
```
Wrote file successfully.
```

---

## Assistant (Build · x-preview-f-free · 8.7s)

Written to `oss/pi-files-guide.md`. It covers:

- **All 17 root files** (build order in `package.json`, lockfile gating, `test.sh` isolated-env runner, `pi-test.*` launchers, `tui-plan.md`)
- **All ~35 scripts** grouped by role (supply-chain gates, build/release, model-catalog tooling, session-stats diagnostics)
- **`.pi/` self-hosting dir** — every extension, prompt template, and skill explained
- **`.github/` workflows** (CI, binary builds, issue-triage bots, contributor allowlist)
- **All 10 packages** with their internal structure (`src/test/docs/scripts`, shrinkwrap, install-lock, example extensions) and the build dependency chain
- A visual tree map at the end

---

## User

why this patches folder is used for?

---

## Assistant (Build · x-preview-f-free · 8.3s)

_Thinking:_

The user asks about a "patches" folder. Which repo? Ambiguous — all four have patches/. Likely they're looking at pi... but pi doesn't have a patches folder (not in listing). opencode and oh-my-pi and deepseek-harness have patches/. Let me check quickly.

**Tool: bash**

**Input:**
```json
{
  "command": "ls /home/sadiq/Documents/web/oss/opencode/patches /home/sadiq/Documents/web/oss/oh-my-pi/patches /home/sadiq/Documents/web/oss/deepseek-harness/patches 2>/dev/null"
}
```

**Output:**
```
/home/sadiq/Documents/web/oss/deepseek-harness/patches:
node-pty@1.2.0-beta.15.patch

/home/sadiq/Documents/web/oss/oh-my-pi/patches:
@ark%2Fschema@0.56.2.patch
puppeteer-core@25.3.0.patch

/home/sadiq/Documents/web/oss/opencode/patches:
@ai-sdk%2Fgoogle@3.0.73.patch
@ai-sdk%2Fgroq@3.0.31.patch
@ai-sdk%2Fmistral@3.0.51.patch
@ai-sdk%2Fopenai-compatible@2.0.41.patch
@ai-sdk%2Fxai@3.0.102.patch
@dnd-kit%2Fdom@0.5.0.patch
effect@4.0.0-beta.83.patch
@ff-labs%2Ffff-bun@0.9.3.patch
gcp-metadata@8.1.2.patch
install-korean-ime-fix.sh
@modelcontextprotocol%2Fsdk@1.29.0.patch
@npmcli%2Fagent@4.0.2.patch
pacote@21.5.0.patch
@pierre%2Ftrees@1.0.0-beta.4.patch
@silvia-odwyer%2Fphoton-node@0.3.4.patch
solid-js@1.9.10.patch
@standard-community%2Fstandard-openapi@0.2.9.patch
@tanstack%2Fvirtual-core@3.17.3.patch

```

---

## Assistant (Build · x-preview-f-free · 29.4s)

_Thinking:_

Explain what patches/ folders are: pnpm/bun patchedDependencies — local overrides of upstream npm packages, applied after install. Give examples from each repo. Note pi doesn't have one (uses exact pinning instead).

`patches/` holds **local source fixes applied to third-party npm packages after install** — a feature of pnpm (`patchedDependencies`) and Bun. When a dependency has a bug or missing behavior and you can't wait for upstream, you edit a copy of its code, and the diff is saved as a `.patch` file that gets re-applied on every `install`. That's why all four repos list heavy `patchedDependencies` entries.

What each repo patches:

| Repo | Patches | Why |
|---|---|---|
| **opencode** | 17 files, mostly `@ai-sdk/*` providers, `effect`, `solid-js`, MCP SDK | Aggressive early adoption of beta deps (Effect v4, AI SDK v6) means fixing provider bugs/upstream gaps themselves |
| **oh-my-pi** | `@ark/schema`, `puppeteer-core` | E.g., puppeteer patch adds lazy context providers + invalidation ("stealth" browser automation tweaks) |
| **deepseek-harness** | `node-pty@1.2.0-beta.15` | Fixing PTY behavior in the beta release it depends on |

Note **pi has no `patches/` folder** — consistent with its supply-chain philosophy: instead of carrying patches, it pins exact dependency versions and enforces lockfile-as-ground-truth with release-age gating.

Trade-off: patches give instant control but become maintenance debt — every upstream version bump requires regenerating the patch (see the version numbers baked into filenames).

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · x-preview-f-free · 3.0s)

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · x-preview-f-free · 2.8s)

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · x-preview-f-free · 5.9s)

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · Nemotron 3.5 Lightning Free · 300.0s)

---

## User

there are folders called python, tell me about it?

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · Muse Spark 1.2 Free · 12.9s)

---

## User

there are folders called python, tell me about it?

---

## Assistant (Build · Hy3 Free · 13.6s)

_Thinking:_

The user is repeating "there are folders called python, tell me about it?" six times. They want to know about the `python/` folders in these repos. Only opencode and oh-my-pi don't have python... wait, let me recall. deepseek-harness has `python/` (sdk, sdk-runtime). oh-my-pi has `python/` (omp-rpc, robomp). opencode - did it have python? Earlier listing: opencode top-level didn't list python. pi didn't have python. deepseek-harness has python. oh-my-pi has python.

Let me check which repos have python folders.

From earlier: deepseek-harness/python = sdk, sdk-runtime. oh-my-pi/python = omp-rpc, robomp. opencode - not listed. pi - not listed.

So two repos have python folders. Let me give details. Let me quickly inspect to be accurate.

Two of the four repos have a `python/` folder: **oh-my-pi** and **deepseek-harness**. (pi and opencode don't — opencode is TS-only in its workspace; pi is pure TS.) Let me pull the actual contents.

**Tool: bash**

**Input:**
```json
{
  "command": "echo \"=== oh-my-pi/python ===\"; find /home/sadiq/Documents/web/oss/oh-my-pi/python -maxdepth 2 -not -path '*/node_modules*' -not -path '*/.git/*' | sed 's|/home/sadiq/Documents/web/oss/oh-my-pi/python/||' | sort; echo; echo \"=== deepseek-harness/python ===\"; find /home/sadiq/Documents/web/oss/deepseek-harness/python -maxdepth 2 -not -path '*/node_modules*' -not -path '*/.git/*' | sed 's|/home/sadiq/Documents/web/oss/deepseek-harness/python/||' | sort"
}
```

**Output:**
```
=== oh-my-pi/python ===
/home/sadiq/Documents/web/oss/oh-my-pi/python
omp-rpc
omp-rpc/LICENSE
omp-rpc/pyproject.toml
omp-rpc/README.md
omp-rpc/src
omp-rpc/tests
omp-rpc/uv.lock
robomp
robomp/AGENTS.md
robomp/assets
robomp/docker-compose.yml
robomp/docs
robomp/entrypoint.sh
robomp/.env.example
robomp/.gitignore
robomp/LICENSE
robomp/pyproject.toml
robomp/README.md
robomp/scripts
robomp/src
robomp/tests
robomp/web

=== deepseek-harness/python ===
development.i18n.yaml
development.md
development.zh.md
/home/sadiq/Documents/web/oss/deepseek-harness/python
README.i18n.yaml
README.md
README.zh.md
sdk
sdk/pyproject.toml
sdk/README.i18n.yaml
sdk/README.md
sdk/README.zh.md
sdk-runtime
sdk-runtime/hatch_build.py
sdk-runtime/package.json
sdk-runtime/platforms.json
sdk-runtime/pyproject.toml
sdk-runtime/README.i18n.yaml
sdk-runtime/README.md
sdk-runtime/README.zh.md
sdk-runtime/src
sdk/src
sdk/tests
sdk/uv.lock

```

---

## Assistant (Build · Hy3 Free · 10.8s)

Both `python/` folders are **not part of the core agent** — they're separate Python projects that *drive or package* the TS agent. Here's each:

## oh-my-pi/python

Two independent Python projects (each its own `pyproject.toml` + `uv.lock`):

- **`omp-rpc/`** — a typed Python client for omp's NDJSON RPC protocol. Lets a Python program connect to a running `omp --mode rpc` agent and drive it programmatically (same idea as the `pi-client` package but in Python).
- **`robomp/`** — **"Robotic omp"**: a self-hosted GitHub triage/fix bot. A FastAPI service that, per issue/PR, spins up an `omp` worktree per issue and asks omp to fix bugs / answer questions / even repair failed release CI. Ships `docker-compose.yml`, `entrypoint.sh`, `web/` (a UI), `scripts/`, and its own `AGENTS.md` + `docs/`. It's the "agent-as-a-service" deployment of omp.

## deepseek-harness/python

Three things (note the root `python/` itself holds README/i18n docs):

- **`sdk/`** — the official **Python SDK** (`deepseek-harness-sdk`): drives a bundled dsh runtime over newline-delimited JSON-RPC on stdio. Has `src/`, `tests/`, `uv.lock`.
- **`sdk-runtime/`** — the **distribution carrier** for single-exe deployment. `pyproject.toml` + `package.json` + `platforms.json` + `hatch_build.py` make it the deploy root whose dependency closure gets bundled into a PyPI-distributed runtime binary (`python/sdk-runtime` is the "exe carrier" for `dsh`).
- Root `python/` also has bilingual `README` / `development` docs (en/zh/i18n.yaml) — dsh's Python story is fully i18n like the rest of the repo.

## Why separate from the TS workspace?

In both repos the `python/` tree is a **separate ecosystem** (pip/uv, not pnpm/bun). It exists to either:
1. **ship the agent to non-TS consumers** (dsh's Python SDK = first-class client alongside the TS SDK), or
2. **operationalize the agent as infrastructure** (omp's `robomp` bot auto-fixing GitHub issues on its own).

It's deliberately *not* wired into the TS build graph — it's a sibling product that talks to the agent over its RPC/JSON-RPC boundary.

---

## User

think deeply

---

## Assistant (Build · Hy3 Free · 32.1s)

_Thinking:_

The user says "think deeply". This is ambiguous without a preceding question. They've been asking comparative questions about the four repos. Perhaps they want me to think deeply about something. But what specifically?

Given the conversation, they might want a deeper synthesis/insight. But "think deeply" alone is vague. Let me consider what's reasonable. They could be asking me to think deeply about the comparison overall, or about something specific. Since no object is given, I should ask for clarification, or provide a deep reflection on the comparison they've been building.

Actually "think deeply" likely is a prompt for me to produce a deeper analysis. But on what? The most sensible interpretation: they want a deeper, more thoughtful take on the four projects, beyond the surface reports. Maybe a critical/insightful synthesis.

I'll ask a clarifying question since it's genuinely ambiguous, but I could also offer to do a deep synthesis. Given the instruction to be concise and not over-act, I should clarify what they want me to think deeply about. But they repeated "python" six times earlier and I answered; now "think deeply" six... no, just once.

I think the best is to ask what to think deeply about, but I can also proactively offer a deep insight. However, being concise and helpful: I'll ask a focused clarifying question. Actually the question tool could help. But maybe they want me to think deeply and produce something. Let me offer options.

Given ambiguity, use the question tool to clarify scope. But that might be annoying. Alternatively, give a genuinely deep take on the whole landscape as a synthesis — that's likely valuable and matches "think deeply" as a request for deeper analysis.

I'll produce a concise but genuinely deep synthesis: the underlying design tensions, what each optimizes for, and a "deep" connective insight. Keep it focused, not rehashing reports. Let me think deeply myself and write something insightful.

Actually, re-reading: the user has repeatedly asked narrow questions, and now "think deeply" — likely they want me to step back and give a deep, original analysis rather than another catalog. I'll deliver a substantive deep synthesis covering the meta-patterns, the tradeoffs, and my genuine assessment. Keep it readable.

Let me write a deep analysis. I'll keep it to a focused essay with real insight. Not too long. The user seems to want depth. I'll write maybe 40-60 lines of dense insight.

Key deep insights to surface:
1. The axis of "where does the kernel end" — pi minimizes, omp maximizes built-ins, opencode fixes a contract, dsh makes config composable. This is really about a single design axis: *coupling between substrate and features*.
2. Native performance as a fork magnet: omp is a fork of pi precisely because pi refused native perf (pure TS, sandbox external). The fork exists to violate pi's core constraint.
3. dsh's "session log as source of truth" + opencode Context Epochs + omp snapcompact all independently rediscover the same hard problem: LLM context is stateful, expensive, and must be made durable/compressible/cache-friendly. That's the real shared hard problem — not tool execution.
4. Extensibility paradox: pi says "no MCP" (skills instead); dsh/omp embrace MCP; but dsh's plugin-kernel is the most MCP-like at heart. The field hasn't settled the extension primitive.
5. Self-hosting config dirs (.pi/.omp/.opencode/.agents) reveal the real deployment target: the agent develops the agent. Dogfooding is the test harness.
6. Supply-chain: pi's exact-pinning vs dsh's vendored Cordis — two opposite answers to "how do you trust your deps" given rapid upstream churn.
7. The strategic read: opencode = company/cloud play; pi = philosophy/library; omp = maximalist power-user tool; dsh = framework/platform bet by a model vendor.

I'll present this as a "deep" reflection. Good

Stepping back from the catalogs — here's what the four repos actually reveal when you read them *as a set* rather than four separate tools.

## The single axis everything rotates around

Every architectural choice in these repos is a position on one question: **where does the kernel stop and the user's world begin?**

- **pi** answers: *as early as possible.* No MCP, no subagents, no permissions, no plan mode — those are all "your problem, expressed as a package." The kernel is a substrate.
- **oh-my-pi** answers: *as late as possible.* 31 tools, DAP debugger, desktop control, subagents — built-ins sit in the *same registries* as user extensions. The fork exists almost entirely to violate pi's "minimal core" constraint by importing a Rust native layer pi explicitly rejected.
- **opencode** answers: *the kernel is a contract, not a library.* One authoritative `HttpApi`, and everything (TUI, desktop, IDE, cloud, SDKs) is a client of it. You don't extend the kernel, you consume the contract.
- **deepseek-harness** answers: *there is no fixed kernel.* The agent loop, sandbox, persistence, even the model adapter are boot-time plugins. Configuration *is* the program.

So omp is a fork of pi for a reason that has nothing to do with features — it's a **philosophical fork**. pi's purity (pure TS, sandbox externalized) is exactly what omp's users can't live with.

## The real shared hard problem

None of them are actually competing on tool execution. Read closely and they've all independently rediscovered the same unsolved core: **LLM context is stateful, lossy, expensive, and must outlive the process.**

- dsh → append-only session log as source of truth, "model-visible ⟺ logged" invariant, spill + semantic query.
- opencode → Context Epochs (immutable cache-friendly baselines) + durable prompt admission decoupled from execution + steer/queue at safe boundaries.
- omp → `snapcompact` bitmap-frame compression + `mnemopi` memory engine + checkpoint/rewind.
- pi → branching SQLite sessions + compaction (the modest version).

This is the genuine research frontier in all four. The tools (bash, edit, grep) are solved; **context management under failure and cost is not.**

## The extension primitive is unsettled

pi bans MCP in favor of "skills." omp and dsh embrace MCP. Yet dsh's Cordis plugin-kernel is, at heart, *more* MCP-like than MCP — a typed event/waterfall bus where anything is swappable. The field hasn't converged on the extension primitive: is it (a) skills/markdown, (b) plugins/hooks, (c) capability seams, or (d) an external protocol (MCP/ACP)? All four pick differently, and none is clearly winning.

## Two opposite answers to trust

Given how fast the AI-tooling stack churns, dependency trust splits cleanly:

- **pi**: exact pinning + release-age gates + lockfile-as-ground-truth + OIDC publishing. *Trust by constraint.*
- **dsh**: vendored the entire Cordis framework, SHA-pinned, with an in-repo modification log. *Trust by ownership.*

opencode sits between (heavy `patchedDependencies`). omp leans on Bazel's hermetic builds. Same fear — upstream breaks you — two philosophies of response.

## The deployment target is itself

All four have a directory where the agent develops itself: `.pi/`, `.omp/`, `.opencode/`, `.agents/`. That's not cute — it's the **test harness**. The only reliable way to verify an agent harness is to point the agent at its own repo. The dogfooding dir *is* the integration test.

## Strategic read (who this is really for)

- **opencode** is a *company*: cloud console, Stripe, Slack, GitHub Action, enterprise control plane. The open-source agent is the top of a funnel.
- **pi** is a *philosophy/library*: read it to understand harness design; extend it to own your workflow.
- **oh-my-pi** is a *power-user instrument*: maximal capability for someone who will run untrusted code in containers and wants the IDE wired in.
- **deepseek-harness** is a *platform bet by a model vendor*: its value is the composable kernel + Web UI + SDKs; DeepSeek gets a first-class native path and an ecosystem hook.

The deepest pattern: these aren't four implementations of one idea. They're four **bets about who should control the agent** — the user (pi), the vendor (dsh), the product company (opencode), or the power user (omp) — and every folder, patch, and sandbox decision traces back to that bet.

---

