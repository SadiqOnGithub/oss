# pi — file, folder, and functionality guide

**Repo:** [`pi/`](../../pi) (`badlogic/pi-mono`, [pi.dev](https://pi.dev))
**Binary:** `pi` (`@earendil-works/pi-coding-agent`)
**Companions:** [comparison-report.md](../compare/comparison-report.md) · [structure-comparison.md](../compare/structure-comparison.md) · [repo-guides.md](../README.md)
**Checkout verified:** 2026-08-29 · `@earendil-works/pi-coding-agent` **0.84.2**

pi is a **minimal, self-extensible coding-agent harness**. The kernel ships four built-in tools (`read`, `bash`, `edit`, `write`), no MCP, no subagents, no permission popups, and no plan mode. Workflow features belong in TypeScript **extensions**, **skills**, **prompt templates**, and **themes** distributed as Pi Packages over npm/git.

Who it is for: people who want a small substrate they can own, not a batteries-included IDE-in-the-terminal.

---

## 1. How it works

```
pi (CLI)
  packages/coding-agent/src/cli.ts  →  main.ts
       │
       ├─ parse args, load settings, pick session
       ├─ createAgentSession()      packages/coding-agent/src/core/
       │     uses pi-agent-core     packages/agent/src/agent-loop.ts
       │     calls pi-ai            packages/ai/src/providers/
       │
       ├─ TUI mode                  packages/tui + modes/interactive
       ├─ print / JSON mode         modes/print-mode.ts, json-event.ts
       ├─ RPC mode                  modes/rpc + rpc-entry.ts
       └─ SDK embed                 src/index.ts  (createAgentSession)
```

**A prompt's path**

1. User input (TUI editor, `-p` flag, RPC request, or SDK call).
2. `AgentSession` appends a user message and wakes `pi-agent-core`.
3. The agent loop streams from a provider in `pi-ai` (OpenAI, Anthropic, Google, Bedrock, Azure, Groq, xAI, OpenRouter, Mistral, …).
4. Tool calls run in `packages/coding-agent/src/core/tools/` (read / bash / edit / write). `find`, `grep`, and `ls` exist as extra factories on the SDK; the CLI harness mounts the four coding tools (`createCodingTools` in `core/sdk.ts`).
5. Session state is **JSONL files** under `~/.pi/agent/sessions/<encoded-cwd>/` (`core/session-manager.ts`). Branching and compaction live there. `packages/session-backends/sqlite-node` is a **separate** `pi-agent-core` backend library — the `pi` CLI does not import it.
6. Events stream to the TUI or RPC client. RPC is snapshot-authoritative: progress events are UI hints, not state.

**Modes** (from `packages/coding-agent/src/modes/`)

| Mode | What it is |
|---|---|
| Interactive TUI | Alternate-screen app on `pi-tui` |
| Print | One-shot prompt, stdout |
| JSON events | Machine-readable event stream |
| RPC | NDJSON/process integration (`rpc-entry.ts`) |
| SDK | `createAgentSession()` from `@earendil-works/pi-coding-agent` |

**Extension points** — extensions (TS hooks), skills (markdown, positioned as the MCP replacement), prompt templates, themes, custom providers. No built-in permission system: sandboxing is documented as Gondolin micro-VM, Docker, or OpenShell.

**Remote sessions** — `protocol` (CBOR + 4-byte length prefix) → `server` (listeners) → `client` (leases). Transport-neutral; auth happens before protocol bytes flow.

---

## 2. Root files

| File | Purpose |
|---|---|
| `package.json` | npm workspaces; **build order** is encoded in the `build` script: tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent |
| `package-lock.json` | Ground-truth lockfile; changes gated (`PI_ALLOW_LOCKFILE_CHANGE=1`) |
| `.npmrc` | Supply-chain settings (exact pins, min release age) |
| `tsconfig.json` / `tsconfig.base.json` | Strict, **erasable-only** TS (Node strip-only: no enums/namespaces) |
| `vitest.base.ts` | Shared Vitest config |
| `biome.json` | Lint/format (`--error-on-warnings`) |
| `AGENTS.md` | Rules for humans and agents: tmux TUI testing, regression placement, no core bloat |
| `README.md` | Philosophy and package table |
| `CONTRIBUTING.md` | Process; core-bloating PRs rejected |
| `SECURITY.md` | Vulnerability reporting |
| `LICENSE` | MIT |
| `.gitignore` / `.gitattributes` | Ignores and line-ending rules |
| `test.sh` | Non-e2e tests in an isolated temp HOME so real credentials are never touched |
| `pi-test.sh` / `pi-test.bat` / `pi-test.ps1` | Run the CLI from source via tsx; `--no-env` unsets provider credentials |
| `tui-plan.md` | Working plan for TUI alt-screen layout (VStack/HStack/ScrollView) |

---

## 3. Root directories

### `packages/` — the whole architecture (see §4)

### `scripts/` — ~35 automation scripts

**Supply-chain / hygiene**

| Script | Purpose |
|---|---|
| `check-pinned-deps.mjs` | External deps must be exact versions |
| `check-lockfile-commit.mjs` | Lockfile diffs must be intentional |
| `check-ts-relative-imports.mjs` | Erasable-TS import rules (explicit `.ts` extensions) |
| `check-browser-smoke.mjs` + `browser-smoke-entry.ts` | Code must also run outside Node |
| `generate-coding-agent-shrinkwrap.mjs` | Published shrinkwrap |
| `generate-coding-agent-install-lock.mjs` | Lifecycle-script allowlist for hardened installs |

**Build / release**

| Script | Purpose |
|---|---|
| `build-binaries.sh` | Standalone Bun binaries per platform |
| `build-coding-agent-bundle.mjs` | esbuild bundle for the CLI |
| `agent-treeshake-smoke-entry.ts` | Dead-code-elimination smoke |
| `release.mjs` / `local-release.mjs` / `release-packages.mjs` | Version, changelog, tag, publish |
| `release-notes.mjs` | Notes from changelogs |
| `publish.mjs` / `publish-release-announcement.mjs` | npm publish + pi.dev announcement marker |
| `publish-model-catalog.mjs` | Generated model catalog |
| `sync-versions.js` | Lockstep versions across packages |
| `create-source-archive.sh` | Release source tarball |

**Diagnostics**

| Script | Purpose |
|---|---|
| `cost.ts`, `stats.ts`, `tool-stats.ts`, `read-tool-stats.mjs`, `edit-tool-stats.mjs`, `session-context-stats.mjs` | Session cost / tool-usage analysis |
| `session-transcripts.ts` | Transcript export |
| `diff-model-catalog.mjs` | Diff generated catalog vs live providers |
| `generate-thinking-capabilities.mjs` | Per-model thinking/reasoning data |
| `profile-coding-agent-node.mjs` | CPU-profile TUI or RPC |
| `repro-5893-wsl-bash.mjs` | WSL bash repro |
| `auto-pi.sh` | Convenience launcher |
| `update-source-imports-to-ts.sh` | Codemod to `.ts` imports |

### `.pi/` — pi developing itself

| Path | Contents |
|---|---|
| `extensions/` | `import-repro.ts`, `prompt-url-widget.ts`, `redraws.ts`, `tps.ts` |
| `prompts/` | `pr.md`, `sa.md`, `cl.md`, `is.md`, `wr.md` |
| `skills/add-llm-provider.md` | How to add a provider to `pi-ai` |
| `git/`, `npm/` | Gitignored local state |

### `.github/`

| Path | Purpose |
|---|---|
| `workflows/ci.yml` | Build order, checks, tests |
| `workflows/pr-gate.yml` | PR gates |
| `workflows/build-binaries.yml` | Per-platform binaries |
| `workflows/npm-audit.yml` | Scheduled audits |
| `workflows/approve-contributor.yml` + `APPROVED_CONTRIBUTORS` | Contributor allowlist |
| `workflows/issue-*.yml`, `remove-inprogress-on-close.yml` | Issue triage bots |
| `workflows/publish-model-catalog.yml` | Catalog refresh |

### `.husky/pre-commit`

Runs Biome, pinned-deps, lockfile, ts-relative-imports, shrinkwrap regen.

No `patches/` folder — pi pins exact versions instead of carrying local npm diffs.

---

## 4. Packages (the architecture)

Every package shares:

```
<pkg>/package.json  README.md  CHANGELOG.md  src/  test/  vitest.config.ts  tsconfig.build.json
```

Lockstep version. Build order: **tui → telemetry → ai → agent → sqlite-node → protocol → client → server → coding-agent**. `evals` is separate.

### Core stack

| Dir | npm name | What it does |
|---|---|---|
| `packages/tui` | `@earendil-works/pi-tui` | Differential renderer, CSI 2026 synchronized output, main/alt screens, Editor/Markdown/SelectList, Kitty/iTerm2 images, bracketed paste. `src/components/`, `native/` |
| `packages/telemetry` | `@earendil-works/pi-telemetry` | Vendor-neutral span/context contracts, no-op + in-memory adapters. No exporter baked in |
| `packages/ai` | `@earendil-works/pi-ai` | Multi-provider LLM API. `src/providers/`, `src/auth/`, `src/api/`, generated `models.generated.ts` / `image-models.generated.ts`. Do not hand-edit generated catalogs — regenerate from `packages/ai/scripts/` |
| `packages/agent` | `@earendil-works/pi-agent-core` | Stateful agent runtime: `agent-loop.ts`, tool execution, attachments, pluggable stream fn and session backends. `src/harness/`, `src/search/` |
| `packages/session-backends/sqlite-node` | `@earendil-works/pi-session-backend-sqlite-node` | Optional `node:sqlite` backend **for `pi-agent-core` consumers** (repository, migrations, FTS). Not used by the `pi` CLI, which writes JSONL via `session-manager.ts` |

### Remote-session split

| Dir | npm name | What it does |
|---|---|---|
| `packages/protocol` | `@earendil-works/pi-protocol` | CBOR schemas + 4-byte big-endian length-prefix framing. No runtime imports |
| `packages/client` | `@earendil-works/pi-client` | `PiClient` over any ordered byte transport (WebSocket, Unix socket). Exclusive/shared leases, snapshot-authoritative state |
| `packages/server` | `@earendil-works/pi-server` | Experimental session server; apps supply `PiServerService` |

### Product

### `packages/coding-agent` — `@earendil-works/pi-coding-agent` — the `pi` CLI

| Path | Role |
|---|---|
| `src/cli.ts` | Process entry (`bin.pi` → `dist/bundle/cli.js`) |
| `src/main.ts` | Arg parse → `createAgentSession()` |
| `src/cli/` | Args, auth, session picker, first-run UI, project trust |
| `src/core/` | Session runtime, settings, skills, slash commands, compaction, system prompt |
| `src/core/tools/` | `read`, `bash`, `edit`, `write` (+ SDK factories for find/grep/ls) |
| `src/modes/` | `interactive/` (TUI), `print-mode.ts`, `json-event.ts`, `rpc/` |
| `src/extensions/` | Extension loader |
| `src/index.ts` | Public SDK |
| `src/rpc-entry.ts` | RPC process entry |
| `src/bun/` | Bun compile entry for standalone binaries |
| `docs/` | User docs including RPC and [containerization](../../pi/packages/coding-agent/docs/containerization.md) |
| `examples/extensions/` | Workspace examples: `with-deps`, `custom-provider-anthropic`, `custom-provider-gitlab-duo`, `sandbox`, `gondolin` |
| `npm-shrinkwrap.json` | Pinned tree shipped to npm consumers |
| `install-lock/` | Allowed lifecycle scripts |

### `packages/evals` — `@earendil-works/pi-evals`

Model-backed behavioral evals wrapping a real `AgentSession` (`vitest-evals`). Isolated temp dirs, JSONL artifacts. Needs real provider auth. `npm run eval` only — not part of `./test.sh`.

---

## 5. Visual map

```
pi/
├── AGENTS.md  README.md  CONTRIBUTING.md  SECURITY.md  LICENSE
├── package.json  package-lock.json  .npmrc
├── tsconfig{,.base}.json  biome.json  vitest.base.ts
├── tui-plan.md
├── test.sh                  # isolated-env tests
├── pi-test.{sh,bat,ps1}     # run CLI from source
├── .husky/pre-commit
├── scripts/                 # pins, shrinkwrap, binaries, release
├── .github/workflows/       # ci, pr-gate, binaries, catalog, triage
├── .pi/                     # dogfooding: extensions, prompts, skills
└── packages/
    ├── tui ─────────────────┐
    ├── telemetry            │  core stack (this build order)
    ├── ai                   │
    ├── agent                │
    ├── session-backends/sqlite-node
    ├── protocol ────────────┤  remote session
    ├── client               │
    ├── server               │
    ├── coding-agent         │  ← product (`pi` CLI)
    └── evals                #  paid-token evals
```

---

## 6. Where do I find X?

| Looking for | Path |
|---|---|
| CLI entry | `packages/coding-agent/src/cli.ts` → `main.ts` |
| Agent loop | `packages/agent/src/agent-loop.ts` |
| Built-in tools | `packages/coding-agent/src/core/tools/` |
| Providers / models | `packages/ai/src/providers/`, `models.generated.ts` |
| TUI framework | `packages/tui/src/` |
| Interactive UI | `packages/coding-agent/src/modes/interactive/` |
| Session store (CLI JSONL) | `packages/coding-agent/src/core/session-manager.ts` → `~/.pi/agent/sessions/` |
| SQLite backend (library only) | `packages/session-backends/sqlite-node/` |
| Compaction | `packages/coding-agent/src/core/compaction/` |
| Skills / extensions | `packages/coding-agent/src/core/skills.ts`, `src/extensions/` |
| RPC protocol | `packages/protocol/src/`, `packages/coding-agent/src/modes/rpc/` |
| Tests | per-package `test/`; root `./test.sh`; regressions in `packages/coding-agent/test/suite/regressions/` |
| CI | `.github/workflows/ci.yml`, `pr-gate.yml` |
| Self-hosting config | `.pi/` |
| Sandbox docs | `packages/coding-agent/docs/containerization.md` |

---

## 7. Conventions

- **Erasable TypeScript only** so Node ≥22.19 can run it without transpile. `scripts/check-ts-relative-imports.mjs` enforces this.
- **Exact-pinned** external deps; lockfile is reviewed like code. No `patches/`.
- **Agent-first:** `.pi/` uses the same extension/prompt/skill types users ship. `AGENTS.md` is binding for both humans and agents.
- **Core-bloat policy:** features that could be an extension are rejected from the kernel. `oh-my-pi` is a fork that reversed this policy (see [oh-my-pi-files-guide.md](./oh-my-pi-files-guide.md)).
- Generated model catalogs are regenerated, never hand-edited.
- `oh-my-pi` still uses four of these package names (`ai`, `agent`, `tui`, `coding-agent`).
