# Add a tool (the same job, four ways)

Goal in every harness: the model can call **`hello`** with `{ name: string }` and get `Hello, <name>!`.

This is the smallest extension that proves you understand the **extension primitive**. After it works, read the real docs linked at the bottom of each section.

You need a working [first hour](./first-hour.md) checkout. Do not send a PR for these snippets unless you mean to.

**Checkout verified:** 2026-08-29 · pi 0.84.2 · omp 18.0.3 · opencode 1.18.21 · dsh 0.1.1-rc.2

---

## What “a tool” is in each repo

| | How you add it | Loaded from | Same registry as builtins? |
|---|---|---|---|
| **pi** | `pi.registerTool(defineTool(…))` inside an extension | `~/.pi/agent/extensions/` or `.pi/extensions/` or `-e` | Yes |
| **omp** | Custom tool factory **or** `pi.registerTool` in an extension | `~/.omp/agent/tools`, `.omp/tools`, plus Claude/Codex dirs | Yes (name conflicts rejected) |
| **opencode** | Plugin `Hooks.tool` via `@opencode-ai/plugin` | `opencode.json` `plugin` array / `.opencode/` | Yes, then permission-checked |
| **dsh** | Cordis plugin `ctx.tools.register(defineTool(…))` | profile bundle / `cordis.patch.yml` / `--patch` | Yes — tools are just another plugin |

pi forbids building this into core. omp and opencode will take a first-party tool if it belongs in the product. dsh *expects* you to add a package (or a scratch overlay).

---

## 1. pi — extension file

Copy of the official example: `pi/packages/coding-agent/examples/extensions/hello.ts`.

```ts
import { Type } from "@earendil-works/pi-ai";
import { defineTool, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

const helloTool = defineTool({
  name: "hello",
  label: "Hello",
  description: "A simple greeting tool. Use it when the user asks to be greeted.",
  parameters: Type.Object({
    name: Type.String({ description: "Name to greet" }),
  }),
  async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },
    };
  },
});

export default function (pi: ExtensionAPI) {
  pi.registerTool(helloTool);
}
```

**Load it**

```bash
# auto-discovery (hot-reload with /reload in TUI)
cp hello.ts ~/.pi/agent/extensions/
# or project:
cp hello.ts .pi/extensions/

# one-off
./pi-test.sh -e ./hello.ts -p "Greet Ada using the hello tool"
```

**Verify:** session should contain a `hello` tool call with `Hello, Ada!`.

**Next in this repo:** `examples/extensions/todo.ts` (state + command + renderer), `permission-gate.ts` (intercept bash), [docs/extensions.md](../../pi/packages/coding-agent/docs/extensions.md). Core-bloat rule: this does **not** belong in `packages/coding-agent/src/core/tools/`.

---

## 2. omp — custom tool module

Custom tools are factories, not a default-export extension (though extensions can `registerTool` too). Schema is Zod via `pi.zod`.

Put `hello.ts` in `.omp/tools/` or `~/.omp/agent/tools/`:

```ts
import type { CustomToolFactory } from "@oh-my-pi/pi-coding-agent";

const factory: CustomToolFactory = (pi) => ({
  name: "hello",
  label: "Hello",
  description: "A simple greeting tool. Use it when the user asks to be greeted.",
  parameters: pi.zod.object({
    name: pi.zod.string().describe("Name to greet"),
  }),
  async execute(_toolCallId, params, onUpdate, _ctx, _signal) {
    onUpdate?.({
      content: [{ type: "text", text: "Greeting…" }],
      details: { phase: "start" },
    });
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },
    };
  },
});

export default factory;
```

**Do not** reuse a name in `BUILTIN_TOOL_NAMES` (`packages/coding-agent/src/tools/builtin-names.ts`) — loader rejects collisions.

**Verify:** `bun src/cli.ts -p "Greet Ada using the hello tool"` from `packages/coding-agent`, cwd = the project that contains `.omp/tools/hello.ts`.

**Next:** [docs/custom-tools.md](../../oh-my-pi/docs/custom-tools.md), [docs/extensions.md](../../oh-my-pi/docs/extensions.md). Discovery also reads `~/.claude/tools` and `~/.codex/tools`.

---

## 3. OpenCode — plugin

Official shape: `opencode/packages/plugin/src/example.ts`.

```ts
import type { Plugin } from "@opencode-ai/plugin"
import { tool } from "@opencode-ai/plugin"

export const HelloPlugin: Plugin = async (_ctx) => {
  return {
    tool: {
      hello: tool({
        description: "A simple greeting tool. Use it when the user asks to be greeted.",
        args: {
          name: tool.schema.string().describe("Name to greet"),
        },
        async execute(args, _context) {
          return `Hello ${args.name}!`
        },
      }),
    },
  }
}
```

Point config at it (`opencode.json` / `opencode.jsonc` in the project or `~/.config/opencode/`):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["./hello-plugin.ts"]
}
```

The plugin function receives `PluginInput` (`client`, `directory`, `worktree`, `$` Bun shell, …) if you need filesystem or the HttpApi.

**Permissions:** the `plan` agent is read-only. Test with the `build` agent (Tab in TUI). Skills that are tools go through `packages/core/src/tool/skill.ts` instead.

**Verify:** `bun dev run "Greet Ada using the hello tool"` from a directory that has the config.

**Next:** `packages/plugin/`, `packages/core/src/plugin/`, never hand-edit `packages/client/src/generated*`.

---

## 4. dsh — Cordis plugin overlay

Tools are consumers of `ctx.tools`. Official hello-plugin tutorial: [docs/user/develop/basic/index.md](../../deepseek-harness/docs/user/develop/basic/index.md). A tool needs `inject: ['tools']`.

`scratch-plugin/src/hello-tool.ts` (product tools declare a JSON **output** DTO plus `render`; they do not return free-form `{ content }` — see `packages/todo/tool-todo` and `packages/skill/tool-skill`):

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'hello-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(
    defineTool({
      name: 'hello',
      description: 'A simple greeting tool. Use it when the user asks to be greeted.',
      parameters: {
        name: {
          type: 'string',
          required: true,
          description: 'Name to greet',
        },
      },
      output: {
        schema: {
          type: 'object',
          additionalProperties: false,
          properties: {
            greeting: { type: 'string', required: true },
          },
        },
        render: (_args, value) => [{ type: 'text', text: value.greeting }],
      },
      async execute(args) {
        return { greeting: `Hello, ${args.name}!` }
      },
      presentCall: (args) => ({
        card: 'generic',
        title: `Greet ${args.name}`,
        kind: 'other',
        rawInput: args.name,
      }),
    }),
  )
}
```

`scratch-plugin/cordis.yml` (path **must be absolute**):

```yaml
- insert:
    - id: hello-tool
      name: '/ABS/PATH/deepseek-harness/scratch-plugin/src/hello-tool.ts'
```

```bash
pnpm dsh web --patch ./scratch-plugin/cordis.yml
# or
pnpm dsh --profile headless --patch ./scratch-plugin/cordis.yml "Greet Ada using the hello tool"
```

Unload = dispose: anything registered on `ctx` is reversed. Use `ctx.effect(() => () => cleanup)` for timers/sockets.

**A real in-tree tool to copy:** `packages/todo/tool-todo/src/index.ts` (`todo_write` via `defineTool` + `ctx.tools.register`). Shipping it “for real” means a new `packages/<group>/<pkg>/` and the [adding-a-package cookbook](../../deepseek-harness/docs/cookbook/adding-a-package.md).

---

## After it works

1. Trace the call with [prompt-traces.md](./prompt-traces.md) — your `execute` sits where `bash.ts` / `tool-bash` sat.
2. Decide if this should be a **skill** (markdown guidance) instead. pi treats skills as the MCP replacement; omp/dsh/opencode have both.
3. For process hosts that want to *expose* a tool from Python/Node rather than ship one inside the agent, see [drive-from-process.md](./drive-from-process.md) (omp host-tools, dsh SDK).
