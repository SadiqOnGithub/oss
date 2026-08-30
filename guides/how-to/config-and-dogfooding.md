# Config and dogfooding

Two different “config” trees in every project:

1. **User/runtime** — what `pi` / `omp` / `opencode` / `dsh` read when *you* run the product (`~/.pi`, `~/.omp`, `~/.config/opencode`, `~/.dsh`).
2. **Dogfooding** — what the *maintainers* commit so the agent can work on its own repo (`.pi/`, `.omp/`, `.opencode/`, `.agents/`).

Mixing them up is the usual “I edited the wrong AGENTS.md” failure.

Pair with [glossary.md](../reference/glossary.md) (dogfooding dir vs home dir) and the file guides.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

---

## Cheat sheet

| | User home (runtime) | Project (runtime) | Committed dogfood | Relocate home with |
|---|---|---|---|---|
| **pi** | `~/.pi/agent/` | `<cwd>/.pi/` | `pi/.pi/` | `PI_CODING_AGENT_DIR` |
| **omp** | `~/.omp/agent/` | `<cwd>/.omp/` | `oh-my-pi/.omp/` | `PI_CODING_AGENT_DIR` (yes, still `PI_`) |
| **opencode** | `~/.config/opencode/` (XDG) | `opencode.json(c)` + `.opencode/` | `opencode/.opencode/` | `OPENCODE_TEST_HOME` (tests) |
| **dsh** | `~/.dsh/` | workspace is cwd; profiles are in home | `.agents/`, `AGENTS.md`, `CLAUDE.md` | `DSH_HOME` |

omp project **settings** do not walk ancestors. omp **context files** (`AGENTS.md`) walk to the nearest non-empty `.omp/`. Those are different algorithms.

---

## pi

### Runtime — `~/.pi/agent/`

From `packages/coding-agent/src/config.ts` (`CONFIG_DIR_NAME = ".pi"`):

| Path | Role |
|---|---|
| `settings.json` | Global settings |
| `auth.json` | Provider credentials |
| `models.json` | Extra/override models |
| `sessions/` | Session store (also sqlite backend) |
| `extensions/` | Auto-loaded extensions (`/reload`) |
| `prompts/` | Prompt templates (`/template`) |
| `themes/` | TUI themes |
| `tools/` | Tool-related files |
| `bin/` | Managed binaries (fd, rg) |
| `pi-debug.log` | Debug log |

`PI_CODING_AGENT_SESSION_DIR` overrides session location only.

### Project — `<cwd>/.pi/`

Extensions, prompts, skills that travel with the repo. Trust: interactive TUI prompts; print/RPC/json use `defaultProjectTrust` (`ask`/`never` skip project resources unless `--approve`).

### Dogfood — `pi/.pi/`

| Path | What it is |
|---|---|
| `extensions/tps.ts` | Tokens-per-second widget |
| `extensions/redraws.ts` | TUI redraw counter |
| `extensions/prompt-url-widget.ts` | Prompt URL widget |
| `extensions/import-repro.ts` | Import diagnostic |
| `prompts/{pr,sa,cl,is,wr}.md` | Maintainer templates |
| `skills/add-llm-provider.md` | How to add a provider to `pi-ai` |

Root `AGENTS.md` is **contributor law** (erasable TS, `./test.sh`, no core bloat), not a skill.

---

## oh-my-pi

### Runtime — `~/.omp/agent/`

Canonical file: **`config.yml`** (YAML). `omp config path` prints the directory.

| Path | Role |
|---|---|
| `config.yml` | Global settings (`omp config set`, `/settings`) |
| `config.yaml` | Compat filename; updated in place if it already exists |
| `settings.json` | Legacy; migrated once to YAML, renamed `.bak` |
| `agent.db` | Auth / agent store |
| `AGENTS.md` | User-level context file |
| `RULES.md` | Sticky rules (re-attached near the turn) |
| `tools/` | Custom tools |
| `extensions/` | Extensions |

`PI_CODING_AGENT_DIR` moves this whole tree. Named profile: `~/.omp/profiles/<name>/agent` (`omp --profile`, `OMP_PROFILE`, `PI_PROFILE`).

`omp config list` / `get` / `set` / `reset` — writes go to **global** YAML except `modelRoles` when `modelRoleStorage: project`.

### Project — `<cwd>/.omp/`

| Path | Role |
|---|---|
| `config.yml` | Project settings (loaded only if `.omp/` is non-empty **in cwd** — no ancestor walk) |
| `settings.json` | Legacy project settings |
| `AGENTS.md` / `RULES.md` | Context / sticky — discovered from nearest non-empty `.omp/` walking **toward repo root** |
| `tools/`, `skills/`, `commands/` | Native discovery |

Also read (read-only, disable per provider): `.claude`, `.cursor`, `.windsurf`, `.gemini`, `.codex`, OpenCode files. See [context-files.md](../../oh-my-pi/docs/context-files.md).

### Dogfood — `oh-my-pi/.omp/`

| Path | What it is |
|---|---|
| `commands/` | Project slash-command-ish maintainer commands |
| `skills/semantic-compression` | Compression skill |
| `skills/system-prompts` | Prompt skill |
| `skills/tool-prompt-optimization` | Tool-prompt skill |

`AGENTS.md` at repo root: work is `packages/coding-agent/` unless said otherwise; catalog values import from `@oh-my-pi/pi-catalog`.

---

## OpenCode

### Runtime — XDG (`packages/core/src/global.ts`)

App name `opencode`:

| XDG | Linux default | Role |
|---|---|---|
| config | `~/.config/opencode/` | `opencode.json` / `opencode.jsonc` |
| data | `~/.local/share/opencode/` | `log/`, `repos/` |
| cache | `~/.cache/opencode/` | `bin/` |
| state | `~/.local/state/opencode/` | locks |
| tmp | `$TMPDIR/opencode` | scratch |

V2 discovers `opencode.json` / `opencode.jsonc` in the global config dir, ancestor project dirs, and `.opencode/` dirs. Legacy `config.json` is **not** V2. (`specs/v2/config.md`)

### Project

| Path | Role |
|---|---|
| `opencode.json(c)` | Project config (`$schema`, providers, plugin, skills, instructions, permissions, agents) |
| `.opencode/agents` or `agent/` | Custom agents |
| `.opencode/command/` | Named commands (dogfood uses this; v2 wants skills for user workflows) |
| `.opencode/skills/` | Skills |
| `.opencode/plugins/` / `plugin` in json | Plugins |
| `.opencode/tool/` | Extra tools (repo dogfood has GitHub helpers) |
| `AGENTS.md` | Instruction discovery (system-context source) |

### Dogfood — `opencode/.opencode/`

| Path | What it is |
|---|---|
| `opencode.jsonc` | How OpenCode configures OpenCode |
| `agent/triage.md`, `duplicate-pr.md` | Maintainer agents |
| `command/{commit,changelog,issues,learn,…}.md` | Slash-like maintainer commands |
| `skills/effect`, `skills/rtl-aware-development` | Skills for this codebase |
| `tool/github-pr-search.ts`, `github-triage.ts` | Extra tools |
| `glossary/` | i18n glossary for the product UI |
| `plugins/` | Local plugins |

Root `AGENTS.md`: layer rule (Client never imports Core/Server), regenerate clients, default branch `dev`.

---

## deepseek-harness

### Runtime — `~/.dsh/` (`DSH_HOME`)

A running `dsh` is **not** “files in the git repo.” It is a composed Cordis tree:

```
empty
  → each bundle in profile.bundles      (dsh-base, then web-app or headless)
  → $DSH_HOME/profiles/<name>/cordis.patch.yml
  → $DSH_HOME/cordis.patch.yml
  → --patch files
```

| Path | Role |
|---|---|
| `$DSH_HOME/profiles/web/` | Default Web profile (auto-created) |
| `$DSH_HOME/profiles/headless/` | One-shot profile |
| `$DSH_HOME/profiles/<name>/package.json` | `dsh.profile.bundles` + out-of-tree plugins |
| `$DSH_HOME/cordis.patch.yml` | User overlay for every profile |
| credentials / settings / sessions | whatever the composed session/settings/credentials plugins store under home |

Inspect without booting work:

```bash
pnpm dsh --profile web --dump-config
pnpm dsh --profile web --dump-default-config
```

`dsh plugin --profile <name> …` forwards to pnpm **inside the profile directory**.

Workspace root = invoking cwd. The Web UI still needs you to **Choose workspace** once.

### Dogfood — repo, not `$DSH_HOME`

| Path | What it is |
|---|---|
| `AGENTS.md` / `CLAUDE.md` | Contributor rules (also nested under `packages/`, `vendor/`) |
| `.agents/skills/` | Skills the team uses on this repo (`dsh-code-review`, `dsh-doc-standards`, …) |
| `.agents/notes/proposed` | Ideas |
| `.agents/notes/implemented` | Decision records |
| `.agents/notes/rejected` | Rejected so the fallacy does not return |
| `.agents/notes/archived` | Frozen history |
| `docs/` | Architecture + generated catalogs |

There is no committed `.dsh/` product config in the git tree the way `.pi` exists. Profiles live on the machine that ran `dsh`.

---

## Instruction files vs product config

| File | pi | omp | opencode | dsh |
|---|---|---|---|---|
| Root `AGENTS.md` | Contributor rules **and** product may ingest project AGENTS.md | Contributor rules; product reads `.omp/AGENTS.md` + user one | Contributor rules; product discovers AGENTS.md as context source | Contributor rules; product has `packages/context/agent-instructions` |
| `CLAUDE.md` | — | interop via claude provider | — | Yes, nested |
| Skills | `.pi/skills`, `~/.pi/agent` | `.omp/skills`, marketplace | `.opencode/skills` | `packages/skill` + `.agents/skills` for *this* repo |
| MCP | not in core | `mcp-config.md` | `opencode.json` + CLI `mcp` | `packages/mcp` plugin |

When you “add instructions for the agent working in *my app*,” put them in the **project** runtime location (`.pi/`, `.omp/AGENTS.md`, `AGENTS.md` / `.opencode/`, or a dsh context plugin) — not in the harness repo’s dogfood dir unless you are hacking the harness itself.

---

## Practical debug

```bash
# which files is omp actually using?
omp config path
omp config list

# which plugins will dsh boot?
pnpm dsh --profile web --dump-config | less

# opencode: start at project opencode.jsonc then ~/.config/opencode/
ls -la .opencode opencode.json opencode.jsonc ~/.config/opencode 2>/dev/null

# pi: sessions + auth
ls -la ~/.pi/agent
```
