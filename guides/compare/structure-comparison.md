# File & Folder Structure Comparison

**Projects:** `deepseek-harness` · `opencode` · `pi` · `oh-my-pi`
**Companion to:** [comparison-report.md](./comparison-report.md) · [repo-guides.md](../README.md)
**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

---

## 1. Size at a Glance

| Metric | pi | oh-my-pi | opencode | deepseek-harness |
|---|---|---|---|---|
| Top-level entries | **18** | 39 | 50 | 46 |
| Git-tracked files | **1,401** | 6,583 | 6,523 | **7,903** |
| Packages under `packages/` | **10** (+1 backend) | 17 TS pkgs + 8 Rust crates | 32 | **50 groups** |
| Extra language roots | — | `crates/`, `python/`, `bazel/` | `sdks/`, `github/`, `infra/` | `apps/`, `python/`, `native/`, `vendor/` |

The shape mirrors the philosophy from the main report: **pi is one order of magnitude smaller than everything else**, dsh is the largest by file count despite shipping the least product surface, and opencode/omp are mid-sized but wide.

---

## 2. Top-Level Layout Side by Side

### Common "monorepo starter kit" (all four have)

```
.git/  .github/  .gitignore  .gitattributes
package.json  tsconfig*.json
AGENTS.md     ← agent-first development rules in all four
LICENSE  README.md(+ translations)  CONTRIBUTING.md  SECURITY.md(pi, omp)
scripts/  docs/(except pi)  patches/
node_modules/ (not tracked)
```

### What each repo adds on top

| Category | pi | oh-my-pi | opencode | deepseek-harness |
|---|---|---|---|---|
| **Build orchestration** | `tsconfig.base.json`, npm scripts, `test.sh` | `BUILD.bazel`, `MODULE.bazel`, `.bazelrc/.bazelversion`, `Cargo.toml/lock`, `rust-toolchain.toml`, `bunfig.toml` | `turbo.json`, `sst.config.ts`, `bunfig.toml`, multiple `vitest.*.config.ts` | `pnpm-workspace.yaml`, `tsdown.config.ts`, `pytest.ini`, many `tsconfig.*.json` planes |
| **Lint/format** | `biome.json` | `biome.json`, `rustfmt.toml`, `deny.toml`, `.fallowrc.jsonc` | `.oxlintrc.json`, `.prettierignore`, `.husky/` | `.oxlintrc.json`(+staged), `.jscpd.json`, `knip.json`, `lefthook.yml` |
| **Nix dev env** | ❌ | ✅ `flake.nix`, `nix/`, `.cargo/` | ✅ `flake.nix`, `nix/` | ❌ |
| **Docker** | ❌ | ✅ 2 Dockerfiles (+robomp variant), `.dockerignore` | ✅ `.dockerignore` only (infra lives elsewhere) | ❌ (sandbox backends instead; E2B POC) |
| **CI extras** | `pi-test.{sh,bat,ps1}` | `infra/bazel-remote`, runner configs | `github/action.yml` (the GitHub Action *is a top-level dir*) | `.gitlab-ci.yml` **+** GitHub workflows, `.agents/` skills dir |
| **i18n** | README translations only | ❌ | 18+ README translations | **Every doc tripled** (`*.md`, `*.zh.md`, `*.i18n.yaml`) |
| **Brand/marketing assets** | ❌ | `assets/` | screenshots in root | `BRAND_GUIDELINES.md`(+zh+i18n), `website/`, `examples/` |

---

## 3. The `packages/` Directories

This is where the architectural differences are most visible.

### pi — 10 packages: a clean dependency stack

```
packages/
├── tui              ─┐
├── telemetry         │ build order:
├── ai                │ tui → ai → agent → sqlite → protocol
├── agent             │ → client → server → coding-agent
├── session-backends/
│   └── sqlite-node   │
├── protocol         ─┤ remote-session split:
├── client           ─┤   protocol → server → client
├── server           ─┘
├── coding-agent      ← the actual `pi` CLI (product)
└── evals             ← behavioral eval harness
```

Every name maps 1:1 to a layer in the architecture diagram. No UI apps, no cloud services, no shared component libraries — the structure *is* the minimalism.

### oh-my-pi — 17 packages + 8 crates: fork of pi's stack, widened

Kept from pi: `agent`, `ai`, `tui`, `coding-agent` (same names — evidence of the fork).

Added around that core:

```
packages/                          crates/
├── catalog      (model DB)       ├── pi-natives    (N-API mega-addon)
├── natives      (Rust loader)    ├── pi-shell      (embedded bash)
├── hashline     (patch lang)     ├── pi-builtins   (coreutils-in-process)
├── omptype      (validation)     ├── pi-walker     (FS walker)
├── utils                         ├── pi-ast        (tree-sitter)
├── wire        (collab proto)    ├── pi-iso        (task isolation)
├── mnemopi     (memory engine)   └── vendor/
├── snapcompact (compression)         └── brush-core (bash fork)
├── stats / collab-web / browser-relay
├── metaharness / typescript-edit-benchmark   (benchmarking)
└── tsconfig.workspace.json
```

Note what's **missing vs pi**: no `protocol`, `client`, `server`, `telemetry`, or `evals` packages — omp replaced pi's experimental remote-stack with its own RPC/ACP/collab surfaces and moved evals into `metaharness`.

### opencode — 32 packages: layered core + product ecosystem

```
Core chain (strict direction):
schema → core → llm/protocol → server → httpapi-codegen → client → sdk-next
                                                     ↘ sdk (legacy)

Surfaces:            tui · app(web/desktop) · desktop · cli · ui · session-ui · storybook
Ecosystem/cloud:     console/*(app,core,function,mail,resource,support) · stats/*
                     enterprise · identity · slack · function · containers · codemode
Infra/support:       effect-drizzle-sqlite · effect-sqlite-node · http-recorder
                     plugin · script · web(docs site) · docs
```

The naming encodes the architecture: everything Effect-related is prefixed (`effect-*`), codegen is explicit (`httpapi-codegen`), and cloud products are grouped (`console/*`, `stats/*`). No other repo here ships its own marketing site, storybook, or Slack bot as workspace members.

Also unique: **product extensions live outside `packages/`** as first-class top-level dirs:

- `sdks/vscode/` — VS Code extension
- `github/` — the official GitHub Action (own package.json/bun.lock!)
- `infra/*.ts` — SST IaC modules per service (app, console, enterprise, lake, stats, monitoring, secret, stage)
- `specs/` — design specs (v2 session/tools/provider/config/storage) kept in-repo

### deepseek-harness — 50 flat groups + apps/: capability seams

```
apps/          cli · web                    ← launchers (like pi's coding-agent role)
packages/      50 FLAT directories, no nesting:
  Spine:         core · api · typert · boot · preset · bundle
  Capabilities:  llm · shell · subprocess · terminal · code-runtime · sandbox
                 fs · lsp · web · compaction · context · skill · mcp · e2b
  Orchestration: subagent · jobs · workflow · todo · plan · guard · extensions
                 goal · schedule · feedback · interaction
  Data plane:    session · session-query · attachment · spill · storage
                 workspace · settings · credentials · identity
  Delivery:      host · client · sdk · acp · hooks
  Support:       util · test-support · runtime-diagnostics · examples · experimental
python/        sdk · sdk-runtime            ← PyPI distribution carrier
native/        landlock-run                 ← C++ N-API sandbox launcher
vendor/        cordis + cosmokit/schemastery etc.  ← vendored framework, pinned SHAs
docs/          ~20 bilingual architecture docs + catalogs + postmortems
.agents/       agent skills + decision records
website/       VitePress docs projection
```

Distinctive structural traits:

- **Flat, uniform naming**: every package is a *capability seam* (`fs`, `shell`, `sandbox`, `lsp`…), not a layer. Compare with opencode where package names describe *positions in a dependency chain*.
- **`vendor/` with governance**: the Cordis framework is source-vendored, rescoped, SHA-pinned, with an exhaustive modification log and its own AGENTS.md/CLAUDE.md inside.
- **Docs are a build artifact**: generated catalogs (`tool-catalog`, `persistence-catalog`, `module-graph`, `graph-atlas`) are freshness-gated in CI; every doc has an English, Chinese, and i18n-metadata variant.
- **`.agents/` directory**: in-repo agent skills and decision-record notes — pi keeps similar config in `.pi/`, omp in `.omp/`, opencode in `.opencode/`; all four are self-hosting their own tooling.

---

## 4. Self-Hosting Config Dirs (each repo develops itself with itself)

| Repo | Dir | Contents |
|---|---|---|
| pi | `.pi/` | extensions (tps/redraw diagnostics), prompt templates (`pr.md`, `sa.md`, `cl.md`…), skills, git/npm state dirs |
| oh-my-pi | `.omp/` | project-local omp configuration |
| opencode | `.opencode/` | agents/config for developing opencode with opencode |
| deepseek-harness | `.agents/` + `CLAUDE.md` + `AGENTS.md` | skills, decision records; plus `packages/AGENTS.md`, `packages/CLAUDE.md`, vendor-level AGENTS.md — nested instruction scopes |

pi goes furthest: it also publishes maintainer dev sessions as datasets, and its test scripts (`pi-test.sh`) exist specifically for tmux-driven self-testing of the CLI.

---

## 5. Structural Conventions Compared

| Convention | pi | oh-my-pi | opencode | deepseek-harness |
|---|---|---|---|---|
| Package grouping | Flat | Flat | Flat (cloud via `console/*`, `stats/*` subdirs) | Flat, **50** dirs |
| Multi-language roots | none | `crates/`, `python/`, `bazel/`, `assets/` | none (TS only in ws) | `apps/`, `python/`, `native/`, `vendor/` |
| Docs location | scattered (README, CONTRIBUTING, rpc docs in pkg) | `docs/` (~80 files, flat) | `specs/` + `packages/web` content + `packages/docs` | `docs/` structured: cookbook/, user/, subsystems/, postmortem/, catalogs |
| Test infra placement | per-package + root `test.sh` | per-package + `scripts/ci-test-ts.ts` orchestrator + `metaharness` pkg | per-package + root vitest configs ×5 + `http-recorder` pkg | per-package + 5 vitest configs + `test-support` + `runtime-diagnostics` pkgs |
| Release scripts | `scripts/build-binaries.sh` | `scripts/` release family + Homebrew/Nix/mise | `script/publish.ts` + sign-windows.ps1 + install script | `scripts/` (~80 gates/generators) + python carrier |
| Lockfiles | package-lock.json | bun.lock + Cargo.lock | bun.lock | pnpm-lock.yaml + Cargo? (no — native built per-platform) |
| Translations | README only (—) | none | README ×18 languages | README + CONTRIBUTING + BRAND + **every doc file** (en/zh/i18n.yaml) |

---

## 6. Reading Structure as Philosophy

1. **pi**: `packages/` reads like a textbook layer diagram. Ten names, zero surprises. If you can't guess where code lives from the architecture, the structure failed — so it doesn't.

2. **oh-my-pi**: pi's skeleton with growth rings around it — the four preserved pi packages sit beside nine purpose-built ones (`hashline`, `mnemopi`, `snapcompact`…) and a whole parallel `crates/` universe bridged by exactly one package (`natives`). The boundary between TS and Rust is a single structural seam.

3. **opencode**: two distinct trees in one repo — a *framework* tree (strictly ordered core chain, codegen, generated clients) and a *business* tree (console, stats, slack, enterprise). Plus infra-as-code and the GitHub Action as sibling top-level dirs: the repo is simultaneously the product and the company's deployment unit.

4. **deepseek-harness**: no layers visible in the tree at all — just capabilities. Layering exists only in the boot composition (profiles/bundles), not the folder layout. The heavy investment is instead in *governance structure*: vendor logs, generated catalogs, gate scripts, trilingual docs, postmortems. It's the only repo where `docs/` out-numbers most other repos' entire source tree per-file discipline.

---

## 7. Quick Reference: Where Do I Find X?

| Looking for… | pi | oh-my-pi | opencode | dsh |
|---|---|---|---|---|
| Agent loop | `packages/agent` | `packages/agent` | `packages/core` (src/session…) | `packages/core` (swappable) |
| Tools impl | `packages/coding-agent/src/core/tools` (read/bash/edit/write) | `src/tools/` (29 named in `builtin-names.ts`; README 31) + Rust crates | `packages/core/src/tool/builtins.ts` (12) | 23 `tool-*` packages |
| TUI framework | `packages/tui` | `packages/tui` | `packages/tui` (@opentui/solid) | ❌ (Web UI: `host`/`client`) |
| Provider layer | `packages/ai` | `packages/ai` + `catalog` | `packages/llm` + AI SDK | `packages/llm*` adapters |
| Remote API/protocol | `protocol`→`server`→`client` | `wire` + RPC mode + ACP | `protocol`→`server`→`httpapi-codegen`→`client` | `sdk`, `acp`, `api`, `typert` |
| Persistence | CLI JSONL (`session-manager.ts`); sqlite-node is a library | JSONL sessions + `mnemopi` memories | `core` + `effect-*-sqlite` | `session` + `session-query` |
| Sandboxing | external (docs) | `crates/pi-iso` | `packages/containers` | `packages/sandbox` + `native/landlock-run` |
| CI/release | `scripts/`, `.github/` | `scripts/`, `bazel/`, `infra/` | `script/`, `infra/`, `.github/` | `scripts/`, `.github/` + `.gitlab-ci.yml` |
