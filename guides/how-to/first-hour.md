# First hour in each repo

**Companions:** [repo-guides.md](../README.md) · [pi-files-guide.md](../maps/pi-files-guide.md) · [oh-my-pi-files-guide.md](../maps/oh-my-pi-files-guide.md) · [opencode-files-guide.md](../maps/opencode-files-guide.md) · [deepseek-harness-files-guide.md](../maps/deepseek-harness-files-guide.md)

This is the **what to type** guide. Folder maps tell you where files live; this tells you how to build, run, and not torch your credentials.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

All four repos are already under this workspace (`pi/`, `oh-my-pi/`, `opencode/`, `deepseek-harness/`). Each section is self-contained. As of 2026-08-29 this workspace: **opencode** `bun dev --help` / `run --help` work; **dsh** `pnpm dsh --help` and `--dump-default-config` work (Node v22.18.0, engine warning). **pi** `npm install --ignore-scripts` works but `./pi-test.sh` needs `npm run build` first (generated `packages/ai/src/providers/data/.manifest.json`). **omp** needs Bun 1.4+; Bun 1.2.13 cannot resolve `catalog:` deps.

**Safe tests:** the commands labeled “safe test” do **not** need provider API keys. Anything that talks to a model will spend tokens.

---

## pi

**Binary:** `pi` · **runtime:** Node ≥22.19 · **install:** npm workspaces

This checkout: `npm install --ignore-scripts` succeeded (321 packages) on Node v22.18.0 with `EBADENGINE` warnings (`>=22.19.0` required). `./pi-test.sh --help` then **failed** with `ERR_MODULE_NOT_FOUND` for `packages/ai/src/providers/data/.manifest.json`. That file is **generated**; install is not enough.

### Setup (from `pi/`)

```bash
npm install --ignore-scripts   # skip lifecycle scripts on purpose
npm run build                  # generates model data under packages/ai/src/providers/data/, then builds tui → … → coding-agent (needs network)
# or, offline after a previous catalog fetch:
npm run build:offline
```

Do not run `./pi-test.sh` until `packages/ai/src/providers/data/.manifest.json` exists.

### Run from source

```bash
./pi-test.sh                   # CLI from this checkout, any cwd (needs the generated manifest)
./pi-test.sh --no-env          # same, with provider env vars stripped
```

Interactive TUI needs a real TTY. One-shot (needs a key, e.g. `ANTHROPIC_API_KEY`):

```bash
./pi-test.sh -p "Reply with the word ping and nothing else"
```

First-run auth also lives in the TUI (`/login` style flows). Credentials land in `~/.pi/agent/auth.json`.

### Where state goes

| Thing | Path |
|---|---|
| Global config / sessions / debug log | `~/.pi/agent/` (`settings.json`, `sessions/`, `pi-debug.log`) |
| Project extensions / prompts / skills | `<repo>/.pi/` |
| Override home | `PI_CODING_AGENT_DIR` |

### Safe test

```bash
./test.sh                      # isolated fake HOME; skips LLM e2e without keys
# NOT: npm test from root with keys set — that can fire paid evals
```

After a code change: `npm run check` (Biome + pins + types). Do not run `npm run build` unless you need artifacts.

### First files to open

1. `packages/coding-agent/src/cli.ts` → `main.ts`
2. `packages/agent/src/agent-loop.ts`
3. `packages/coding-agent/src/core/tools/read.ts`

---

## oh-my-pi (omp)

**Binary:** `omp` · **runtime:** Bun **1.4+** (`package.json` `"packageManager": "bun@1.4.0"`) · **also:** Cargo for native, Bazel to *ship* natives

This checkout: Nix `bun` 1.2.13 **cannot** `bun install` — it dies on `catalog:` (`failed to resolve @oh-my-pi/pi-ai@catalog:`). Use Bun 1.4+ (mise/proto/official installer), then install.

### Setup (from `oh-my-pi/`)

```bash
bun --version                  # must be 1.4.x
bun install
```

The native addon (`packages/natives` → `crates/pi-natives`) must load. Host addons are produced by the natives package `build` script (cargo/napi-rs by default; Bazel if you ask):

```bash
bun --cwd=packages/natives run build
# same from repo root:
bun run build:native
```

That is `bun ../../scripts/bazel-natives.ts host --dest native`. The `host` target uses the local cargo/napi-rs path unless you set `OMP_NATIVE_BUILD_BACKEND=bazel` or pass extra bazel args after `--`. Cross-platform shipping builds use Bazel: `bun scripts/bazel-natives.ts linux-x64-modern --dest packages/natives/native`. Cargo/`cargo nextest` in `crates/` is for rust-analyzer and unit tests, not the `.node` you run. See [docs/natives-build-release-debugging.md](../../oh-my-pi/docs/natives-build-release-debugging.md).

### Run from source

From `packages/coding-agent/` (this *is* the product):

```bash
bun run check                  # typecheck + lint gate — never raw tsc
bun src/cli.ts                 # interactive TUI against cwd
bun src/cli.ts -p "Reply with ping"
```

Repo root may expose `omp` via install/link (`scripts/link-omp.sh`). `DEVELOPMENT.md` is the map: [oh-my-pi/packages/coding-agent/DEVELOPMENT.md](../../oh-my-pi/packages/coding-agent/DEVELOPMENT.md).

### Where state goes

| Thing | Path |
|---|---|
| Global settings | `~/.omp/agent/config.yml` (`omp config path` prints the dir) |
| Auth / agent.db | same agent dir |
| Project config / skills | `<cwd>/.omp/` (does **not** walk ancestors for settings) |
| Relocate agent dir | `PI_CODING_AGENT_DIR` |
| Named profile | `~/.omp/profiles/<name>/agent` (`omp --profile`, `OMP_PROFILE`) |

### Safe test

```bash
cd packages/coding-agent && bun run check
# bun run test is the heavy suite (scripts/ci-test-ts.ts) — run when you changed behavior
```

### First files to open

1. `packages/coding-agent/src/cli.ts` → `main.ts` → `sdk.ts`
2. `packages/coding-agent/src/tools/builtin-names.ts`
3. `crates/pi-natives/` (why bash/grep do not fork)

---

## OpenCode

**Binary:** `opencode` · **runtime:** Bun 1.3 · **default git branch:** `dev`

### Setup (from `opencode/`)

```bash
bun install
bun dev                        # TUI against packages/opencode
bun dev .                      # TUI against *this* repo (dogfood)
bun dev /path/to/other/project
```

Root `package.json` `dev` is `packages/opencode/src/index.ts`. Desktop: `bun run dev:desktop`. Web app: `bun run dev:web`.

Verified on this checkout: `bun dev --help` lists `run`, `serve`, `web`, `acp`, … One-shot:

```bash
bun dev run "Reply with ping"
# or after a local compile:
# ./packages/opencode/script/build.ts --single
```

`opencode serve` tries port **4096** first, then any free port (`packages/opencode/src/server/server.ts`). The yargs `--port` default of `0` means “unspecified, use that prefer-4096 logic.”

### Where state goes (XDG)

| Thing | Path (Linux) |
|---|---|
| User config | `~/.config/opencode/` (`opencode.json` / `opencode.jsonc`) |
| Data / logs | `~/.local/share/opencode/` (`log/`, `repos/`) |
| Cache | `~/.cache/opencode/` |
| Project config | `<repo>/opencode.json(c)` and `<repo>/.opencode/` |
| Override home in tests | `OPENCODE_TEST_HOME` |

### Safe test

```bash
# Root `bun test` is designed to FAIL. Run inside a package:
cd packages/core && bun test
cd packages/opencode && bun test
bun turbo typecheck
```

### First files to open

1. `packages/opencode/src/index.ts` (yargs: tui, run, serve, …)
2. `packages/protocol/src/groups/session.ts` (`POST /api/session/:id/prompt`)
3. `packages/core/src/session/input.ts` + `execution.ts`

---

## deepseek-harness (dsh)

**Binary:** `dsh` · **runtime:** Node `^22.19.0 || >=24` · **package manager:** pnpm 11.7 via Corepack

This checkout ran `pnpm dsh --help` and `pnpm dsh --profile web --dump-default-config` successfully on Node v22.18.0, with an engine warning. Prefer 22.19+ to match `package.json`.

### Setup (from `deepseek-harness/`)

```bash
corepack enable                # if pnpm isn't the pinned one
pnpm install
pnpm run typecheck             # first-time gate: must pass
pnpm run build                 # needed before `pnpm dsh web` uses artifacts
```

### Run from source

```bash
pnpm dsh web                   # Web UI, default http://127.0.0.1:3080
pnpm dsh web --no-open
pnpm dsh --profile headless "Reply with ping"
pnpm dsh --profile web --dump-config    # print composed plugin tree, no boot
```

In the UI: **Settings → Models** (DeepSeek API key), **Choose workspace**, then send a prompt. Fresh UI has no workspace until you pick one.

### Where state goes

| Thing | Path |
|---|---|
| Harness home | `~/.dsh/` (`DSH_HOME` overrides) |
| Profiles | `$DSH_HOME/profiles/<name>/` (`package.json` + `cordis.patch.yml`) |
| Home overlay | `$DSH_HOME/cordis.patch.yml` |
| Repo dogfooding | `.agents/skills`, `.agents/notes` |
| Session files | under home/storage as composed by session-persistence plugins |

### Safe test

```bash
pnpm test                      # vitest unit — no keys
# Smaller smoke verified here: pnpm exec vitest run packages/util/timeout
# Heavier (still mostly keyless): pnpm run test:snapshot
# Needs build first: pnpm run test:web:built
# Do not start with check:all — that is the full CI mountain
```

### First files to open

1. `apps/cli/src/bin.ts` → `profile-boot.ts`
2. `docs/architecture.md` (turn flow)
3. `packages/core/agent-loop/` and `packages/core/tools/`

---

## Side by side

| | pi | omp | opencode | dsh |
|---|---|---|---|---|
| Dev run | `./pi-test.sh` | `bun src/cli.ts` in coding-agent | `bun dev` | `pnpm dsh web` |
| One-shot | `pi -p "…"` | `omp -p "…"` | `opencode run "…"` | `dsh --profile headless "…"` |
| Safe check | `./test.sh` | `bun run check` | `bun test` *in a package* | `pnpm test` |
| Do not | `npm test` at root with keys | raw `tsc` | `bun test` at root | skip `pnpm run build` then expect `dsh web` to work |
| Surface | TUI | TUI | TUI (server+client) | Web UI |

Next: follow one prompt through the code in [prompt-traces.md](./prompt-traces.md).
