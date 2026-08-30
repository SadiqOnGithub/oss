# AI Coding Agent Harnesses: Comparative Report

**Projects compared:** `deepseek-harness` · `opencode` · `pi` · `oh-my-pi`
**Date:** 2026-08-24 (facts refreshed 2026-08-29)
**Scope:** Capabilities, architecture, extensibility, tooling/testing, and how the four projects relate to each other.
**Checkout verified:** pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2
**Companions:** [repo-guides.md](../README.md) · [pi-files-guide.md](../maps/pi-files-guide.md) · [oh-my-pi-files-guide.md](../maps/oh-my-pi-files-guide.md) · [opencode-files-guide.md](../maps/opencode-files-guide.md) · [deepseek-harness-files-guide.md](../maps/deepseek-harness-files-guide.md) · [first-hour.md](../how-to/first-hour.md) · [prompt-traces.md](../how-to/prompt-traces.md) · [add-a-tool.md](../how-to/add-a-tool.md) · [glossary.md](../reference/glossary.md) · [config-and-dogfooding.md](../how-to/config-and-dogfooding.md) · [drive-from-process.md](../how-to/drive-from-process.md) · [structure-comparison.md](./structure-comparison.md)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Lineage & Relationships](#2-lineage--relationships)
3. [Project Profiles](#3-project-profiles)
   - [3.1 pi (pi-mono)](#31-pi-pi-mono)
   - [3.2 oh-my-pi (omp)](#32-oh-my-pi-omp)
   - [3.3 opencode](#33-opencode)
   - [3.4 deepseek-harness (dsh)](#34-deepseek-harness-dsh)
4. [Head-to-Head Comparison Matrix](#4-head-to-head-comparison-matrix)
5. [Architecture Deep Dive](#5-architecture-deep-dive)
6. [Capability Comparison](#6-capability-comparison)
7. [Extensibility Models Compared](#7-extensibility-models-compared)
8. [Engineering Quality: Testing, Build, Release](#8-engineering-quality-testing-build-release)
9. [Which One Should You Use?](#9-which-one-should-you-use)

---

## 1. Executive Summary

All four are **open-source agentic coding harnesses** — terminal-first (or multi-surface) tools that wire an LLM to a codebase with file editing, shell execution, and session management. They differ radically in philosophy:

| Project | One-liner | Core bet |
|---|---|---|
| **pi** | Minimal, self-extensible agent harness | "No MCP. No sub-agents. No permission popups." Everything is a user-supplied TypeScript extension |
| **oh-my-pi (omp)** | Pi fork turned batteries-included coding agent | Native Rust performance core + maximal built-in tool surface (README **31** tools / 29 names in `builtin-names.ts` + hidden; LSP/DAP) |
| **opencode** | The productized multi-surface coding agent | Client/server split with one authoritative HTTP API; TUI, desktop, IDE, web, cloud all as clients |
| **deepseek-harness (dsh)** | Plugin-kernel harness where *everything* is a plugin | Vendored Cordis framework; model adapter, tools, sandbox, even the agent loop are swappable config |

Key facts at a glance:

- **pi** (`@earendil-works/pi-*`, Earendil Works, pi.dev): 100% erasable-syntax TypeScript on Node ≥22 / Bun; deliberately minimal core; packages distributed via npm/git.
- **oh-my-pi** (`@oh-my-pi/pi-coding-agent`, omp.sh, by Can Bölük / Stencil Labs): fork of pi-mono; ~80k lines of Rust compiled into one N-API addon (embedded bash, in-process grep/coreutils, AST tools); 60+ providers, 31 built-in tools.
- **opencode** (`opencode-ai`, anomalyco): Bun + Effect-TS monorepo; strict layered architecture (Schema → Core → Server → Client → SDK); SST/AWS+Cloudflare cloud ecosystem; VS Code extension and GitHub Action.
- **deepseek-harness** (`@deepseek-ai/dsh`, v0.1.1-rc.2 developer preview, MIT): DeepSeek's plugin-based harness; capability seams (LLM, shell, sandbox, FS, LSP, subagent…) each with swappable providers; Web UI + headless + ACP + JSON-RPC/TS/Python SDKs.

Notable cross-pollination: **dsh is not a pi fork**, but it uses `@earendil-works/pi-ai` as an optional multi-provider LLM adapter behind its own `llm` seam. **omp is a direct fork of pi**, keeping `pi-*` package names and an upstream-sync process.

---

## 2. Lineage & Relationships

```
                    ┌──────────────────────────────┐
                    │   pi (badlogic/pi-mono)      │
                    │  minimal-core agent harness  │
                    └──────────┬───────────────────┘
                               │ direct fork (tracked sync point,
                               │ keeps @…/pi-* package names)
                               ▼
                    ┌──────────────────────────────┐
                    │   oh-my-pi (can1357)         │
                    │  batteries-included rewrite  │
                    │  + Rust native core          │
                    └──────────────────────────────┘

┌──────────────────────────────┐        uses as optional lib
│  deepseek-harness (@deepseek)│◄───── @earendil-works/pi-ai
│  Cordis plugin kernel        │       (provider wire protocols
│  NOT a pi fork               │        & model catalogs only)
└──────────────────────────────┘

┌──────────────────────────────┐
│  opencode (anomalyco)        │   independent lineage
│  Effect-TS client/server     │
└──────────────────────────────┘
```

---

## 3. Project Profiles

### 3.1 pi (pi-mono)

> *"No MCP. No sub-agents. No permission popups. No plan mode. No to-dos. No background bash."*

**What it is.** A terminal coding agent built on a deliberately minimal core that users adapt through **TypeScript Extensions, Skills, Prompt Templates, and Themes** distributed as shareable "Pi Packages" over npm/git. Workflow features are intentionally excluded from core; they belong in extensions/packages. There is no built-in permission system either — sandboxing is delegated to containers/micro-VMs (documented patterns: Gondolin micro-VM extension, Docker, OpenShell).

**Tech stack.**
- 100% TypeScript using only *erasable* syntax (Node strip-only mode — no enums/namespaces/parameter properties), enforced by a CI check.
- Node ≥22.19 (native `node:sqlite`) and Bun for standalone binaries.
- npm workspaces, Biome lint/format, `tsgo` (TypeScript native preview), Vitest + node:test, esbuild/tsx, Husky.
- Length-prefixed **CBOR** framing for its remote-session protocol.

**Monorepo layout** (lockstep versioning across all packages):

| Package | Purpose |
|---|---|
| `pi-tui` | Minimal TUI framework: differential rendering, CSI 2026 sync output, alt/main screen renderers, editor/select-list/markdown components, Kitty/iTerm2 inline images |
| `pi-telemetry` | Vendor-neutral telemetry contracts (callback-based spans), no-op + in-memory reference adapters |
| `pi-ai` | Unified multi-provider LLM API (~15+ providers incl. OpenAI, Anthropic, Google, Bedrock, Azure, Groq, xAI, OpenRouter, Mistral); streaming, thinking levels, image I/O, auth resolution, cost tracking, mid-session model hand-off |
| `pi-agent-core` | General-purpose stateful agent runtime: tool loop, event streaming, attachments, pluggable session backends |
| `session-backends/sqlite-node` | Optional SQLite backend for `pi-agent-core` hosts (repository, migrations, FTS). **Not imported by the `pi` CLI**, which writes JSONL |
| `pi-coding-agent` | The `pi` CLI: interactive TUI, print/JSON mode, RPC mode, SDK embedding; read/bash/edit/write tools; sessions w/ branching & compaction; extensions/skills/themes/packages |
| `pi-protocol` | Transport-neutral CBOR schemas + byte-stream framing for remote sessions |
| `pi-client` | Transport-neutral remote-session client: exclusive/shared session leases, snapshot-authoritative state |
| `pi-server` | Experimental session server composing pluggable transport listeners |
| `pi-evals` | Model-backed behavioral evals via `vitest-evals`, isolated temp dirs |

**Capabilities.**
- Four run modes: interactive TUI, print/JSON one-shot, RPC (for process integration), embeddable SDK.
- Built-in tools limited to read/bash/edit/write; per-invocation disable lists; read-only mode.
- Sessions: the `pi` CLI persists **JSONL** under `~/.pi/agent/sessions/` (`session-manager.ts`, branching + compaction). `packages/session-backends/sqlite-node` is an optional `pi-agent-core` library with FTS — the CLI does not import it.
- Extensions with lifecycle hooks; custom providers; skills explicitly positioned as the MCP replacement; themes; prompt templates.
- Agent-first development culture: AGENTS.md rules for humans and AI; maintainers publish their dev sessions as OSS datasets on Hugging Face.

**Architecture highlights.**
- **Minimal-core doctrine**: hook points require maintainer discussion; PRs bloating core are rejected.
- **Layered stack**: `tui → ai → agent (+backends) → coding-agent`; remote access splits cleanly `protocol → server → client`.
- **Snapshot-authoritative RPC**: server/client sync via authoritative snapshots; progress events are transient UI hints never folded into state; leases enforce single-writer ownership; auth happens at transport layer before protocol bytes flow.
- **Transport neutrality everywhere**: protocol/client/telemetry avoid runtime-specific imports; the SQLite backend accepts an injected DB factory.
- **Supply-chain hardening as architecture**: exact-pinned deps, `min-release-age=2` days, lockfile-as-ground-truth with pre-commit gating, generated shrinkwrap + lifecycle-script allowlist, `npm ci --ignore-scripts` in CI, npm trusted publishing (OIDC).

### 3.2 oh-my-pi (omp)

> *"A coding agent with the IDE wired in."* — fork of pi-mono, rewritten as a coding-first surface.

**What it is.** A batteries-included terminal coding agent: CLI/TUI plus SDK, RPC, and ACP surfaces. Where pi strips features out, omp builds them in — subagents, slash commands, debugger integration, desktop control, collaboration sharing — while pushing hot-path work into a large native Rust layer. Headline stats: **60+ providers, 31 built-in tools, 14 LSP ops, 28 DAP ops, ~80k lines of Rust core** (+~80k vendored: brush bash fork + dozens of in-process CLI utilities).

**Tech stack.**
- **TypeScript** on **Bun** (≥1.3.14) for all product logic (agent, tools, TUI, providers).
- **Rust** (edition 2024, pinned nightly) compiled to one platform-tagged **N-API `.node` addon** (`pi_natives` cdylib).
- **Python** for `robomp` (GitHub triage bot) and `omp-rpc` (typed Python RPC client).
- **Bazel** (hermetic shipping builds of the Rust addon, zig cc cross-toolchain) alongside **Cargo** for local iteration; bun workspaces; Biome; Nix flake.

**Repo layout — key TS packages:** `coding-agent` (main CLI/SDK), `ai` (multi-provider client), `catalog` (bundled model DB), `agent`, `tui`, `natives` (N-API loader), `hashline` (content-hash patch language behind the edit tool), `omptype` (ArkType-compatible validation), `utils`, `wire` (collab protocol), `mnemopi` (SQLite memory engine), `snapcompact` (context compression + SQuAD evals), `stats` (usage dashboard), `collab-web` (browser guest + relay), `browser-relay` (Chrome extension driving real tabs), `metaharness` (benchmark runners/Harbor storage/dashboard).

**Rust crates:** `pi-natives` (~25k LoC; grep, diff, ast-grep, PTY, clipboard, highlighting, desktop control, tokenizers, sixel…), `pi-shell` (~38k LoC embedded bash wrapping vendored brush), `pi-builtins` (bash builtins + ported coreutils/findutils/sed/jq running zero-fork), `vendor/brush-core`, `pi-walker` (parallel ignore-aware FS walker), `pi-iso` (task isolation: APFS clones, btrfs/zfs reflinks, overlayfs, projfs), `pi-ast` (tree-sitter + ast-grep, 50+ grammars), `pi-voice` (audio capture/playback, Opus, WebRTC).

**Capabilities (selection).**
- Four entry points: interactive TUI, one-shot `-p`, Node SDK (`createAgentSession`), stdio NDJSON RPC + ACP for editors like Zed.
- **Tools:** README markets **31**. `BUILTIN_TOOL_NAMES` lists **29** (`read`, `write`, `bash`, `edit`, `glob`, `grep`, `ast_edit`, `ast_grep`, `ask`, `debug`, `eval`, `github`, `lsp`, `inspect_image`, `browser`, `computer`, `checkpoint`, `rewind`, `security_scan`, `task`, `hub`, `todo`, `web_search`, `memory_edit`, `retain`, `recall`, `reflect`, `learn`, `manage_skill`). Hidden: `yield`, `goal`, `think`. `tts.ts` / `image-gen.ts` exist on disk and complete the README 31 with generate_image + tts. Plus in-process bash, DAP, browser, computer, isolated-worktree `task`.
- **Model routing**: 10 roles (default/smol/slow/plan/vision/designer/task/advisor/commit/tiny), fallback chains, path-scoped models, round-robin credentials.
- **Advisor model**: second reviewer model watching every turn on its own context.
- **Time-traveling stream rules**: regex-triggered mid-stream system-reminder injection with retry.
- **Collab**: `/collab` share link + QR; teammates join by terminal or browser; client-side encryption.
- **Interop**: inherits rules/skills/MCP configs from `.claude`, `.cursor`, `.windsurf`, `.gemini`, `.codex`; reads `pr://1428`, `conflict://N`, ssh paths as URLs inside every file tool.
- Secrets redaction before provider-visible text leaves the process.

**Architecture highlights.**
- **One N-API addon, no fork/exec on hot paths**: ripgrep-as-library, embedded bash, in-process utilities → single binary across macOS/Linux/Windows without WSL.
- Blocking Rust work runs on the libuv pool with panic-catching at the napi boundary (panics become rejected promises).
- Bun workers re-enter `cli.ts` via hidden argv selectors so compiled binaries need no separate worker entries; prompts live in static `.md` imports, not code-built strings.
- **Dual build split**: Cargo authoritative locally (rust-analyzer, nextest); **Bazel ships hermetically** — rules_rust + zig cc, crate_universe from Cargo.lock, 8 platform/ISA targets including musl and dual AVX2/baseline x64.
- Extension modules share the same tool/slash-command/hotkey registries as built-ins; MCP support; managed skills marketplace; settings-gated `xd://` discoverable off-by-default tools.

### 3.3 opencode

> *"The open source AI coding agent"* — TUI, desktop app, IDE, browser, and cloud, all clients of one HTTP API.

**What it is.** A heavily productized coding-agent ecosystem: the agent runs as a local **server** exposing an authoritative HTTP API (SSE event streams); every UI (TUI, Electron desktop, VS Code extension, shared web view, SDKs) is a client. 75+ providers via models.dev + Vercel AI SDK adapters. Ships a cloud console (Stripe billing), stats site, GitHub Action (`/opencode fix this issue` → PR), Slack bot, and enterprise control-plane.

**Tech stack.**
- Bun 1.3 monorepo (workspaces + catalog deps), Turborepo orchestration.
- **Effect v4** pervasive: Effect Schema domain types, `HttpApi` for the API, Drizzle ORM + SQLite for durable storage, Hono HTTP server.
- SolidJS/SolidStart + Tailwind v4 + Vite for web UIs; **@opentui/solid** for the TUI; Electron desktop; node-pty terminals; tree-sitter/WASM.
- oxlint + tsgolint, prettier, husky, Playwright e2e, SST v4 IaC (Cloudflare home + AWS us-east-1, Stripe, PlanetScale, Honeycomb, Sentry).
- Heavy upstream patching (patchedDependencies: effect, ai-sdk providers, MCP SDK, solid-js…).

**Monorepo layout (dependency direction: Schema ← Core/Protocol ← Server ← Client ← sdk-next).**

Core chain:

| Package | Purpose |
|---|---|
| `schema` | Shared public domain schemas (Effect Schema records), dependency-free leaf |
| `core` | Domain core: sessions, agents, config, permissions, tools, LSP, MCP, plugins, git, storage |
| `opencode` | Main binary/CLI entrypoint wiring everything into the distributable |
| `llm` | Provider/protocol layer: wire protocols, cache policy, tool runtime, provider errors |
| `protocol` | Public API composition: endpoint paths, envelopes, cursors, streams |
| `server` | Hono/Effect server hosting the authoritative public `HttpApi` |
| `httpapi-codegen` | Compiles the HttpApi into SDK Contract IR → generated clients |
| `client` (+`/effect`) | Generated Promise + Effect API clients (browser-safe) |
| `sdk` / `sdk-next` | Legacy JS SDK; Effect-native in-process host ("Embedded OpenCode") |

Surfaces & ecosystem: `tui`, `app` (web/desktop frontend), `desktop` (Electron), `ui` + `session-ui` + `storybook` (shared components), `console/*` (cloud console: app/core/functions/mail/resource/support), `stats/*`, `enterprise`, `identity`, `slack`, `containers` (sandboxing), `codemode` (Effect-native confined code execution over schema-described tools), `web` (docs/marketing site), `plugin` (public plugin API), `http-recorder` (cassette record/replay testing). Plus `sdks/vscode` (VS Code extension), `github/` (official GitHub Action), `infra/` (SST modules: app, console, enterprise, lake, stats, monitoring), `specs/` (design specs for v2 session/tools/provider/config/storage).

**Capabilities.**
- Rich SolidJS TUI with two built-in agents switchable via Tab: `build` (full access) and `plan` (read-only), plus a `general` subagent invoked with `@general`.
- Tools: file read/edit/write, bash, glob/grep, webfetch, todo, patch/diff, snapshot/checkpointing, permission-checked skills, question prompts, PTY terminals.
- Agents/subagents configurable via markdown/JSON; **permission system** with allow/deny rules and per-command approval.
- Sessions: persistent SQLite storage, share/public links, compaction, steering vs queued prompts, resumption.
- Extensibility: plugin API (`@opencode-ai/plugin` with hooks including PTY environment observation), MCP client, LSP, custom providers/models/commands, AGENTS.md instruction discovery.
- Integrations: VS Code extension, Zed support, ACP, GitHub Action, Slack bot, macOS/Windows/Linux desktop apps.

**Architecture highlights.**
- **Strict layered dependency direction**, mechanically enforced: Client runtime never imports Core/Server; Protocol owns endpoint/middleware placement; Server supplies concrete keys.
- **Single authoritative HttpApi**: codegen compiles it into SDK Contract IR generating both a zero-Effect Promise client and a rich Effect client. Generated files are never hand-edited.
- **V2 session runtime** (extensively specified in `specs/v2/`): durable prompt admission decoupled from model execution; serialized process-local "Session Drains"; steer/queue semantics at safe provider-turn boundaries; crash-safe history without durable-execution identity.
- **System Context algebra**: composable typed Context Sources (AGENTS.md instructions, date, skill guidance…) rendered into a baseline prompt with immutable provider-cache-friendly **Context Epochs**; changes emitted as durable mid-conversation system messages rather than mutating the baseline.
- Tool-output bounding: oversized outputs truncated to previews with full text spilled to managed temp files.
- Two deliberately different event streams: durable replayable per-session SSE vs live-only instance-wide subscription.

### 3.4 deepseek-harness (dsh)

> *"Everything is a plugin."* — plugin-based agent harness from DeepSeek AI, developer preview.

**What it is.** An open-source, plugin-based coding/automation harness where the model adapter, tool registry, session log, sandbox policy — **and even the agent loop itself** — are plugins mounted into a composable tree at boot. Ships as: a CLI launcher (`npx @deepseek-ai/dsh web`), a **Web UI** (React client + Node host at `127.0.0.1:3080`), a headless one-shot runner, an **ACP server**, a **JSON-RPC server + TS SDK**, and a **Python SDK** driving a bundled runtime over newline-delimited JSON-RPC on stdio. Single-exe distribution via the Python runtime carrier. Explicit no-compatibility-promise until first tagged release.

**Tech stack.**
- Strict-mode ESM-only TypeScript; Node `^22.19 || >=24`; pnpm 11 workspaces.
- **Cordis** plugin/context/event/waterfall framework (from cordiverse), **source-vendored** under `vendor/`, rescoped to `@deepseek-ai/*`, pinned SHAs with exhaustive modification logging.
- Vite + React web client; VitePress docs site; tsc project references + tsdown bundling.
- Vitest (unit/e2e/keyless snapshots/web stress/perf), fast-check property tests, jscpd duplication checks, knip/publint, lefthook, ~30 custom gate scripts.
- C++/N-API addon: **Landlock self-restrict-then-exec launcher** (Linux sandboxing, per-platform npm packages); bubblewrap/Landlock/Seatbelt + Windows ACL restricted-token sandbox backends; patched node-pty.
- Python SDK + PyPI-distributed runtime binary carrier.

**Repo layout (50 `packages/` groups, nested `@deepseek-ai/dsh-*` packages; 23 `tool-*` consumers).** Highlights:

| Area | Packages |
|---|---|
| Spine | `core` (session, system-prompt, tools, agent registry, **swappable agent-loop**), `api` (remote BFF + Typert RPC gateway), `typert` (type-graph registry) |
| Model | `llm` seam + `llm-deepseek`, **`llm-pi-ai`** (multi-provider via `@earendil-works/pi-ai`), `llm-retry`, `token-meter` |
| Execution | `shell` (local/pwsh providers), `subprocess`, `terminal` (persistent PTY), `code-runtime` ("Code Mode" worker-thread execution), `sandbox` (bwrap/Landlock/Seatbelt/Windows ACL), `fs` (+ policy fencing), `lsp`, `web` (search/fetch: DeepSeek/Exa/Perplexity), `e2b` (cloud-sandbox POC) |
| Orchestration | `subagent` (fork/spawn in-process, Claude Code, Codex, ACP, DSH-SDK providers + delegation tools), `jobs` (background jobs), `workflow` (+ ralph tools), `todo`, `plan`, `guard` (loop hygiene + tool deadlines), `extensions` (**agent inspects/mounts/unmounts its own plugins live**) |
| Data plane | `session` (JSONL & SQLite persistence, projections, telemetry/OTel, LLM titles), `session-query` (bounded reads, lineage, semantic filter, FTS), `attachment`/`spill`/`storage`/`workspace`, `compaction`, `context` |
| Delivery | `preset`/`bundle` (`--profile` installable bundles: base, web-app, headless), `boot`, `sdk` (JSON-RPC + TS client), `acp`, `host`/`client` (Web GUI halves; ~30 React `ui-*` plugins) |
| Compat | `hooks` (Claude Code/Codex hook bridges), `mcp`, `skill` |

**Capabilities.**
- Agent loop event flow: `turn/start` → prompt assembly → `agent/pre-step` waterfall (listeners can rewrite/reject input) → `agent/request` → `llm/stream` → tool pipeline `pre-execute → execute → post-execute` → repeat. The loop is itself a swappable plugin.
- Tools: bash/pwsh, persistent PTY terminals, FS read/write/edit + bash-backed discovery, LSP, web search/fetch, todo_write, plan mode, ask-user, subagent delegation/control/report, background `job_*`, workflow/ralph, skills catalog/loader, MCP client tools, session-query. Tool UI render intent (`generic`/`terminal`/`diff`) is part of tool design.
- Providers: native DeepSeek adapter plus generic multi-provider adapter on pi-ai (OpenAI/Anthropic/etc.), retry policies, token metering.
- Safety: approvals + permission presets, OS-level sandbox backends, FS write fencing, tool deadlines.
- Session intelligence: compaction, spill of oversized tool results, LLM-generated titles, OTel telemetry, checkpointing, forking.
- Self-modification: the `extensions` capability lets the agent inspect its own Cordis graph and mount/unmount plugins live; HMR reload; profile patch layers.

**Architecture highlights.**
- **Cordis kernel**: services, declaration-merged typed events (`SessionEventMap`), reversible effects with disposers, waterfalls requiring explicit `next()`.
- **Profiles & bundles**: a running dsh is a config tree composed from ordered patch layers — base bundle → mode bundle (web-app/headless) → profile `cordis.patch.yml` → home patch → `--patch` overlays; `--dump-config` reveals the composed tree.
- **Capability seams**: each capability = Service Definition + Service Providers + Consumers; extensions depend only on definitions, so swapping one provider relocates dependent capabilities together (e.g., a remote sandbox moves Bash, PTY, and LSP at once).
- **Session log as source of truth**: append-only events with a build-time-enforced "model-visible ⟺ logged" invariant; `deriveMessages()` projects model history; fork/resume/transcripts/telemetry all derive from the stream; monotonic SQLite schema versions.
- Documented extension-point table maps every goal (new provider/tool/command/backend/UI node) to a mechanism — "plugins, not loop changes."
- Notably: **no first-party TUI** — the flagship UX is the Web UI; terminal apps mount via profiles.

---

## 4. Head-to-Head Comparison Matrix

| Dimension | pi | oh-my-pi | opencode | deepseek-harness |
|---|---|---|---|---|
| Maturity | Stable-ish, lockstep 0.x | Actively developed fork | Production product + cloud | Developer preview (0.1.1-rc.2), no compat promise yet |
| License / origin | MIT, Earendil Works | MIT, Stencil Labs / Can Bölük | (open) anomalyco | MIT, DeepSeek AI |
| Primary language(s) | Erasable TS only | TS (Bun) + Rust nightly + Python | TS (Bun) + Effect v4 | Strict ESM TS + vendored Cordis; C++ Landlock addon; Python SDK |
| Runtime | Node ≥22.19 / Bun binaries | Bun (Node compat secondary) | Bun 1.3 | Node ^22.19\|\|≥24 (pnpm) |
| Primary UX | TUI (+ print/RPC modes) | TUI (+ RPC/ACP) | TUI, desktop, web, IDE, GitHub, Slack | **Web UI**, headless, ACP, SDKs (no first-party TUI) |
| Package manager / build | npm workspaces, esbuild | bun ws + Cargo + Bazel (shipping) + Nix | Bun ws + Turborepo | pnpm ws, tsc refs + tsdown |
| LLM providers | ~15+ (own catalog, generated from live data) | 60+, 1000+ models, 10 routing roles | 75+ via models.dev + AI SDK | DeepSeek native + others via pi-ai adapter |
| Built-in tools | 4 coding (`read`/`bash`/`edit`/`write`; SDK also has find/grep/ls) | README 31; 29 in `builtin-names.ts` + hidden yield/goal/think; AST/DAP/browser/desktop/memory | **12** in `packages/core/src/tool/builtins.ts` (apply_patch, bash, edit, glob, grep, question, read, skill, todowrite, webfetch, websearch, write) + MCP/plugin tools | **23** `tool-*` packages (bash/pwsh, PTY, fs, lsp, web, jobs, workflow, subagents, MCP, session-query, …) |
| Subagents | ❌ by design (extension territory) | ✅ parallel, isolated worktrees | ✅ `@general` + configurable agents | ✅ multiple providers (fork/spawn/Claude Code/Codex/ACP) |
| Permissions/approvals | ❌ none (sandbox externally) | Settings-gated tools, secrets redaction | ✅ allow/deny rules, per-command approval, plan/build agents | ✅ approvals, permission presets, sandbox backends |
| Sandboxing | Externalized (Docker/Gondolin docs) | `pi-iso`: APFS clones, reflinks, overlayfs, projfs | Containers package | bwrap/Landlock/Seatbelt/Windows ACL + E2B POC |
| MCP | ❌ explicit non-goal (skills replace it) | ✅ + inherits .claude/.cursor/etc. configs | ✅ MCP client | ✅ MCP client |
| LSP | ❌ | ✅ 14 ops, rename-aware | ✅ | ✅ (capability + tool) |
| Debugger (DAP) | ❌ | ✅ 28 ops (lldb/dlv/debugpy) | ❌ | ❌ |
| Sessions storage | CLI: JSONL under `~/.pi/agent/sessions/` (`session-manager.ts`). sqlite-node is a library for other `pi-agent-core` hosts | JSONL session tree (`~/.omp/agent/sessions/…jsonl`) + mnemopi SQLite **memories** | SQLite via Drizzle/Effect + V2 `session_input` admit table | JSONL or SQLite, append-only log as truth |
| Compaction | ✅ | ✅ + snapcompact bitmap-frame compression | ✅ | ✅ + spill + semantic query |
| Remote/collab | Experimental CBOR server/client, leases | `/collab` links, browser guests, encrypted | Share links, cloud console | Web host/client, JSON-RPC SDK, ACP |
| Telemetry | Vendor-neutral contracts pkg | Local stats dashboard | Cloud stats site + Honeycomb/Sentry | OTel + local observability |
| Native perf layer | None (pure TS) | ~80k LoC Rust N-API addon, zero-fork shell/grep/utils | opentui Go-core heritage (TUI binaries) | Landlock launcher (C++), otherwise pure TS |
| Config model | Extensions/skills/themes/packages | models.yml, settings, skills marketplace | Markdown/JSON agents + config files | Ordered patch layers → profiles/bundles |

---

## 5. Architecture Deep Dive

### 5.1 Kernel / composition model

| | Kernel concept | Swap granularity |
|---|---|---|
| **pi** | Plain layered libraries; composition happens in `pi-coding-agent`'s bootstrap | Extensions hook lifecycle events; whole features live outside core |
| **oh-my-pi** | pi's layering + capability providers (tools/skills/settings discovered via `loadCapability`) | Extensions register into the same registries as built-ins; Rust behind one N-API boundary |
| **opencode** | Effect layers composing Schema→Core→Server; single HttpApi as contract | New Context Sources, plugins (hook-based), custom providers/models/commands |
| **dsh** | **Cordis plugin tree**: services, events, waterfalls, disposers | Anything — including the agent loop, session log implementation, sandbox backend — via patch layers |

The philosophical spectrum: **pi minimizes the kernel** → **opencode fixes the kernel and generates everything downstream from one API contract** → **dsh makes even the kernel's contents negotiable at boot** → **omp maximizes what ships inside the kernel**.

### 5.2 Client/server and protocols

- **opencode**: strongest client/server story. One authoritative Effect `HttpApi`; codegen produces both Promise and Effect clients; same API executed in-process ("Embedded OpenCode") or over the network; durable replayable per-session SSE vs live instance-wide events — guarantees are explicit and different by design.
- **pi**: experimental remote sessions with transport-neutral CBOR framing (4-byte length prefix), snapshot-authoritative sync, exclusive/shared leases, ID-correlated requests, transport-layer auth. Progress events never mutate state.
- **dsh**: JSON-RPC (TS SDK + Python SDK over stdio NDJSON), ACP server for editors, Typert RPC gateway for the web host, Claude Code/Codex hook wire protocols, MCP.
- **omp**: stdio NDJSON RPC + ACP (Zed etc.), collab wire types in `packages/wire`, browser guest relay; Python client (`omp-rpc`) exists for the robomp bot.

### 5.3 Session/state architecture

- **dsh** is the most rigorous: append-only session log is the source of truth; "model-visible ⟺ logged" invariant checked at build time; everything (resume, fork, transcripts, telemetry, titles) derived from the event stream; bounded reads/lineage/FTS in a query layer.
- **opencode** specifies deeply (specs/v2): durable prompt admission decoupled from execution, serialized drains, steering at safe turn boundaries, Context Epochs for provider-cache stability.
- **omp/pi**: both persist the conversation as a **JSONL session tree** (`session-manager.ts` / omp `src/session/`), with branching (pi) and compaction (both). omp adds checkpoint/rewind, snapcompact, and a separate SQLite memory engine (mnemopi). pi's sqlite-node package is not the CLI store.
- All four treat oversized tool output as a first-class problem: opencode spills to temp files, dsh has an explicit spill package, omp bounds/truncates via native helpers.

### 5.4 Performance strategy

- **omp** stands alone here: in-process Rust for grep/shell/AST/highlight/PTY/walking; embedded bash with 58+ ported utilities; zero fork/exec on hot paths; hermetic Bazel builds producing musl + AVX2/baseline variants.
- **opencode/dsh/pi** accept JS-speed tool implementations; their performance work goes into rendering (opentui differential rendering, pi-tui CSI 2026 synchronized output) rather than tool execution.

---

## 6. Capability Comparison

### Tools & integrations

| Capability | pi | omp | opencode | dsh |
|---|---|---|---|---|
| File read/edit/write | ✅ | ✅ + hashline content-hash patches + AST edit | ✅ + patch/diff + checkpoints | ✅ + policy fencing |
| Shell | bash | **in-process embedded bash** + 46+ builtins | bash + PTY | bash/pwsh + persistent PTY |
| Code search | glob/grep via tools | native rg-as-library + ast-grep | glob/grep | fs discovery + bash |
| LSP | ❌ | ✅ | ✅ | ✅ |
| Debugger | ❌ | ✅ DAP | ❌ | ❌ |
| Browser automation | ❌ | ✅ Puppeteer/Electron/Chrome relay | webfetch only | web fetch/search providers |
| Desktop/computer use | ❌ | ✅ windows/screenshots/input/AX tree | ❌ | ❌ |
| Background jobs | ❌ (non-goal) | ✅ task subagents/worktrees | ❌ core (plugins possible) | ✅ job_* + workflow/ralph |
| Memory systems | ❌ | ✅ mnemopi retain/recall/reflect/learn | ❌ core | session-query semantic retrieval |
| Image/audio gen | via pi-ai | generate_image/inspect_image/tts/voice | ❌ core | via llm/web seams |
| Skills | ✅ (replaces MCP) | ✅ marketplace + cross-tool import | ✅ permission-checked | ✅ catalog + loader |
| Hooks (Claude Code-style) | extensions instead | extensions + stream rules | plugin hooks | ✅ dedicated bridges + guard/deadlines |

### Model routing & providers

- **omp** is deepest: 10 role slots, fallback chains, path-scoped models, round-robin credentials, custom OpenAI-compatible declarations, per-provider reasoning-effort normalization with 400/422 fallback parsing.
- **opencode** leverages the AI SDK + models.dev for breadth; mid-session switching supported.
- **pi** generates catalogs from live provider data; auth resolution across env/OAuth/credential store; mid-session hand-off.
- **dsh** centers on DeepSeek but gains breadth by delegating to pi-ai behind its adapter seam — the clearest evidence that these projects compose rather than merely compete.

---

## 7. Extensibility Models Compared

| | Mechanism | Distribution | Can change core behavior? |
|---|---|---|---|
| **pi** | TS extensions (lifecycle hooks), skills, themes, prompt templates | npm/git "Pi Packages" | Only via hooks maintainer approves adding; core PRs rejected if bloating |
| **omp** | TS extensions sharing built-in registries (tools/slash/hotkeys), MCP, skills marketplace, `xd://` devices, stream rules | Files/marketplace/npm | Yes — extensions sit in the same registries as built-ins |
| **opencode** | Plugins (hooks, e.g., PTY env observation), custom providers/models/commands, Context Sources | npm packages | Moderately — plugin hook API; core internals not directly exposed |
| **dsh** | Cordis plugins mounted at boot; profiles/patch layers; HMR; **the agent can self-modify its plugin graph** | Bundles/profiles, npm | Yes — maximally: swap the agent loop, sandbox, persistence; agent does it at runtime |

Distinctive stance each way:
- **pi**: extensibility is the *product*; core stays a substrate.
- **omp**: built-ins are just privileged extensions; parity between user code and shipped code.
- **opencode**: extension points are documented and versioned against the generated API contract.
- **dsh**: configuration *is* code — boot-time composition trees, plus the unusual ability for the agent to reconfigure itself mid-session.

---

## 8. Engineering Quality: Testing, Build, Release

All four are unusually disciplined. Standout practices per project:

| | Testing | Build | Release/Distribution |
|---|---|---|---|
| **pi** | Vitest + node:test; isolated temp-env test runner (fake HOME, scrubbed env, askpass stub); faux-provider harness (no real tokens); model-backed evals with JSONL artifacts; tmux-driven TUI testing prescribed | Dependency-chain build order; Bun standalone binaries; offline model-data variant | Lockstep versioning, isolated smoke installs, OIDC trusted publishing, release-marker verification on pi.dev; supply-chain gates (release-age, script allowlist) |
| **omp** | bun test tiered orchestrator (singleton/ui/runtime/native/heavy splits); cargo nextest + pedantic clippy; pytest; contract-test rules (no source-grep tests, no mock.module) | Cargo local / Bazel hermetic shipping (zig cc cross, 8 targets, remote cache); Nix flake | curl installer, PowerShell, npm, Homebrew tap, NixOS/Home-Manager modules, mise; signing/notarization; `--smoke-test` probe in install CI |
| **opencode** | Per-package Bun tests (root-run guarded); http-recorder cassettes; Playwright e2e; style guide forbids mocks/globalThis | Turborepo typecheck/build tasks; always package-scoped `bun typecheck` | SST v4 deploy (Cloudflare+AWS, Stripe, stage-gated); Windows code-signing; multi-channel install (npm/brew/scoop/choco/pacman/nix/curl) |
| **dsh** | **Per-file 100% coverage CI gate**; fast-check property tests; keyless snapshot replay of ACP/headless transcripts; web stress/perf configs; jscpd clone detection; real-API e2e self-skipping | tsc project references + tsdown; separate source/artifact planes; mock LLM dev server | ~30 machine-checked repo gates (doc budgets, link checking, catalog freshness, vendored-mod logs); wine-reproducible Windows gates; GitLab + GitHub CI; Python single-exe carrier on PyPI |

---

## 9. Which One Should You Use?

- **Choose pi** if you want maximum control with minimal machinery: you enjoy writing TypeScript, want your workflow features as your own packages, run untrusted work in containers yourself, and value supply-chain rigor. It's also the best codebase to *read* if you want to understand agent-harness fundamentals.
- **Choose oh-my-pi** if you want the most capable *tool surface* today: real debugging (DAP), LSP-aware edits, parallel subagents in isolated worktrees, desktop/browser automation, deep model routing — and you're fine with a large native component and a fast-moving fork.
- **Choose opencode** if you want a polished product spanning terminal/desktop/IDE/cloud with a mature permission model, team features (share links, GitHub Action, Slack), and the strongest client/server + generated-SDK story for building *on top of* the agent programmatically.
- **Choose deepseek-harness** if you're building a *custom harness* rather than using one: the capability-seam design lets you swap LLM, shell, sandbox, persistence, even the loop; the Web UI is first-class; DeepSeek models get the native path. Caveat: developer preview, no stability promise, no first-party TUI.

They also stack: dsh already depends on pi's provider library; omp tracks pi upstream; and any of them can be driven headlessly from CI. For teams, the pragmatic pairing is often **one product for daily use** (opencode or omp) **plus one hackable substrate** (pi or dsh) for bespoke automations.
