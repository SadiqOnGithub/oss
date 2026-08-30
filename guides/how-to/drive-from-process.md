# Drive it from another process

When the agent should be a **library or subprocess**, not a TUI you sit in.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

| Harness | Default wire | First-party clients | Prefer this when |
|---|---|---|---|
| **pi** | JSONL stdin/stdout (`--mode rpc`) | TS `RpcClient`; or in-process `AgentSession` | Embedding in Node, or a simple spawn |
| **omp** | JSONL RPC + protocol v2 chunks; also ACP | TS RPC types; **Python `omp-rpc`** | Python host, editors (ACP), robomp-style bots |
| **opencode** | HTTP + SSE (`HttpApi`) | `@opencode-ai/client`, `sdk-next` embedded | Multiple UIs on one agent; IDE/desktop |
| **dsh** | JSON-RPC stdio | TS `@deepseek-ai/dsh-sdk-client`; **Python `deepseek-harness-sdk`**; ACP | Headless automation, Python, subagent-as-runtime |

This is **not** the remote CBOR session stack in pi (`packages/protocol` / `client` / `server`) — that is a different, experimental transport.

Pair with [prompt-traces.md](./prompt-traces.md) (same loop, different I/O) and [add-a-tool.md](./add-a-tool.md) (host-exposed tools).

---

## 1. pi

### In-process (Node/TS) — usually better than RPC

```ts
import { createAgentSession } from "@earendil-works/pi-coding-agent";
// see packages/coding-agent/src/core/agent-session.ts
```

If you already have a Node app, do not spawn `pi --mode rpc` just to call `prompt()`.

### Subprocess RPC

Docs: [packages/coding-agent/docs/rpc.md](../../pi/packages/coding-agent/docs/rpc.md). Client: `packages/coding-agent/src/modes/rpc/rpc-client.ts`. Example: `packages/coding-agent/test/rpc-example.ts`.

```bash
pi --mode rpc --provider anthropic --model sonnet
```

JSONL, LF only. **Do not use Node `readline`** — it splits on U+2028/U+2029, which are legal inside JSON strings.

```json
{"id": "req-1", "type": "prompt", "message": "Hello, world!"}
```

Response `{ "type": "response", "success": true }` means **accepted**, not “the model finished.” Completion is in the event stream.

While streaming, a second prompt **must** set `streamingBehavior`: `"steer"` (inject after this assistant+tools, before next LLM call) or `"followUp"` (wait until idle). Omission → error.

Other useful commands (see the doc): `get_state`, `steer`, bash helpers, session switch. `--no-session` / `--session-dir` for storage. `--name` sets display name.

Auth: same `~/.pi/agent/auth.json` as the TUI unless you pass provider flags/env.

---

## 2. oh-my-pi

### RPC

```bash
omp --mode rpc [same CLI flags as interactive]
```

Docs: [docs/rpc.md](../../oh-my-pi/docs/rpc.md). Implementation: `packages/coding-agent/src/modes/rpc/` (`rpc-mode.ts`, `rpc-client.ts`). `src/jsonrpc/message-framing.ts` is only the length/chunk helper, not the server.

Startup writes a **`ready`** frame **before** commands:

```json
{
  "type": "ready",
  "protocolVersion": 1,
  "supportedProtocolVersions": [1, 2],
  "maxFrameBytes": 1048576,
  "maxReassembledFrameBytes": 67108864
}
```

Clients that support v2 should immediately:

```json
{ "id": "protocol-1", "type": "negotiate_protocol", "protocolVersion": 2 }
```

Then oversized stdout is split into `rpc_chunk` frames (base64) and reassembled. v1 caps a physical frame at 1 MiB.

stdin close → reject pending UI/host-tool requests, drain accepted commands, exit 0.

`@file` CLI args are rejected in RPC mode. Auto session titles are off by default (saves a model call).

### Python: `omp-rpc`

```python
from omp_rpc import RpcClient

with RpcClient(provider="anthropic", model="claude-sonnet-4-5") as client:
    print(client.get_state().model.id if client.get_state().model else "no model")
    turn = client.prompt_and_wait("Reply with just the word hello")
    print(turn.require_assistant_text())
```

Handles v2 negotiation, chunk reassembly, event listeners, host-tools (Python can **expose** tools to the agent with JSON Schema). Source: `oh-my-pi/python/omp-rpc/`.

This is what **robomp** uses: spawn `omp --mode rpc` per GitHub issue worktree.

### ACP

Editor protocol (Zed-class). Separate from JSONL RPC. See coding-agent ACP/bridge tools (`src/tools/acp-bridge.ts`).

### SDK

`createAgentSession` from `@oh-my-pi/pi-coding-agent` (`src/sdk.ts`) — in-process, like pi.

---

## 3. OpenCode

The agent **is** a server. TUI, desktop, VS Code, `opencode run` are clients.

### Local HTTP

```bash
opencode serve          # or bun dev, which starts the same stack
# POST /api/session/:sessionID/prompt
```

Endpoint: `packages/protocol/src/groups/session.ts` → `session.prompt`.

Payload (shape): `{ prompt, delivery?, resume?, id? }`. Success: admitted input row, then the drain runs unless `resume: false`.

Events:

- Per session, replayable: `sessions.events({ sessionID, after })` (SSE)
- Instance-wide, live only: `events.subscribe()`

Do not treat those as interchangeable.

### TypeScript client

```ts
import { createOpencodeClient } from "@opencode-ai/sdk/v2"

const client = createOpencodeClient({ baseUrl: "http://127.0.0.1:4096" }) // serve prefers 4096, then any free port
await client.session.prompt({ sessionID, prompt: { ... } })
```

Generated from the HttpApi — regenerate, don’t edit (`packages/client`). Legacy: `packages/sdk/js`.

`opencode run "…"` in `packages/opencode/src/cli/cmd/run.ts` is this client aimed at a one-shot session.

### Embedded (no HTTP)

`packages/sdk-next`: compose Client + Core + Server in-process. Same API, in-memory transport. Use when you are already in the same JS process and do not want a port.

### ACP / GitHub

`opencode acp` — editor protocol. `github/` Action comments `/opencode fix this` → clone, run, PR. That Action is just another client of the same product.

---

## 4. deepseek-harness

### Python SDK (easiest)

```sh
python -m pip install deepseek-harness-sdk
```

Pulls `deepseek-harness-runtime-bin` (bundled exe). No Web UI.

```python
from deepseek_harness import DeepSeekHarness

with DeepSeekHarness() as harness:
    result = harness.run("Say hi.")
    print(result.final_response)
```

Custom composition (keep the JSON-RPC server plugin in the Cordis file):

```python
with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    cordis="examples/jsonrpc-agent/cordis.yml",
) as harness:
    result = harness.run("Make the requested code change.")
```

`run()` waits from durable inbox receipt to whole-agent **idle**. `final_response` is last root assistant text in that interval — steering can contribute. Low-level `session_prompt()` only returns a queued message id.

Env: `DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `DSH_CORDIS_CONFIG`, `DSH_SESSION_ROOT`. Docs: [python/sdk/README.md](../../deepseek-harness/python/sdk/README.md), [docs/user/guide/python-sdk.md](../../deepseek-harness/docs/user/guide/python-sdk.md).

### TypeScript SDK

`@deepseek-ai/dsh-sdk-client` — **you** name the executable (no bundled-runtime resolver; that’s Python’s job).

```ts
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'

await using harness = new DeepSeekHarness({
  launch: { command: 'node', args: ['lib/bin.js', 'cordis.yml'] },
  provider: 'deepseek-official',
  model: 'deepseek-v4-flash',
})
const result = await harness.run('say hi')
```

Wire: `packages/sdk/{protocol,client,server}`. Server plugin: `dsh-sdk-jsonrpc-server`.

No mid-turn cancel on the wire; abandoning a turn means closing the runtime.

### Headless CLI (not JSON-RPC)

```bash
pnpm dsh --profile headless "Say hi."
```

Fine for scripts; not a programmatic session API. For a live UI, `dsh web` is HTTP to **host**, not the SDK protocol.

### ACP

`packages/acp` — automation-only ACP server. Demo: `packages/examples/acp-demo`.

---

## Choosing a wire

```
Need a Python script quickly?
  omp  → omp-rpc
  dsh  → deepseek-harness-sdk
  pi   → spawn pi --mode rpc (no official Python client)
  opencode → HTTP client (no official Python SDK in-tree)

Already in Node and want in-process?
  pi / omp → createAgentSession
  opencode → sdk-next
  dsh → possible but the product pitch is plugins + spawnable runtime

Multiple frontends on one agent?
  opencode HttpApi (this is the design)
  dsh web host (GUI-shaped, not a general SDK)

Editor?
  ACP: omp, dsh, opencode
```

## Framing pitfalls (copy these, they hurt)

- **pi RPC:** split on `\n` only; not `readline`.
- **omp RPC:** wait for `ready`; negotiate v2 if you can; 1 MiB v1 frame cap.
- **opencode:** `success` on `session.prompt` is **admit**, not idle; watch session SSE.
- **dsh Python `run()`:** idle-bounded interval, not “this prompt’s unique answer.”
- **Auth:** subprocesses inherit env and user home unless you isolate (`PI_CODING_AGENT_DIR`, `DSH_HOME`, scrubbed env on dsh TS client).
