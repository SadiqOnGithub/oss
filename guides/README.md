# Guides

Notes for the four coding-agent checkouts in this workspace: `pi/`, `oh-my-pi/`, `opencode/`, `deepseek-harness/`.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

```
guides/
├── README.md                 ← you are here
├── compare/                  cross-repo comparison
├── maps/                     one file/folder map per repo
├── how-to/                   run it, follow a prompt, extend it
└── reference/                glossary
```

The four source repos are untouched. A session transcript lives in [`sessions/`](../sessions/ses_fce4.md).

---

## Compare — pick a project, see how they differ

| Doc | Use when |
|---|---|
| [compare/comparison-report.md](./compare/comparison-report.md) | Capabilities, architecture, matrix, which to choose |
| [compare/structure-comparison.md](./compare/structure-comparison.md) | Side-by-side folder layout |

## Maps — open a folder and know what it is

| Doc | Binary | One-liner |
|---|---|---|
| [maps/pi-files-guide.md](./maps/pi-files-guide.md) | `pi` | Minimal kernel. Four tools. Extensions instead of features. |
| [maps/oh-my-pi-files-guide.md](./maps/oh-my-pi-files-guide.md) | `omp` | Pi fork + Rust native core + IDE wiring. |
| [maps/opencode-files-guide.md](./maps/opencode-files-guide.md) | `opencode` | One HttpApi; TUI/desktop/IDE/cloud are clients. |
| [maps/deepseek-harness-files-guide.md](./maps/deepseek-harness-files-guide.md) | `dsh` | Everything is a Cordis plugin, including the agent loop. |

Each map: **what it is** → **how it works** → **root files** → **folders** → **packages** → **visual map** → **where do I find X**.

## How-to — do a thing

| Doc | Use when |
|---|---|
| [how-to/first-hour.md](./how-to/first-hour.md) | Install, run TUI/Web, safe tests, where state lives |
| [how-to/prompt-traces.md](./how-to/prompt-traces.md) | Follow one user prompt through each call stack |
| [how-to/add-a-tool.md](./how-to/add-a-tool.md) | Make the model call your `hello` tool |
| [how-to/config-and-dogfooding.md](./how-to/config-and-dogfooding.md) | `~/.pi` vs `.pi/`, `$DSH_HOME` vs `.agents/` |
| [how-to/drive-from-process.md](./how-to/drive-from-process.md) | RPC, HttpApi, Python SDKs, ACP — not a TUI |

## Reference

| Doc | Use when |
|---|---|
| [reference/glossary.md](./reference/glossary.md) | “Plugin”, “session”, “provider”, or “patch” showed up again |

---

## Suggested reading order

1. This page — pick a repo.
2. That repo’s **map**.
3. **First hour** for that repo.
4. **Prompt traces** if you need the call stack.
5. **Add a tool** if you need to extend it.
6. Keep the **glossary** open. Config / process-drive as needed.

## Lineage

```
pi (minimal core)
 └── fork → oh-my-pi (maximal built-ins + Rust)

deepseek-harness  (not a fork; optionally uses pi-ai for providers)

opencode          (independent; client/server product)
```
