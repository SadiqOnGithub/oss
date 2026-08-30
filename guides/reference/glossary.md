# Glossary

Words these four repos reuse with **different meanings**. Read this when a sentence in [comparison-report.md](../compare/comparison-report.md) or a file guide feels circular.

Alphabetized. “Who uses it” is the home repo; others may borrow the word.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

---

**ACP (Agent Client Protocol)** — JSON-RPC stdio protocol for editors/automation to drive an agent. omp and dsh ship servers; opencode has an `acp` CLI command. Not the same as MCP.

**Admit / prompt admission** — (opencode) Writing a user prompt to `session_input` **before** the model runs. Crash-safe. See `packages/core/src/session/input.ts`.

**Agent loop** — The while-stream-tools-repeat driver. pi/omp: `packages/agent` (`agent-loop.ts`). opencode: session drain + `runner/llm.ts`. dsh: **swappable plugin** `ctx.agentLoop` (`packages/core/agent-loop`).

**Agent (opencode)** — A permissioned persona (`build` full access, `plan` read-only), not the loop itself. Tab switches them.

**AGENTS.md** — Markdown instructions discovered into the system prompt. All four read some flavor of this. Also the **contributor rules** file at each repo root (binding for humans and coding agents working *on* the harness).

**Bundle** — (dsh) Installable layer of Cordis config rows (`dsh-base`, `dsh-web-app`, `dsh-headless`). First pieces of a profile.

**Capability seam** — (dsh) A capability split into Service **Definition** + **Provider(s)** + **Consumer** (often `tool-*`). Depend on the definition so swapping `fs-local` for `fs-e2b` moves bash/PTY/LSP together.

**CBOR** — (pi) Binary encoding for the remote-session protocol (`packages/protocol`). Length-prefixed frames. Not used by omp RPC (JSONL) or dsh SDK (JSON-RPC).

**Compaction** — Shrink conversation context when it gets expensive. pi: `coding-agent/src/core/compaction`. omp: that plus `snapcompact` (vision bitmap frames). opencode: session compaction + Context Epochs. dsh: `packages/compaction` plugins.

**Context Epoch** — (opencode) Immutable, provider-cache-friendly baseline system prompt. Changes become durable mid-conversation system messages instead of mutating the baseline. `packages/core/src/session/context-epoch.ts`.

**Context file** — (omp) Markdown loaded at session start (`AGENTS.md`, etc.) into `<repo-rules>`. Distinct from **sticky rules** (`RULES.md`) and from **skills**.

**Cordis** — (dsh) Plugin kernel: `Context`, services, typed events, waterfalls, reversible `ctx.effect` disposers. Vendored under `vendor/cordis`.

**Custom tool vs extension vs hook vs skill** — (omp, also pi)

| Thing | Model-callable code? | What it is |
|---|---|---|
| Custom tool / `registerTool` | Yes | `execute` + schema |
| Extension | Optional | Lifecycle factory; may register tools/commands/UI |
| Hook | No (intercepts) | omp legacy interceptor loaded through the extension runner |
| Skill | No | Markdown guidance / catalog entry |

**Delivery (steer / queue / follow-up)** — What to do with a new user message while a turn is running. opencode: `SessionInput.Delivery` at admit time. pi RPC: `streamingBehavior: "steer" | "followUp"`.

**Dogfooding dir** — Config the project uses to develop *itself*: `.pi/`, `.omp/`, `.opencode/`, `.agents/`. Not the same as the user home dir (`~/.pi/agent`, `~/.omp/agent`, `~/.config/opencode`, `~/.dsh`).

**Drain / Session Drain** — (opencode) Process-local serialized worker that turns admitted inputs into model turns. One drain per session in a process.

**Embedded OpenCode** — (opencode) `packages/sdk-next`: Client + Core + Server in one process, no HTTP. Same HttpApi, in-memory transport.

**Erasable TypeScript** — (pi) Syntax Node can run with `--experimental-strip-types` only: no `enum`, `namespace`, parameter properties. Enforced by `scripts/check-ts-relative-imports.mjs`.

**Extension (pi/omp)** — Default-export factory `(pi: ExtensionAPI) => void`. Registers tools, commands, event handlers. Not an opencode **plugin** and not a dsh **plugin**, though the idea rhymes.

**Hashline** — (omp) Line-anchored patch language behind the edit tool (`packages/hashline`).

**HttpApi** — (opencode) Authoritative Effect HTTP API on the server. Codegen produces `packages/client`. Do not edit generated clients.

**JSONL / JSON-RPC / NDJSON** — Line-oriented JSON. pi/omp RPC: JSONL commands on stdin. dsh SDK / ACP: JSON-RPC (request id + method). omp protocol v2 can chunk oversized frames (`rpc_chunk`).

**Landlock** — (dsh) Linux sandbox: process self-restricts then execs. `native/landlock-run`. One backend of `packages/sandbox`.

**MCP (Model Context Protocol)** — External tool servers. pi: **not in core** (skills instead). omp/opencode/dsh: first-class clients.

**Mnemopi** — (omp) Local SQLite memory engine (`packages/mnemopi`): retain / recall / reflect / learn tools.

**Patch (Cordis)** — (dsh) YAML overlay that inserts/replaces plugin rows by `id`. `--patch`, profile `cordis.patch.yml`, `$DSH_HOME/cordis.patch.yml`.

**patches/** — (npm) Local diffs applied to third-party packages on install (`patchedDependencies`). opencode has many; omp two; dsh `node-pty`; pi none (exact pins instead). Unrelated to Cordis patches.

**Plan mode** — Read-only / design-first agent. opencode: built-in `plan` agent. omp: `src/plan-mode/`. dsh: `packages/plan/plan-mode`. pi: **not in core** (example extension `examples/extensions/plan-mode/`).

**Plugin (opencode)** — `(input: PluginInput) => Hooks` from `@opencode-ai/plugin`. Can add tools, intercept, observe PTY. Listed in `opencode.json`.

**Plugin (dsh)** — Cordis module with `name`, optional `inject`, `apply(ctx)`. Everything is one of these, including the loop.

**Profile (dsh)** — Named composition under `$DSH_HOME/profiles/<name>`: ordered bundles + user patch. `web` and `headless` auto-init.

**Profile (omp)** — Named agent dir `~/.omp/profiles/<name>/agent`. Relocates settings/auth, not the same as dsh profiles.

**Provider** — Ambiguous. **Model provider** = Anthropic/OpenAI/…. **Discovery provider** (omp) = adapter that reads `.claude` / `.cursor` / `.omp` files. **Capability provider** (dsh) = implementation of a seam (`bash-local`).

**RPC mode** — Agent as a stdio server. `pi --mode rpc`, `omp --mode rpc`. Headless, JSONL. Different from dsh JSON-RPC SDK and from opencode HTTP.

**Session** — Durable conversation. pi CLI: JSONL under `~/.pi/agent/sessions/` (`session-manager.ts`); `sqlite-node` is a separate library for `pi-agent-core`, not the CLI store. omp: JSONL session tree + mnemopi SQLite **memories**. opencode: SQLite + V2 input table. dsh: append-only **session event log** (JSONL and/or SQLite).

**Skill** — Markdown (often `SKILL.md`) the agent may load. pi: the blessed MCP alternative. Others: catalog + loader tools (`manage_skill`, `tool-skill`, opencode `skill` tool).

**Snapcompact** — (omp) Bitmap-frame context compression for vision models (`packages/snapcompact`).

**Spill** — Drop oversized tool output to a file/store and keep a preview in the model context. opencode: `tool-output-store.ts`. dsh: `packages/spill` (explicit seam). pi: truncation helpers in tools.

**Subagent** — Child agent. pi: **not in core** (example extension). omp: `task` tool + `crates/pi-iso` worktrees. opencode: `@general` / `task` tool. dsh: whole `packages/subagent` family (in-process, ACP, Claude Code, Codex, dsh-sdk).

**System Context (opencode)** — Typed sources (AGENTS.md, date, skills, …) rendered into the epoch baseline (`packages/core/src/system-context`).

**Tool** — Model-callable function with a JSON schema. Implementation site differs (table in [add-a-tool.md](../how-to/add-a-tool.md)).

**Typert** — (dsh) Type-graph generator + runtime registry for remote/BFF contracts (`packages/typert`). Spelling is `Typert`, not TypeRT.

**Waterfall** — (Cordis) Event that listeners must `next()`. `agent/pre-step` can rewrite or reject the turn. Not a Node EventEmitter.

**Worktree / iso** — Isolated copy of the repo for a child task. omp: `crates/pi-iso` (APFS clone, overlayfs, …). opencode: `packages/opencode/src/worktree`. dsh subagents may spawn/fork without that Rust layer.

**xd://** — (omp) Discoverable-tool namespace. `read`/`write` stay essential so devices remain reachable; other builtins can hide behind xdev.
