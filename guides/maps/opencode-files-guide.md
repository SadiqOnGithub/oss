# OpenCode — file, folder, and functionality guide

**Repo:** [`opencode/`](../../opencode) (`anomalyco/opencode`, [opencode.ai](https://opencode.ai))
**Binary / npm:** `opencode` / `opencode-ai`
**Default git branch:** `dev`
**Companions:** [comparison-report.md](../compare/comparison-report.md) · [structure-comparison.md](../compare/structure-comparison.md) · [repo-guides.md](../README.md)
**Checkout verified:** 2026-08-29 · `opencode` **1.18.21** · default branch `dev`

OpenCode is **the open-source AI coding agent as a product**: TUI, desktop, IDE, browser, and CI are all **clients of one HTTP API**. The agent process is a local server. 75+ providers via models.dev + Vercel AI SDK. The same repo also holds the cloud console (Stripe), stats site, GitHub Action, Slack bot, and SST infrastructure.

Who it is for: people who want one agent that shows up in the terminal, a desktop app, VS Code, and GitHub comments — and a company-shaped OSS repo behind it.

---

## 1. How it works

```
opencode CLI  packages/opencode/src/index.ts  (yargs)
     │
     ├─ tui / run / serve / attach / acp / web / github / mcp / …
     │
     └─ starts or talks to the Server
            packages/server          Hono + Effect HttpApi   ← source of truth
                 │
                 ├─ packages/protocol    endpoint paths, envelopes, streams
                 ├─ packages/core        sessions, tools, permissions, plugins
                 ├─ packages/llm         providers, cache, tool runtime
                 └─ SQLite (Drizzle)     packages/core/src/database + effect-*-sqlite

Clients of the same HttpApi:
  TUI (@opencode-ai/tui in packages/tui; CLI bootstrap is packages/opencode/src/cli/cmd/tui.ts)
  Desktop (packages/desktop Electron + packages/app SolidStart)
  Web share view (packages/app)
  VS Code (sdks/vscode)
  Generated SDK (packages/client, packages/sdk-next)
  GitHub Action (github/)
```

**A prompt's path**

1. A client (TUI keystroke, `opencode run`, desktop, VS Code) POSTs a prompt onto the HttpApi.
2. V2 session runtime (`packages/core/src/session/`): durable **prompt admission** (`input.ts`) is decoupled from model execution (`execution.ts`, `runner/`).
3. A process-local **Session Drain** runs the turn. Steer vs queue happens at safe provider-turn boundaries.
4. `packages/llm` talks to the provider. System prompt is a **Context Epoch** — immutable cache-friendly baseline; changes become durable mid-conversation system messages (`system-context/`).
5. Tools run from `packages/core/src/tool/` and `packages/opencode/src/tool/` under the permission system (`build` = full access, `plan` = read-only). Oversized tool output is truncated; full text spills to a managed store.
6. Events: **replayable per-session SSE** (`sessions.events({sessionID, after})`) vs **live-only instance** `events.subscribe()` — different guarantees on purpose.

**Built-in agents:** Tab switches `build` / `plan`. `@general` invokes a subagent.

**Tools (core + CLI):** read, write, edit, bash/shell, glob, grep, webfetch, websearch, apply_patch, todowrite, skill, question, lsp, plan enter/exit, task, code-mode.

**Extension points:** `@opencode-ai/plugin` hooks (including PTY env), MCP, LSP, custom providers/commands, AGENTS.md discovery, skills.

**Embedded mode:** `packages/sdk-next` composes Client + Core + Server **in-process** (no HTTP hop) for “Embedded OpenCode”.

**Dependency direction (enforced):** Schema → Core / Protocol → Server → Client → sdk-next. Client runtime must never import Core or Server.

---

## 2. Root files

| File | Purpose |
|---|---|
| `package.json` | Bun workspaces (`packages/*`, `packages/console/*`, `packages/stats/*`, `packages/sdk/js`, `packages/slack`). Root `test` **exits 1** — tests run per package |
| `bun.lock` / `bunfig.toml` | Bun lockfile |
| `turbo.json` | Turborepo `typecheck` / `build` |
| `sst.config.ts` / `sst-env.d.ts` | SST v4 IaC (Cloudflare home + AWS us-east-1) |
| `tsconfig.json` | Root TS |
| `AGENTS.md` | Style + architecture rules (generated clients, layer direction, `dev` branch) |
| `CONTEXT.md` | Extra architecture context for agents |
| `CONTRIBUTING.md` / `SECURITY.md` / `LICENSE` | Process, reporting, MIT |
| `STATS.md` | Repo stats notes |
| `flake.nix` / `flake.lock` | Nix runner (`nix run github:anomalyco/opencode`) |
| `.oxlintrc.json` / `.prettierignore` / `.editorconfig` | Lint/format |
| `.dockerignore` / `.gitleaksignore` | Docker / secret scan |
| `README.md` + `README.*.md` | 18+ language translations of the landing README (treat as one artifact) |

---

## 3. Root directories

| Dir | Purpose |
|---|---|
| `packages/` | Almost the entire product (see §4) |
| `sdks/vscode/` | VS Code extension (launches opencode, file-reference shortcuts) |
| `github/` | Official GitHub Action (`/opencode fix this` → branch/PR). Own `package.json` / `bun.lock` |
| `infra/` | SST modules: `app.ts`, `console.ts`, `enterprise.ts`, `lake.ts`, `stats.ts`, `monitoring.ts`, `secret.ts`, `stage.ts` |
| `specs/` | Design specs: `project.md`, `tui-package.md`, `storage/`, `v2/` (session, tools, provider-model, config, instructions) |
| `script/` | Publish, changelog, generate, Windows signing, opentui upgrade, i18n |
| `nix/` | Nix packaging bits |
| `patches/` | 17 local npm patches (Effect, AI SDK providers, MCP SDK, solid-js, …) |
| `install/` | curl installer sources |
| `artifacts/` | Build artifacts (typically gitignored/generated) |
| `perf/` | Benchmarks |
| `.opencode/` | Dogfooding: `agent/`, `command/`, `skills/`, `plugins/`, `tool/`, `glossary/`, `opencode.jsonc` |
| `.github/workflows/` | `test.yml`, `typecheck.yml`, `publish.yml`, `deploy.yml`, desktop/vscode/action publish, triage, storybook, … |
| `.husky/` | Git hooks |

---

## 4. Packages — grouped by role

Cover every `packages/*` member. Cloud products nest under `console/` and `stats/`.

### Runtime contract chain

| Dir | npm name | What it does |
|---|---|---|
| `packages/schema` | `@opencode-ai/schema` | Public domain schemas (Effect Schema). No DB/runtime deps. **Leaf.** |
| `packages/core` | `@opencode-ai/core` | Domain core. `src/session/` (V2 drain, epochs, compaction), `src/tool/`, `src/permission/`, `src/plugin/`, `src/skill/`, `src/config/`, `src/database/`, `src/pty/`, `src/provider.ts`, `src/system-context/` |
| `packages/llm` | `@opencode-ai/llm` | Provider wire protocols, cache policy, tool runtime, provider errors |
| `packages/protocol` | `@opencode-ai/protocol` | Public API composition: paths, envelopes, cursors, streams |
| `packages/server` | `@opencode-ai/server` | Hono/Effect server hosting the authoritative `HttpApi` |
| `packages/httpapi-codegen` | `@opencode-ai/httpapi-codegen` | Compiles HttpApi → SDK Contract IR → generated clients |
| `packages/client` | `@opencode-ai/client` | Generated Promise client + Effect client (`/effect`). **Do not hand-edit `src/generated*`** — `bun run generate` from this package |
| `packages/sdk` / `sdk/js` | `@opencode-ai/sdk` | Legacy JS SDK (`./packages/sdk/js/script/build.ts` to regenerate) |
| `packages/sdk-next` | `@opencode-ai/sdk-next` | Effect-native in-process host (Client + Core + Server) |
| `packages/opencode` | `opencode` | **Product binary.** `src/index.ts` yargs CLI; `src/cli/cmd/` (tui, run, serve, attach, acp, web, github, mcp, plugin, session, db, …); `src/tool/` CLI-side tools; `src/server/` wiring; `bin/opencode` |

### Surfaces (UIs)

| Dir | npm name | What it does |
|---|---|---|
| `packages/tui` | `@opencode-ai/tui` | **The TUI.** SolidJS on `@opentui/solid`. `src/app.tsx`, `src/routes/`, `src/keymap.tsx`, `src/feature-plugins/`, `src/editor-zed.ts`. `packages/opencode/src/cli/cmd/tui.ts` only boots a worker and talks to this package (upstream CONTRIBUTING still points at `cli/cmd/tui/` as if the UI lived there) |
| `packages/app` | `@opencode-ai/app` | SolidStart web/desktop frontend (session viewer + desktop shell UI) |
| `packages/desktop` | `@opencode-ai/desktop` | Electron packaging (electron-builder / electron-vite) |
| `packages/ui` | `@opencode-ai/ui` | Shared SolidJS components |
| `packages/session-ui` | `@opencode-ai/session-ui` | Reusable session/conversation UI |
| `packages/storybook` | `@opencode-ai/storybook` | Component storybook |
| `packages/cli` | `@opencode-ai/cli` | Extra CLI helpers package |
| `packages/web` | `@opencode-ai/web` | Marketing/docs site (opencode.ai) |
| `packages/docs` | (Mintlify-style docs) | `docs.json`, `quickstart.mdx`, `essentials/`, `openapi.json` |

### Cloud / company

| Dir | npm name | What it does |
|---|---|---|
| `packages/console/app` | `@opencode-ai/console-app` | Cloud console web app (auth, billing UI) |
| `packages/console/core` | `@opencode-ai/console-core` | Console domain logic |
| `packages/console/function` | `@opencode-ai/console-function` | Console Lambdas |
| `packages/console/mail` | `@opencode-ai/console-mail` | Transactional email |
| `packages/console/resource` | `@opencode-ai/console-resource` | SST/cloud resources |
| `packages/console/support` | `@opencode-ai/console-support` | Support tooling |
| `packages/stats/app` | `@opencode-ai/stats-app` | Usage-stats site |
| `packages/stats/core` | `@opencode-ai/stats-core` | Stats domain |
| `packages/stats/server` | `@opencode-ai/stats-server` | Stats API |
| `packages/slack` | `@opencode-ai/slack` | Slack bot |
| `packages/enterprise` | `@opencode-ai/enterprise` | Enterprise control-plane |
| `packages/identity` | (assets) | Brand marks/icons (no package.json) |
| `packages/function` | `@opencode-ai/function` | Shared deployable function helpers |
| `packages/containers/` | (images) | Sandbox container defs: `base`, `bun-node`, `rust`, `tauri-linux`, publish scripts |

### Support

| Dir | npm name | What it does |
|---|---|---|
| `packages/plugin` | `@opencode-ai/plugin` | Public plugin API |
| `packages/codemode` | `@opencode-ai/codemode` | Effect-native confined code execution over schema-described tools |
| `packages/effect-drizzle-sqlite` | `@opencode-ai/effect-drizzle-sqlite` | Effect wrappers around Drizzle SQLite |
| `packages/effect-sqlite-node` | `@opencode-ai/effect-sqlite-node` | Effect bindings for node SQLite |
| `packages/http-recorder` | `@opencode-ai/http-recorder` | Record/replay Effect HTTP as deterministic cassettes |
| `packages/script` | `@opencode-ai/script` | Shared repo-script helpers |

### Outside `packages/`

| Path | What it does |
|---|---|
| `sdks/vscode/` | VS Code extension |
| `github/` | GitHub Action (`action.yml`, `index.ts`) |
| `infra/*.ts` | SST modules per service |
| `specs/v2/` | Session/tools/provider/config design docs |

---

## 5. Visual map

```
opencode/
├── AGENTS.md  CONTEXT.md  CONTRIBUTING.md  README*.md
├── package.json  bun.lock  turbo.json  sst.config.ts  flake.nix
├── patches/                      # AI SDK, Effect, MCP, solid-js, …
├── script/                       # publish, generate, sign-windows, i18n
├── specs/v2/                     # session runtime design
├── infra/                        # SST (app, console, stats, lake, …)
├── sdks/vscode/                  # IDE client
├── github/                       # GitHub Action (own package)
├── .opencode/                    # dogfooding agents, commands, skills
└── packages/
    ├── schema → core → llm/protocol → server → httpapi-codegen → client → sdk-next
    ├── opencode                  # CLI binary
    ├── tui  app  desktop  ui  session-ui  storybook  cli  web  docs
    ├── plugin  codemode  http-recorder  effect-*-sqlite  script
    ├── console/{app,core,function,mail,resource,support}
    ├── stats/{app,core,server}
    ├── slack  enterprise  function  containers  identity
    └── sdk/js                    # legacy SDK
```

---

## 6. Where do I find X?

| Looking for | Path |
|---|---|
| CLI entry | `packages/opencode/src/index.ts`, `packages/opencode/bin/opencode` |
| HttpApi | `packages/server/` + `packages/protocol/` |
| Session runtime | `packages/core/src/session/` (`execution.ts`, `input.ts`, `runner/`, `context-epoch.ts`) |
| Tools | `packages/core/src/tool/`, `packages/opencode/src/tool/` |
| Permissions / agents | `packages/core/src/permission/`, `packages/core/src/agent.ts` |
| Providers | `packages/llm/`, `packages/core/src/provider.ts`, `models-dev.ts` |
| TUI | `packages/tui/src/` (`@opencode-ai/tui`); bootstrap `packages/opencode/src/cli/cmd/tui.ts` |
| Desktop | `packages/desktop/` + `packages/app/` |
| VS Code | `sdks/vscode/` |
| GitHub Action | `github/` |
| Cloud console | `packages/console/`, `infra/console.ts` |
| Plugins | `packages/plugin/`, `packages/core/src/plugin/` |
| SQLite | `packages/core/src/database/`, `packages/effect-drizzle-sqlite/` |
| Generated clients | `packages/client/src/generated*` (regen, don't edit) |
| V2 specs | `specs/v2/` |
| Tests | **inside each package** (`bun test`); root `bun test` is forbidden |
| CI | `.github/workflows/test.yml`, `typecheck.yml`, `publish.yml` |
| Self-hosting config | `.opencode/` |

---

## 7. Conventions

- **One HttpApi is law.** Change protocol/server, then regenerate clients. Never edit generated files.
- **Layer rule:** Client never imports Core/Server. `sdk-next` is the allowed in-process composer.
- **Effect v4** everywhere in the runtime chain; SolidJS for UIs; Bun 1.3 workspaces.
- **Tests:** run from the package directory. Playwright e2e; `http-recorder` cassettes instead of mocks where possible.
- **Default branch is `dev`.**
- Heavy `patches/` — Effect/AI SDK/MCP are pinned *and* patched.
- Dogfood dir `.opencode/` is how the team develops OpenCode with OpenCode (agents, commands, skills, glossary translations).
