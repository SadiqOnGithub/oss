# DeepSeek Harness (dsh) — file, folder, and functionality guide

**Repo:** [`deepseek-harness/`](../../deepseek-harness) (`deepseek-ai/deepseek-harness`)
**Binary:** `dsh` (`@deepseek-ai/dsh`)
**Status:** developer preview — breaking changes expected
**Companions:** [comparison-report.md](../compare/comparison-report.md) · [structure-comparison.md](../compare/structure-comparison.md) · [repo-guides.md](../README.md)
**Checkout verified:** 2026-08-29 · `@deepseek-ai/dsh-root` **0.1.1-rc.2** · **50** package groups, **23** `tool-*` packages

DeepSeek Harness is a **plugin-kernel agent harness**: the model adapter, tools, session log, sandbox, and **the agent loop itself** are Cordis plugins composed at boot. There is no privileged core to patch — you mount another plugin. Not a fork of pi, but `packages/llm/llm-pi-ai` optionally uses `@earendil-works/pi-ai` for multi-provider adapters.

Who it is for: people building on a composable kernel (Web UI, headless, ACP, TS/Python SDKs) rather than a single TUI binary.

Start here in-repo: [docs/architecture.md](../../deepseek-harness/docs/architecture.md), [docs/capability-seams.md](../../deepseek-harness/docs/capability-seams.md), [packages/README.md](../../deepseek-harness/packages/README.md).

---

## 1. How it works

```
dsh CLI                 apps/cli/src/bin.ts
  parse args            apps/cli/src/args.ts
  load profile          apps/cli/src/profile-boot.ts
       │
       │  composition (empty tree → patches):
       │    bundle(s) in dsh.profile.bundles order
       │      dsh-base           always first (tools, llm, session, sandbox, …)
       │      dsh-web-app  or  dsh-headless
       │    profile cordis.patch.yml
       │    $DSH_HOME/cordis.patch.yml
       │    --patch overlays
       │
       ▼
  Cordis Loader         vendor/cordis + vendor/loader
       │
       ├─ ctx.sessions / ctx.agentLoop / ctx.tools / ctx.llm / ctx.fs / …
       ├─ Web: host HTTP + apps/web frontend at 127.0.0.1:3080
       ├─ Headless: one-shot session, print answer, exit
       ├─ ACP / JSON-RPC SDK as other profiles/plugins
       └─ Python SDK spawns the same runtime over stdio JSON-RPC
```

**A prompt's path** (from `docs/architecture.md`)

```
turn/start
  claim next-step input
  assemble prompt + tool schemas     core/system-prompt
  → agent/pre-step                   waterfall: rewrite or reject
     step/start
     agent/request → llm/stream      packages/llm
     tool/call → tools/pre-execute → execute → post-execute
     step/end
  → agent/turn-stopping
```

Facts that must survive reload are **session events** appended to the log (`model-visible ⟺ logged`). Oversized tool text **spills**; compaction prunes; `session-query` searches.

**Entry modes** (`apps/cli`)

| Command | Purpose |
|---|---|
| `dsh web` | Alias of `--profile web` — Web UI |
| `dsh --profile headless "job"` | One-shot persisted session |
| `dsh --profile <name>` | Named composition under `$DSH_HOME/profiles/` |
| `dsh plugin --profile <name> …` | Forward to pnpm in that profile dir |
| `--dump-config` / `--dump-default-config` | Print the composed plugin tree without booting |

**Capability seam pattern:** Service **Definition** + **Providers** + **Consumers** (often a `tool-*`). Extensions depend on definitions, never concrete providers — swapping `fs-local` for `fs-e2b` relocates bash/PTY/LSP together.

---

## 2. Root files

| File | Purpose |
|---|---|
| `package.json` | pnpm 11 workspaces: `vendor/*`, `packages/*/*`, `native/landlock-run`, `apps/*`, `website`. Gate scripts: `check:all`, `check:ci*` |
| `pnpm-workspace.yaml` / `pnpm-lock.yaml` | Workspace + lockfile |
| `tsconfig.json` / `tsconfig.base.json` / `tsconfig.host.json` / `tsconfig.client.json` / `tsconfig.base.client.json` | Host vs browser type planes |
| `tsdown.config.ts` | Bundling (`DSH_BUILD_FACE` host\|client) |
| `vitest.config.ts` plus `vitest.e2e`, `vitest.snapshot`, `vitest.web`, `vitest.web.perf`, `vitest.web-stress`, `vitest.shared.ts` | Test planes |
| `pytest.ini` | Python tests |
| `knip.json` / `lefthook.yml` / `.jscpd.json` | Unused exports, git hooks, duplication |
| `.oxlintrc.json` / `.oxlintrc.staged.json` | Lint |
| `AGENTS.md` / `CLAUDE.md` | Agent rules (nested copies also live under `packages/` and `vendor/`) |
| `README.md` + `README.zh.md` + `README.i18n.yaml` | **Every doc is a triple:** English, Chinese, i18n metadata |
| `CONTRIBUTING.*` / `BRAND_GUIDELINES.*` / `BENCHMARK.md` / `THIRD_PARTY_NOTICES.md` / `LICENSE` | Process, brand, bench, licenses (MIT) |
| `.gitlab-ci.yml` | GitLab mirror of gates |
| `.editorconfig` / `.rgignore` | Editor / ripgrep |

---

## 3. Root directories

| Dir | Purpose |
|---|---|
| `apps/cli` | `dsh` launcher (`@deepseek-ai/dsh`) |
| `apps/web` | Vite React frontend (`@deepseek-ai/dsh-web-frontend`); `dist/` served by the web profile |
| `packages/` | **50** capability groups, each holding one or more `@deepseek-ai/dsh-*` packages (see §4) |
| `python/sdk` | `deepseek-harness-sdk` — Python JSON-RPC client |
| `python/sdk-runtime` | PyPI **runtime binary carrier** (`hatch_build.py`, `platforms.json`) |
| `native/landlock-run` | C++ N-API Landlock self-restrict-then-exec launcher (Linux sandbox) |
| `vendor/` | Source-vendored Cordis family, SHA-pinned, rescoped to `@deepseek-ai/*` |
| `docs/` | Architecture, catalogs, cookbooks, user guide — all trilingual |
| `website/` | VitePress projection of docs |
| `examples/` | Extra examples at repo root (packages also has `examples/`) |
| `scripts/` | ~168 gate/generator scripts (`run-gates.ts`, catalogs, i18n, vendor checks) |
| `patches/` | `node-pty@1.2.0-beta.15.patch` |
| `.agents/` | Dogfooding: `skills/` + `notes/` (proposed / implemented / rejected / archived) |
| `.claude/` | Extra Claude-oriented instructions |
| `.github/workflows/` | `ci.yml`, `ci-master.yml`, e2e, sandbox, landlock, python-release, vendor-publish, docs-pages, … |

### `vendor/` (owned framework)

| Path | Role |
|---|---|
| `cordis` | Plugin kernel: context, services, events, waterfalls, disposers |
| `loader` | Config/plugin loader |
| `cosmokit` / `schemastery` / `group` / `hmr` / `include` / `timer` / `logger-console` | Cordis ecosystem, vendored |
| `AGENTS.md` / `CLAUDE.md` / modification logs | Why each upstream delta exists |

### `docs/` (read these, ignore i18n triples)

Architecture: `architecture.md`, `capability-seams.md`, `cordis-primer.md`, `cordis-tutorial/`, `tool-execution-pipeline.md`, `agent-lifecycle.md`, `event-producer-consumer.md`, `defensive-patterns.md`.

Generated catalogs (CI freshness-gated): `tool-catalog.md`, `config-catalog.md`, `persistence-catalog.md`, `module-graph.md`, `graph-atlas.md`.

Also: `development.md`, `testing.md`, `glossary.md`, `user/`, `cookbook/`, `subsystems/`, `postmortem/`, `api-gateway.md`.

---

## 4. Apps

### `apps/cli` — `@deepseek-ai/dsh`

| File | Role |
|---|---|
| `src/bin.ts` | Process entry (`bin.dsh`). Dynamic-import per mode |
| `src/args.ts` | Command grammar |
| `src/profile-boot.ts` | Compose bundles + patches, run Loader |
| `src/plugin.ts` | `dsh plugin` → pnpm in the profile dir |
| `src/dump-config.ts` | `--dump-config` |
| `src/process-shutdown.ts` | Exit / signal handling |
| `config/` | Shipped profile templates (`web`, `headless`) |
| `reference/` | Exact flag/layer/shutdown behavior |

### `apps/web` — `@deepseek-ai/dsh-web-frontend`

Vite app over `@deepseek-ai/dsh-client-web`. `src/main.ts` is the browser entry. Host serves `dist/`.

---

## 5. Package groups (`packages/<group>/<pkg>/`)

npm names are `@deepseek-ai/dsh-<pkg>` (the **pkg** directory, not the group). Group READMEs own the ctx-key map.

### Spine / boot

| Group | Role | Nested packages |
|---|---|---|
| `core/` | Product API: sessions, prompts, tools, agents, **swappable loop** | `session`, `system-prompt`, `tools`, `agent`, `agent-loop`, `agent-default-model`, `agent-tool-presentation`, `scope` |
| `boot/` | App-bin glue: `.env`, Loader, snapshot-aware config | `app-boot`, `cmdline` |
| `bundle/` | Installable `--profile` layers | `base` (always first), `web-app`, `headless` |
| `preset/` | Per-session composition from `cordis.yml` | `agent-presets`, `persona` |
| `api/` | Remote BFF + Typert RPC gateway | `gateway`, `remotes` |
| `typert/` | Type-graph generator + runtime registry | `generator`, `loader`, `protocol`, `registry` |

`ctx` keys (core): `ctx.sessions`, `ctx.systemPrompt`, `ctx.tools`, `ctx.agents`, `ctx.agentLoop`.

### Model

| Group | Role | Nested packages |
|---|---|---|
| `llm/` | LLM seam + adapters + retry + metering | `llm`, `llm-deepseek`, `llm-pi-ai`, `llm-retry`, `token-meter` |
| `credentials/` | Credential records + local env provider + authz | `credentials`, `credentials-local`, `authorization` |
| `identity/` | Anonymous user id | `anonymous-user-id` |

### Execution (tools + OS)

| Group | Role | Nested packages |
|---|---|---|
| `fs/` | Filesystem seam, sandbox fencing, file tools | `fs`, `fs-local`, `fs-sandbox`, `fs-observation-policy`, `tool-fs`, `tool-fs-search`, `tool-str-replace-editor` |
| `shell/` | Bash/pwsh executors + tools | `shell`, `shell-env`, `bash-local`, `bash-sandbox`, `pwsh-local`, `pwsh-sandbox`, `tool-bash`, `tool-bash-persistent`, `tool-pwsh`, `tool-pwsh-persistent` |
| `subprocess/` | Process groups, spill-backed output | `subprocess`, `subprocess-local` |
| `terminal/` | Persistent PTY | `terminal`, `terminal-bash`, `tool-terminal` |
| `code-runtime/` | Code Mode (worker thread / Python) | `code-runtime`, `code-runtime-worker-thread`, `code-runtime-python` |
| `sandbox/` | bwrap / Landlock / Seatbelt / Windows ACL | `sandbox`, `sandbox-local`, `sandbox-policy`, `sandbox-windows-acl` |
| `lsp/` | LSP seam + stdio provider + `lsp` tool | `lsp`, `lsp-stdio`, `tool-lsp` |
| `web/` | Search/fetch | `web`, `tool-web`, `web-fetch-http`, `web-search-deepseek`, `web-search-exa`, `web-search-perplexity` |
| `e2b/` | E2B cloud sandbox (POC) | `e2b`, `fs-e2b`, `subprocess-e2b` |

### Orchestration

| Group | Role | Nested packages |
|---|---|---|
| `subagent/` | Child agents: in-process fork/spawn, ACP, Claude Code, Codex, dsh-sdk | `subagent`, `subagent-in-process-driver`, `subagent-fork-in-process`, `subagent-spawn-in-process`, `subagent-acp`, `subagent-claude-code`, `subagent-codex`, `subagent-dsh-sdk`, `tool-subagent`, `tool-subagent-control`, `tool-subagent-report` |
| `jobs/` | Background jobs | `jobs`, `jobs-local`, `tool-jobs` |
| `workflow/` | Model-written JS orchestration + Ralph loop | `workflow`, `workflow-worker-thread`, `tool-workflow`, `tool-ralph` |
| `todo/` | `todo_write` | `tool-todo` |
| `plan/` | Plan mode | `plan-mode` |
| `goal/` | Same-session goals | `goal`, `goal-round-driver`, `command-goal`, `tool-goal` |
| `schedule/` | Session-local reminders | `schedule` |
| `guard/` | Repeat-tool reminders + tool deadlines | `repeat-tool-reminder`, `timeout-policy` |
| `skill/` | Skills catalog/loader | `skill`, `skill-badge`, `skill-filesystem`, `tool-skill` |
| `mcp/` | MCP client | `mcp-client` |
| `extensions/` | Agent inspects/mounts/unmounts its own plugins live | `tool-cordis`, `cordis-host-runner`, `cordis-client-runner`, `ui-cordis` |
| `hooks/` | Claude Code / Codex hook bridges | `hook-protocol`, `hooks-claude-code`, `hooks-codex` |
| `experimental/` | Unreleased prototypes | `agent-team`, `tool-agent-team` |

### Data plane

| Group | Role | Nested packages |
|---|---|---|
| `session/` | Persistence (JSONL + SQLite), projections, titles, OTel | `session-persistence`, `session-persistence-jsonl`, `session-persistence-sqlite`, `session-projection`, `session-projection-cache`, `session-checkpoint-policy`, `session-stats`, `session-telemetry`, `session-telemetry-otel`, `session-title`, `session-title-llm`, `session-title-first-prompt-llm`, `session-title-all-prompts-llm` |
| `session-query/` | Bounded reads, lineage, FTS, semantic filter | `session-query`, `session-query-sqlite`, `session-log-export`, `tool-session-query` |
| `attachment/` | Content-addressed blobs | `attachment`, `attachment-local` |
| `spill/` | Oversized tool-text store | `spill`, `spill-local`, `spill-policy` |
| `storage/` | Non-session KV hub | `storage`, `storage-domain`, `storage-json`, `storage-sqlite` |
| `workspace/` | Workspace registry | `workspace` |
| `compaction/` | Context compression | `compaction`, `compaction-basic`, `compaction-tool-result-pruner`, `command-compact` |
| `context/` | Request context (AGENTS.md, time, tmux, file refs) | `agent-instructions`, `time-context`, `tmux-context`, `file-reference`, `file-reference-local`, `session-reference` |
| `settings/` | User settings | `settings`, `settings-file` |

### Delivery (Web GUI + SDKs)

| Group | Role | Nested packages |
|---|---|---|
| `host/` | Node half of the GUI: HTTP, static frontend, API proxy | `webserver`, `frontend-static`, `apiproxy`, `plugin-inventory`, `directory-picker*` |
| `client/` | Browser half: runtime, connection, **~30 `ui-*` slot plugins**, locale, HMR | `web`, `runtime`, `connection`, `modules`, `locale`, `hmr`, `ui-layout`, `ui-conversation`, `ui-sidebar`, `ui-tool`, `ui-settings*`, `ui-plan`, `ui-skill`, `ui-subagent`, `ui-workflow-run`, … |
| `sdk/` | Out-of-process JSON-RPC | `protocol`, `client`, `server` |
| `acp/` | Agent Client Protocol server (stdio JSON-RPC) | `acp` |
| `interaction/` | Approvals, permission presets, ask-user, commands | `user-approval`, `user-questions`, `permission-presets`, `commands`, `tool-ask-user` |
| `feedback/` | Recorded human feedback | `message-feedback`, `command-feedback` |

### Support

| Group | Role | Nested packages |
|---|---|---|
| `util/` | Zero-dep helpers | `atomic-write`, `brand`, `home-paths`, `launch-environment`, `native-command`, `output-retention`, `timeout` |
| `test-support/` | Testkits, LLM replay/mock, loader smokes | `agent-loop-testkit`, `llm-replay`, `llm-mock-server`, `acp-snapshot`, `client-runtime`, `loader-smoke` |
| `runtime-diagnostics/` | Invariant checks | `invariants` |
| `examples/` | Demo bundles | `agent-spine-demo`, `acp-demo`, `jsonrpc-demo` |

Also at `packages/`: `README.md`, `AGENTS.md`, `CLAUDE.md` (group-level rules).

---

## 6. Python, native, website

| Path | What it does |
|---|---|
| `python/sdk` | Official Python SDK (`deepseek_harness.DeepSeekHarness`) over newline-delimited JSON-RPC on stdio |
| `python/sdk-runtime` | Platform wheel that **is** the bundled `dsh` runtime the SDK launches |
| `python/*.md` | Bilingual development docs for the Python tree |
| `native/landlock-run` | Landlock launcher used by Linux sandbox backends |
| `website/` | VitePress docs site (`docs:dev` / `docs:check`) |

---

## 7. Visual map

```
deepseek-harness/
├── AGENTS.md  CLAUDE.md  README{,.zh}.md  README.i18n.yaml
├── package.json  pnpm-workspace.yaml  tsconfig*.json  vitest*.ts
├── patches/node-pty@1.2.0-beta.15.patch
├── scripts/                 # ~168 gates + catalog generators
├── docs/                    # architecture, catalogs, cookbooks (en/zh/i18n)
├── website/                 # VitePress
├── .agents/skills + notes   # dogfooding + decision records
├── apps/
│   ├── cli                  # `dsh` binary
│   └── web                  # Vite frontend
├── packages/                # capability groups (not layers)
│   ├── core  boot  bundle  preset  api  typert
│   ├── llm  credentials  identity
│   ├── fs  shell  subprocess  terminal  code-runtime  sandbox  lsp  web  e2b
│   ├── subagent  jobs  workflow  todo  plan  goal  schedule  guard  skill  mcp
│   ├── session  session-query  attachment  spill  storage  workspace  compaction  context  settings
│   ├── host  client  sdk  acp  interaction  feedback  extensions  hooks
│   └── util  test-support  examples  experimental
├── python/{sdk,sdk-runtime}
├── native/landlock-run
└── vendor/{cordis,loader,cosmokit,schemastery,…}
```

---

## 8. Where do I find X?

| Looking for | Path |
|---|---|
| CLI entry | `apps/cli/src/bin.ts` |
| Profile composition | `apps/cli/src/profile-boot.ts`, `packages/boot/app-boot/`, `packages/bundle/` |
| Agent loop | `packages/core/agent-loop/` (`ctx.agentLoop`, swappable) |
| Session log | `packages/core/session/` + `packages/session/` |
| Tools registry | `packages/core/tools/` |
| File / bash / PTY tools | `packages/fs/`, `packages/shell/`, `packages/terminal/` |
| LLM adapters | `packages/llm/` (`llm-deepseek`, `llm-pi-ai`) |
| MCP / LSP / ACP | `packages/mcp/`, `packages/lsp/`, `packages/acp/` |
| Sandbox | `packages/sandbox/` + `native/landlock-run/` |
| Web UI host / client | `packages/host/`, `packages/client/`, `apps/web/` |
| Python SDK | `python/sdk/` |
| Vendored kernel | `vendor/cordis/` |
| Architecture prose | `docs/architecture.md`, `docs/capability-seams.md` |
| Tool catalog | `docs/tool-catalog.md` (generated) |
| Tests | root `pnpm test` (vitest) + e2e/snapshot/web planes |
| CI | `.github/workflows/ci.yml` + `.gitlab-ci.yml` |
| Self-hosting | `.agents/` (skills + notes). Runtime profiles live in `$DSH_HOME` |

---

## 9. Conventions

- **Everything is a plugin.** Depend on Service Definitions, never on a concrete provider. The agent loop is not sacred.
- **Trilingual docs:** every user-facing markdown has `.md` + `.zh.md` + `.i18n.yaml`. CI checks wrap, links, and catalogs.
- **Two type planes:** host (Node) vs client (browser). `tsdown` builds both (`DSH_BUILD_FACE`).
- **Vendor, don't patch Cordis in node_modules:** changes go through `vendor/` with a modification log.
- **Generated catalogs** (`tool-catalog`, `module-graph`, …) are freshness-gated — don't hand-wave them stale.
- **`.agents/notes`:** proposed → implemented / rejected / archived. Skills under `.agents/skills/` are how the repo develops itself.
- npm scope is always `@deepseek-ai/dsh-*`. New packages join an existing **group**; new groups update `packages/README.md`.
