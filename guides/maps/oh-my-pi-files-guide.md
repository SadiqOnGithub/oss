# oh-my-pi (omp) — file, folder, and functionality guide

**Repo:** [`oh-my-pi/`](../../oh-my-pi) (`can1357/oh-my-pi`, [omp.sh](https://omp.sh))
**Binary:** `omp` (`@oh-my-pi/pi-coding-agent`)
**Companions:** [comparison-report.md](../compare/comparison-report.md) · [structure-comparison.md](../compare/structure-comparison.md) · [repo-guides.md](../README.md)
**Checkout verified:** 2026-08-29 · `@oh-my-pi/pi-coding-agent` **18.0.3**

omp is a **fork of pi** turned into a batteries-included coding agent: CLI/TUI plus SDK, stdio NDJSON RPC, and ACP for editors. It keeps `pi-*` package names and pi's agent/TUI/AI split, then adds ~80k lines of **Rust** (one N-API addon), LSP, DAP, browser/desktop control, collab, and a Python GitHub bot.

Who it is for: power users who want the IDE wired into the terminal and will run untrusted work in containers.

Headline numbers from the README: **60+** providers · **31** built-in tools · **14** LSP ops · **28** DAP ops · **~80k** LoC Rust core.

**Tool count:** `src/tools/builtin-names.ts` lists **29** `BUILTIN_TOOL_NAMES` plus hidden `yield` / `goal` / `think`. The README 31 adds `generate_image` and `tts` (`image-gen.ts`, `tts.ts` on disk).

---

## 1. How it works

```
omp (CLI)
  packages/coding-agent/src/cli.ts   (bin.omp, Bun shebang)
       │
       ├─ register commands, load natives  packages/natives → crates/pi-natives
       ├─ AgentSession                     packages/agent  (fork of pi-agent-core)
       ├─ providers                        packages/ai + packages/catalog
       ├─ tools                            packages/coding-agent/src/tools/
       │     hot paths (grep, bash, ast, PTY, desktop) run in Rust
       ├─ TUI                              packages/tui
       ├─ RPC / ACP                        src/modes/rpc (JSONL server), src/jsonrpc/message-framing.ts, src/tools/acp-bridge.ts
       └─ workers re-enter cli.ts via hidden argv (blob, LSP mux, computer, eval, daemon)
```

**A prompt's path**

1. `omp` parses flags/commands (`src/cli/`, `src/cli-commands.ts`).
2. Session + settings load; native addon is loaded once (`packages/natives`).
3. User prompt goes through `packages/agent` (same loop idea as pi).
4. Model call via `packages/ai` using the bundled catalog (`packages/catalog`).
5. Tool calls hit `src/tools/` — bash is an **embedded shell** (`crates/pi-shell` + `pi-builtins`), grep/glob/ast go to Rust, browser/computer/eval have their own workers.
6. Oversized context can be compressed (`snapcompact`); durable memories live in `mnemopi`.
7. Optional `/collab` shares a live session (`packages/wire` + `collab-web`).

**Surfaces:** interactive TUI, one-shot `-p`, Node SDK (`createAgentSession`), `omp --mode rpc` (NDJSON), ACP for Zed-class editors.

**Built-in tools** (`src/tools/builtin-names.ts`)

| Group | Names |
|---|---|
| Files | `read`, `write`, `edit`, `glob`, `grep` |
| AST | `ast_edit`, `ast_grep` |
| Shell / code | `bash` (in-process), `eval` (Python/JS/Julia/Ruby kernels) |
| IDE | `lsp`, `debug` (DAP) |
| Browser / desktop | `browser`, `computer` |
| GitHub / web | `github`, `web_search` |
| Images | `inspect_image` (+ `image-gen.ts` / `tts.ts` on disk) |
| Memory | `memory_edit`, `retain`, `recall`, `reflect`, `learn` |
| Orchestration | `task` (subagents in isolated worktrees), `hub`, `todo`, `ask`, `manage_skill` |
| Safety / time travel | `checkpoint`, `rewind`, `security_scan` |
| Hidden | `yield`, `goal`, `think` |

Essential (always top-level schema): `read`, `write`, `bash`, `edit`, `glob`, `computer`, `eval`, `task`, `hub`, `learn`, `manage_skill`.

**Extension points:** same registries as built-ins (tools, slash commands, hotkeys); MCP; managed skills; settings-gated `xd://` discoverable tools; interop with `.claude`, `.cursor`, `.windsurf`, `.gemini`, `.codex` configs.

No pi-style `protocol`/`client`/`server`/`telemetry`/`evals` packages — replaced by RPC/ACP/collab and `metaharness`.

---

## 2. Root files

| File | Purpose |
|---|---|
| `package.json` | Bun workspaces + catalog versions (`@oh-my-pi/*` lockstep, e.g. 18.0.3). `patchedDependencies` for `@ark/schema` and `puppeteer-core` |
| `bun.lock` / `bunfig.toml` | Bun lockfile and runtime config |
| `Cargo.toml` / `Cargo.lock` | Workspace of Rust crates |
| `rust-toolchain.toml` / `rust-analyzer.toml` / `rustfmt.toml` / `deny.toml` | Nightly pin, analyzer, format, cargo-deny advisories |
| `BUILD.bazel` / `MODULE.bazel` / `MODULE.bazel.lock` | Hermetic **shipping** builds of the native addon (rules_rust + zig cc) |
| `.bazelrc` / `.bazelversion` / `.bazelignore` | Bazel flags and version |
| `flake.nix` / `flake.lock` | Nix package, overlay, NixOS/Home Manager modules (`programs.omp`) |
| `biome.json` | TS lint/format |
| `tsconfig.json` / `tsconfig.base.json` / `tsconfig.tools.json` | TS planes |
| `AGENTS.md` | Agent rules; primary package is `packages/coding-agent` |
| `README.md` | Install (`curl omp.sh/install`, brew, bun, nix) |
| `CONTRIBUTING.md` | Contribution process |
| `LICENSE` / `THIRD-PARTY-NOTICES.txt` | MIT + native/third-party notices |
| `about.toml` | Project metadata |
| `Dockerfile` / `Dockerfile.robomp` + dockerignores | Container images for omp and the GitHub bot |
| `.fallowrc.jsonc` | Additional linter/config |

---

## 3. Root directories

| Dir | Purpose |
|---|---|
| `packages/` | TS workspace (see §4) |
| `crates/` | Rust native core (see §5) |
| `python/` | `omp-rpc` (typed RPC client) and `robomp` (GitHub triage bot) — see §6 |
| `bazel/` | Bazel macros / native build helpers |
| `docs/` | User + architecture docs (tools, MCP, collab, natives, compaction, memory, …) |
| `scripts/` | CI test orchestrator, release, brew formula, macOS signing, natives Bazel, installers |
| `nix/` | Nix packaging pieces consumed by `flake.nix` |
| `infra/` | e.g. bazel-remote / runner support |
| `assets/` | Branding / hero image |
| `patches/` | `puppeteer-core@25.3.0`, `@ark/schema@0.56.2` |
| `types/` | Extra TS type roots |
| `.omp/` | Dogfooding: `commands/`, `skills/` (`semantic-compression`, `system-prompts`, `tool-prompt-optimization`) |
| `.github/workflows/` | `ci.yml`, `nix.yml`, `bazel-cache-warm.yml` |
| `.cargo/` | Cargo config for local vs CI |

**`scripts/` worth knowing**

| Script | Purpose |
|---|---|
| `ci-test-ts.ts` | Per-package TS test orchestrator (used by `packages/*/test` scripts) |
| `ci-release-build-binaries.ts` / `ci-release-publish.ts` / `ci-release-checksums.ts` | Release pipeline |
| `ci-update-brew-formula.ts` | Homebrew tap |
| `ci-macos-sign.sh` | Notarization / signing |
| `bazel-natives.ts` | Drive Bazel native builds from TS |
| `install.sh` / `install.ps1` | Public installers |
| `sync-versions.ts` | Lockstep package versions |
| `link-omp.sh` | Dev symlink of the CLI |

---

## 4. TypeScript packages

### Kept from pi

| Dir | npm name | What it does |
|---|---|---|
| `packages/ai` | `@oh-my-pi/pi-ai` | Multi-provider LLM client (fork of pi-ai). Catalog *values* must be imported from `@oh-my-pi/pi-catalog`, not this barrel |
| `packages/agent` | `@oh-my-pi/pi-agent-core` | Agent runtime / tool loop (fork of pi-agent-core) |
| `packages/tui` | `@oh-my-pi/pi-tui` | Differential TUI framework |
| `packages/coding-agent` | `@oh-my-pi/pi-coding-agent` | The `omp` product (see below) |

### Added around the fork

| Dir | npm name | What it does |
|---|---|---|
| `packages/catalog` | `@oh-my-pi/pi-catalog` | Bundled model DB, provider descriptors, identity/classification |
| `packages/natives` | `@oh-my-pi/pi-natives` | Loads the platform `.node` addon (`pi_natives`) |
| `packages/hashline` | `@oh-my-pi/hashline` | Line-anchored patch language behind the edit tool (works on disk or in-memory) |
| `packages/omptype` | `@oh-my-pi/omptype` | ArkType-compatible validation with lazy JIT |
| `packages/utils` | `@oh-my-pi/pi-utils` | Shared CLI runner, dirs, workers, logger |
| `packages/wire` | `@oh-my-pi/pi-wire` | Collab/live-session wire types |
| `packages/mnemopi` | `@oh-my-pi/pi-mnemopi` | Local SQLite memory engine |
| `packages/snapcompact` | `@oh-my-pi/snapcompact` | Bitmap-frame context compression for vision models |
| `packages/stats` | `@oh-my-pi/omp-stats` | Local usage dashboard (`omp stats`) |
| `packages/collab-web` | `@oh-my-pi/collab-web` | Browser guest + local relay for `/collab` |
| `packages/browser-relay` | `@oh-my-pi/browser-relay` | Chrome extension so `browser` can drive existing tabs |
| `packages/metaharness` | `@oh-my-pi/pi-metaharness` | Benchmark runners, Harbor storage, REST/SSE, dashboard (replaced pi `evals`) |
| `packages/typescript-edit-benchmark` | `@oh-my-pi/typescript-edit-benchmark` | Edit-tool benchmark corpus |

`packages/tsconfig.workspace.json` is a shared TS config, not a package.

### `packages/coding-agent` — the `omp` CLI

| Path | Role |
|---|---|
| `src/cli.ts` | Process entry (`bin.omp`) and hidden worker selectors |
| `src/main.ts` | App startup |
| `src/cli/` | Commands (auth, bench, browser-relay, …) |
| `src/tools/` | All built-in tools (see §1) |
| `src/tools/browser/` | Puppeteer/Electron/relay |
| `src/tools/computer/` | Desktop control worker |
| `src/dap/` `src/lsp/` `src/debug/` | Debugger + language server |
| `src/session/` `src/mnemopi/` `src/memory-backend/` | Persistence + memory |
| `src/collab/` | Live sharing |
| `src/mcp/` | MCP client |
| `src/extensibility/` `src/slash-commands/` | User extensions |
| `src/task/` `src/advisor/` | Subagents + advisor model |
| `src/modes/rpc/` | JSONL RPC server (`rpc-mode.ts`, `rpc-client.ts`, protocol v2 chunks) |
| `src/jsonrpc/` | Shared message framing (`message-framing.ts`) used by RPC |
| `src/eval/` | Persistent language kernels |
| `src/prompts/` | Static `.md` prompt imports (not string-built in code) |
| `src/sdk.ts` | Public SDK |

---

## 5. Rust crates (`crates/`)

One N-API cdylib is what ships. Cargo is authoritative **locally** (rust-analyzer, nextest); **Bazel ships hermetically** (8 platform/ISA targets, including musl and dual AVX2/baseline x64).

| Crate | Job | Notable `src/` |
|---|---|---|
| `pi-natives` | Mega-addon: grep, diff, glob, PTY, clipboard, highlighting, desktop, tokenizers, PDF, crash handler, … | `grep.rs`, `desktop/`, `ast.rs`, `audio.rs` |
| `pi-shell` | Embedded bash wrapping vendored brush | `shell.rs`, `process.rs` |
| `pi-builtins` | Bash builtins + ported coreutils/findutils/sed/jq **in-process** (no fork) | `cat.rs`, `cd.rs`, `rg`-class utils |
| `pi-ast` | tree-sitter + ast-grep, 50+ grammars | `ops.rs`, `language/` |
| `pi-walker` | Parallel ignore-aware filesystem walker | `cache.rs` |
| `pi-iso` | Task isolation: APFS clones, btrfs/zfs reflinks, overlayfs, Windows projfs | `apfs.rs`, `overlayfs.rs`, `projfs.rs` |
| `pi-voice` | Capture/playback, Opus, WebRTC | `audio.rs`, `live.rs` |
| `vendor/brush-core` | Vendored bash implementation used by `pi-shell` | |

Blocking Rust work runs on the libuv pool; panics at the napi boundary become rejected promises.

---

## 6. Python (`python/`)

Not on the TS build graph. Separate uv/pip projects that **drive** `omp`.

### `python/omp-rpc`

Typed client for `omp --mode rpc` (NDJSON). Process-backed client, protocol v2 negotiation, typed tools so a Python host can expose custom tools.

### `python/robomp` — "robotic omp"

Self-hosted GitHub triage/fix bot: FastAPI service, per-issue git worktree, `omp --mode rpc` subprocess, PAT sidecar. Classifies issues → fix+PR / comment / close. `docker-compose.yml`, `web/`, `docs/`, own `AGENTS.md`. `Dockerfile.robomp` at repo root.

---

## 7. Visual map

```
oh-my-pi/
├── README.md  AGENTS.md  CONTRIBUTING.md  LICENSE
├── package.json  bun.lock  bunfig.toml  biome.json
├── Cargo.toml  rust-toolchain.toml  deny.toml
├── BUILD.bazel  MODULE.bazel  .bazelrc
├── flake.nix  nix/
├── Dockerfile  Dockerfile.robomp
├── patches/                 # puppeteer-core, @ark/schema
├── docs/                    # user + native architecture
├── scripts/                 # CI, release, install, bazel-natives
├── .omp/                    # dogfooding commands + skills
├── packages/                # TS (pi fork + extras)
│   ├── ai  agent  tui  coding-agent     ← kept names
│   ├── catalog  natives  hashline  omptype  utils  wire
│   ├── mnemopi  snapcompact  stats
│   ├── collab-web  browser-relay
│   └── metaharness  typescript-edit-benchmark
├── crates/                  # Rust → one N-API addon
│   ├── pi-natives  pi-shell  pi-builtins
│   ├── pi-ast  pi-walker  pi-iso  pi-voice
│   └── vendor/brush-core
└── python/
    ├── omp-rpc              # Python RPC client
    └── robomp               # GitHub bot
```

---

## 8. Where do I find X?

| Looking for | Path |
|---|---|
| CLI entry | `packages/coding-agent/src/cli.ts` |
| Agent loop | `packages/agent/` |
| Tool list | `packages/coding-agent/src/tools/builtin-names.ts` |
| Tool impls | `packages/coding-agent/src/tools/` |
| Native loader | `packages/natives/` |
| Rust addon | `crates/pi-natives/` |
| Embedded bash | `crates/pi-shell/` + `crates/pi-builtins/` |
| Providers | `packages/ai/` + `packages/catalog/` |
| TUI | `packages/tui/` |
| Memory | `packages/mnemopi/` + `packages/coding-agent/src/mnemopi/` |
| Compaction | `packages/snapcompact/` |
| Collab | `packages/wire/`, `packages/collab-web/`, `src/collab/` |
| Browser tool | `packages/coding-agent/src/tools/browser/` + `packages/browser-relay/` |
| RPC | `packages/coding-agent/src/modes/rpc/` (+ `src/jsonrpc/message-framing.ts`) · Python: `python/omp-rpc/` |
| GitHub bot | `python/robomp/` |
| Isolation / worktrees | `crates/pi-iso/` |
| Tests | `scripts/ci-test-ts.ts`; per-package tests |
| CI | `.github/workflows/ci.yml` |
| Self-hosting config | `.omp/` |

---

## 9. Conventions

- **Fork of pi:** four package directories keep `pi-*` names. Missing vs pi: `protocol`, `client`, `server`, `telemetry`, `evals`.
- **Two build systems:** Cargo for local Rust iteration; Bazel for reproducible shipping of `.node` binaries.
- **One native addon**, no fork/exec on hot paths (grep, bash builtins, AST).
- Prompts live in static `.md` imports under `src/prompts/`.
- Bun workers re-enter `cli.ts` via hidden argv so a compiled binary needs no extra worker entries.
- `AGENTS.md`: unless specified, work is `packages/coding-agent/`. Catalog values import from `@oh-my-pi/pi-catalog`, not `@oh-my-pi/pi-ai`.
