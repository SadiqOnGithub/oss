# One prompt, four traces

Same user message in all four harnesses:

> List the markdown files in this directory.

No new features — just the call stack you can click. Pair with [first-hour.md](./first-hour.md) and the per-repo file guides.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

Delivery modes used here: pi/omp **print** (`-p`), opencode **`run`**, dsh **headless**. TUI/Web wrap the same loop with extra I/O.

---

## Shared shape (so the four traces rhyme)

```
input admitted
  → session/history updated
  → system prompt assembled
  → provider stream (tool calls)
  → tool execute
  → tool result back into history
  → stream again until the model stops
  → persist
```

They disagree on **where “admitted” is durable**, **which process owns the loop**, and **whether tools are a package or a plugin**.

---

## 1. pi

**Command:** `./pi-test.sh -p "List the markdown files in this directory."`

```
argv
  packages/coding-agent/src/cli.ts
    process.title, HTTP dispatcher
  packages/coding-agent/src/main.ts
    parseArgs (cli/args.ts) → createAgentSession()
  packages/coding-agent/src/core/agent-session-runtime.ts
  packages/coding-agent/src/core/agent-session.ts
    AgentSession.prompt(text)
      this.agent.prompt(messages)
  packages/agent/src/agent.ts
    Agent.prompt()
  packages/agent/src/agent-loop.ts
    agentLoop(prompts, context, config, signal, streamFn)
      runAgentLoop:
        convert messages → LLM Message[]
        streamFn → packages/ai/src/providers/<provider>
        on tool_call → AgentTool.execute
  packages/coding-agent/src/core/tools/
    typical calls for this prompt: ls / grep / find factories exist,
    CLI harness mounts read + bash + edit + write
    (the model usually shells `ls *.md` via bash.ts)
  packages/coding-agent/src/core/session-manager.ts
    append JSONL under ~/.pi/agent/sessions/<encoded-cwd>/
  packages/coding-agent/src/modes/print-mode.ts
    print assistant text, exit
```

**Events:** `pi-agent-core` emits an `EventStream<AgentEvent>`. Print mode subscribes and writes. RPC mode serializes the same events as JSONL ([drive-from-process.md](./drive-from-process.md)).

**What is *not* durable:** the in-flight loop identity. Crash mid-tool: session history is on disk; there is no opencode-style admitted-input queue.

**Steer vs follow-up:** `AgentSession.prompt` options; in RPC, `streamingBehavior: "steer" | "followUp"` (see `docs/rpc.md`).

---

## 2. oh-my-pi (omp)

**Command:** `bun src/cli.ts -p "List the markdown files in this directory."`  
(from `packages/coding-agent/`)

Same ancestor as pi, then a much thicker tool/native layer.

```
argv
  packages/coding-agent/src/cli.ts
    worker-host dispatch, Bun version guard
    default subcommand → launch
  packages/coding-agent/src/main.ts
    theme / settings / model registry / session opts
    createAgentSession(...)            src/sdk.ts
  packages/coding-agent/src/session/   (AgentSession, JSONL tree)
  packages/agent/                      (fork of pi-agent-core; agent-loop.ts)
  packages/ai + packages/catalog       (providers + bundled model DB)
  packages/coding-agent/src/tools/
    glob.ts / grep.ts / bash.ts
    bash does NOT fork a user shell for builtins:
      crates/pi-shell + crates/pi-builtins  (in-process)
    grep/glob hot path:
      packages/natives → crates/pi-natives
  print path:
    src/modes/  runPrintMode
```

**Persistence:** JSONL session tree (`docs/session.md`, `docs/session-tree-plan.md`), not only SQLite. Memories can also hit `packages/mnemopi`.

**Extra hops this prompt might take:** `glob` tool instead of bash; advisor model (`src/advisor/`) watching the turn; snapcompact if context is huge — skip those until the basic loop is solid.

---

## 3. OpenCode

**Command:** `bun dev run "List the markdown files in this directory."`  
(from `opencode/`)

The loop does **not** live in the TUI. The CLI is a client of the HttpApi.

```
packages/opencode/src/index.ts          yargs
packages/opencode/src/cli/cmd/run.ts
  createOpencodeClient from @opencode-ai/sdk/v2
  client.session.prompt({ ... })

HTTP (or in-process embedded transport)
  POST /api/session/:sessionID/prompt
  packages/protocol/src/groups/session.ts   session.prompt
  packages/server/src/handlers/session.ts

packages/core/src/session/input.ts
  SessionInput.admit(...)
    INSERT session_input row          durable *before* the model runs
    delivery: steer | queue | …
  schedule drain unless resume: false

packages/core/src/session/execution.ts
  process-local Session Drain (serialized per session)
packages/core/src/session/runner/
  llm.ts  →  packages/llm  →  AI SDK / models.dev
packages/core/src/system-context/
  Context Epoch baseline (cache-friendly, immutable)
packages/core/src/tool/
  glob.ts / grep.ts / bash.ts / read.ts
  permission.ts            build vs plan agent
packages/core/src/session/event.ts
  SSE: sessions.events({ sessionID, after })   replayable
       vs events.subscribe()                    live-only
```

**The important difference:** the prompt is a **database row** (`session_input`) first. If the process dies after admit and before the provider returns, history is still there and a later drain can continue. That is the V2 “prompt admission decoupled from execution” design in `specs/v2/`.

**TUI path:** `packages/tui` → same `session.prompt`. Desktop/VS Code too.

---

## 4. deepseek-harness

**Command:** `pnpm dsh --profile headless "List the markdown files in this directory."`

There is no single `agentLoop.ts` you *must* hit — the loop is a **plugin** (`ctx.agentLoop`). Default implementation:

```
apps/cli/src/bin.ts
  parseDshArgs → mode 'profile'
apps/cli/src/profile-boot.ts
  loadLayeredEnv
  compose:
    @deepseek-ai/dsh-base
    @deepseek-ai/dsh-headless
    profile cordis.patch.yml
    $DSH_HOME/cordis.patch.yml
    --patch
  Cordis Loader (vendor/loader + vendor/cordis)

headless runner claims the argv string as the first user message

packages/core/session          ctx.sessions   append-only SessionEvent log
packages/core/system-prompt    ctx.systemPrompt
packages/core/agent-loop       ctx.agentLoop
  turn/start
  agent/pre-step          waterfall (rewrite | reject | enter)
  step/start
  agent/request → packages/llm  (llm-deepseek or llm-pi-ai)
  tool/call
    packages/core/tools        ctx.tools
      pre-execute → execute → post-execute
    this prompt → packages/fs/tool-fs-search  and/or  packages/shell/tool-bash
  tool/result  (oversized text → packages/spill)
  step/end → maybe another request
  agent/turn-stopping

packages/session/session-persistence-*    JSONL and/or SQLite
```

**Invariant:** model-visible ⇔ logged. If it was in the prompt the model saw, it is a session event.

**Web path:** same loop; `packages/host` + `packages/client` + `apps/web` are I/O. Approvals go through `packages/interaction`.

---

## Where to put a breakpoint

| Question | pi | omp | opencode | dsh |
|---|---|---|---|---|
| Prompt just accepted | `AgentSession.prompt` | `src/sdk.ts` / session AgentSession | `SessionInput.admit` | session log append + `turn/start` |
| About to call the vendor | `packages/ai` streamFn | same + `catalog` | `packages/llm` + `runner/llm.ts` | `ctx.llm` / `llm/stream` |
| Tool about to run | `core/tools/*.ts` | `src/tools/*.ts` | `core/src/tool/*.ts` | `ctx.tools` execute waterfall |
| Result about to persist | `session-manager.ts` JSONL | `src/session/` JSONL | drizzle `SessionMessageTable` | `session-persistence-*` |
| UI just painting | `modes/print-mode.ts` or tui | `src/modes/` | TUI is a **client** of SSE | React `packages/client/ui-*` |

Next: make the model call *your* function — [add-a-tool.md](./add-a-tool.md).
