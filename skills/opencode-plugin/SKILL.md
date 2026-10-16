---
name: opencode-plugin
description: "CRITICAL: Load when building, modifying, updating or debugging OpenCode plugins. Missing this = silent failures, broken hooks, and wasted hours. Covers V1 (legacy) vs V2 generation selection, server vs TUI plugin types, domain hooks, Context, keymap/theme/dialog APIs, discovery, config, SolidJS UI slots, and deployment. Works for ANY plugin type."
---

## When to use me

- Building a new OpenCode plugin (server or TUI) — **first pick the generation (v1 vs v2) below**
- Debugging an existing plugin that doesn't load or behave correctly
- Understanding the OpenCode plugin API (v2 `Context`, or the legacy v1 `TuiPluginApi`)
- Working with OpenTUI SolidJS components in a TUI plugin
- Configuring plugin registration (`cli.json` / `opencode.json` for v2; `tui.json` for v1)

## Not intended for

- General SolidJS development outside OpenCode
- OpenCode SDK client usage (standalone scripts) → use SDK docs directly
- OpenCode configuration (providers, models, themes) → use `/config`
- Dialog spacing/layout/interaction rules → use the `opencode-dialogs` skill

---

## Step 0 — Choose the generation, then the plugin type (READ FIRST)

OpenCode has **two independent axes**: the **generation** (v1 vs v2) selects the host
API, and within either generation a module is either a **server plugin** or a
**TUI plugin**. Pick the generation first.

### V1 vs V2 generation (READ FIRST)

v1 and v2 can be installed side by side (`opencode` vs `opencode2`) but a plugin
**cannot be shared**: a v1 plugin does not work in v2 and vice-versa. Pick one
generation per module and **never mix the two host APIs in one runtime adapter**.

| Concern | V1 (legacy) | V2 (current) |
|---|---|---|
| Executable | `opencode` | `opencode2` |
| Package | `@opencode-ai/plugin`, `@opencode-ai/plugin/tui` | `@opencode/plugin`, `@opencode/plugin/tui` (+ `/effect`, `/host`, `/runtime`) |
| Module shape | server: `export const X = async () => ({ hook map })`; TUI: `export default { id, tui }` | `Plugin.define({ id, setup })` (server **and** TUI) |
| Server hooks | flat dotted hook map (`"experimental.chat.system.transform"`, `"tool.definition"`, …) | **domain methods**: `ctx.session.hook("context" \| "prompt" \| "compaction" \| …)`, `ctx.tool.hook(...)`, `ctx.event.subscribe(...)` |
| TUI API | `TuiPluginApi` function argument | `Plugin.Context` obtained in `setup(context)` / `usePlugin()` |
| TUI config | `tui.json` → `{ "plugin": ["./plugins/x.tsx"] }` | `cli.json` → `{ "$schema": "https://opencode.ai/v2/cli.json", "plugins": [...] }` (**TUI-only**) |
| Plugin config list | `opencode.json` (server list) | `opencode.json(c)` `plugins` — **core/server package list** (**not** a TUI list; TUI-only is `cli.json`) |
| Keymap | `api.keymap.registerLayer({ commands, bindings })` | `ctx.keymap.layer(() => KeymapLayer)` |
| Command fields | `name`, `category`, `namespace:"palette"`, `slashName` | `id`, `group`, `palette:true`, `slash:{ name, aliases?, arguments? }`, `bind` |
| Theme | `api.theme.current` (drifting prop names) | `ctx.theme` (`ResolvedTheme`, semantic tokens) |
| Dialog | `api.ui.dialog.replace/setSize/clear` | `ctx.ui.dialog.show(render, onClose?) / set({ size, centered }) / clear()` |
| Slots | `api.slots.register({ order, slots })` | `ctx.ui.slot(claim)` with `append/prepend/before/after/replace` on a `SlotPath` |

**Config lists (v2) — two lists, two consumers.**
- `opencode.json(c)` `plugins` is consumed by **core/server**. Every entry names a
  **server-capable package**; a **local target must be a directory**, not a loose file.
  A package **directory may expose separate entrypoints** — a server entry
  (`server.ts` / `index.ts`), a TUI entry (`tui.tsx`), and an rpc entry (`rpc.ts`). Core
  advertises **`features.tui`**, and when it does the TUI loads that **server-provided
  TUI entry** (`tui.tsx`). So an `opencode.json(c)` entry does **not** itself directly
  register a TUI-only plugin — the TUI side is delivered through core's advertised
  `features.tui`.
- `cli.json` `plugins` is a **separate, explicit TUI-only list**. It resolves **only**
  the `tui` entrypoint, so it is how a **TUI-only package** (one with no server entry)
  is loaded.
- **Auto-discovery** is independent of both lists: a plugin **directory** under
  `plugins/` is picked up with no config entry at all.

(`opencode.json` remains the v1 server list.)

### Server vs TUI plugin type (within a generation)

OpenCode has **two separate, mutually exclusive plugin systems**:

| | Server Plugin | TUI Plugin |
|---|---|---|
| **Purpose** | Hook into events, shell env, tools | UI in sidebar, commands, routes |
| **File ext** | `.js` or `.ts` | `.tsx` (JSX required) |
| **Export** | `export const MyPlugin = async ({ $, client, ... }) => { return hooks }` | `export default { id, tui }` |
| **Auto-loaded** | ✅ from `.opencode/plugins/` or `~/.config/opencode/plugins/` | ❌ must register in `tui.json` |
| **Config file** | `opencode.json` (optional, npm plugins) | `tui.json` (required) |
| **Types** | `Plugin` from `@opencode-ai/plugin` | `TuiPlugin`, `TuiPluginModule` from `@opencode-ai/plugin/tui` |

Within a **single v1 module**, a module can only be one type — it is a server plugin
**or** a TUI plugin, never both; the v1 `TuiPluginModule` type encodes this as
`server?: never`. This is a **v1-only** constraint. A **v2 plugin directory/package**
may expose **separate entrypoints** — a server entry (`server.ts`, or `index.ts` as the
fallback), a TUI entry (`tui.tsx`), and an rpc entry (`rpc.ts`) — because each entrypoint
is a separate module, not a single module that is "both types".

### V2 confirmed facts (OpenCode 2.0.27)

**Server/TUI entrypoints & hooks.**
- Server: `import { Plugin } from "@opencode/plugin"` → `export default Plugin.define({ id, setup })`.
  `ctx.session.hook("context")` carries `event.system: SystemPart[]` **and**
  `event.tools: Record<name,{description,input}>` together. Other hook kinds:
  `"prompt" | "compaction" | "generate" | "title" | "model.request" | "http.request" | "http.response" | "retry"`.
- TUI: `import { Plugin } from "@opencode/plugin/tui"` → `Plugin.define({ id, setup })`;
  `setup(context)` receives the v2 `Context` (also reachable via `usePlugin()` inside a
  component). The v2 definition is just `{ id: string, setup: (context) => cleanup | void }`.
- A single **entrypoint module** is either a server or a TUI plugin — never both — but a
  **v2 plugin directory/package** may expose **separate entrypoints**: a server entry
  (`server.ts`, or `index.ts` as the fallback), a TUI entry (`tui.tsx`), and an rpc entry
  (`rpc.ts`). The v1 `TuiPluginModule.server?: never` constraint does **not** apply to v2,
  and core advertises **`features.tui`** so the TUI can load the server-provided TUI entry.

**Discovery: core (server) vs TUI.**
- **Core/server discovery** scans the immediate children of **both** `<configDir>/plugin/`
  **and** `<configDir>/plugins/`, admitting **bare `.ts`/`.js` files AND directories**.
  A **bare `.tsx` file is NOT auto-discovered** by core — only `.ts`/`.js` files and
  directories are, so a loose `.tsx` is silently ignored (neither loaded nor an error).
- **TUI auto-discovery** scans **directories under the plural `<configDir>/plugins/`**
  only (symlinks-to-directories count) — **never bare files**, `.tsx` included.
- **Entrypoint resolution** (`Host.resolve`): server entry = `server` first, then `index`;
  TUI entry = `tui`; rpc = `rpc`.
- **Local layout:** one directory per plugin under the plural `plugins/`, e.g.
  `~/.config/opencode/plugins/<name>/index.ts` (or `server.ts`) + `tui.tsx`.
  A directory is auto-discovered; an explicit local **file** target is rejected.
- ⚠️ **Avoid v1 loose files in a v2 config root.** Core discovery admits bare `.ts`/`.js`
  children, so a v1 flat file such as `model-usage-server.ts` in a v2 `plugins/` is
  auto-discovered and fails to load. A loose v1 **`.tsx`** is not auto-discovered, so it
  neither loads nor errors. Keep v1 and v2 in **separate config roots** to avoid both
  the load failure and the silently-ignored file.

**Keymap layer mode.**
- `KeymapLayer.mode` omitted ⇒ `base`. While a dialog is open the active mode is
  pushed to `modal`, so a layer left at the `base` default is **inactive exactly when
  the dialog is visible**.
- Plugin/command and dialog keymap layers should set `mode: "global"` — v2 has no
  `api.keymap.registerLayer`; use `ctx.keymap.layer(() => ({ mode: "global", commands }))`.

**Theme.**
- `context.theme` is **resolved**: action states drop the `$` sigil — `base`,
  `focused`, `selected`, `hovered`, `pressed`, `disabled` (see the host's `resolvedKey`).
- **Raw theme definitions** (authored themes) use `$focused` / `$selected`.
- Theme colors are `RGBA` objects: **pass values through, never string-coerce**.

**Dialogs & Solid.**
- `context.ui.dialog.show(render, onClose?)`, `.set({ size, centered })`, `.clear()`.
  Dialog **render callbacks can re-run** whenever a signal read in the body changes.
- Long-lived controllers must be created through a stable `createOwnedInstance` owner,
  lazily `.get()`-ed **inside the render callback** (so the owner inherits the dialog's
  `Keymap.Provider` context and survives re-runs), and `.dispose()`-ed in `onClose`.
  See the `opencode-dialogs` skill for the full pattern.
- **`.tsx` vs `.ts` — the OpenCode local plugin-loader transform boundary (2.0.27).**
  OpenCode 2.0.27's local plugin loader applies its Solid runtime transform to
  **`.tsx`/`.jsx`** plugin files. A **plugin-local module** that relies on that host
  rewrite must therefore be named `.tsx`: one that **returns JSX**, **calls
  Solid/OpenTUI runtime APIs** (`createRoot`, `createEffect`, signals, components),
  or **has a runtime `solid-js` / `@opentui/solid` import whose code depends on the
  host's JSX/Solid transform**. A module that is **pure `.ts` helpers** — or whose only
  `solid-js` import is **type-only** (`import type { JSX } from "solid-js"`) — stays
  **`.ts`**. This is a boundary of the OpenCode plugin loader, **not** a universal rule
  that every `solid-js` runtime import requires `.tsx`: a `.ts` module is fine when its
  Solid code is transformed elsewhere (e.g. a prebuilt/bundled dependency). Code that
  does need the host transform silently fails to transform if kept in a `.ts` file.

> **Legacy v1 note:** the remainder of this skill documents the **v1** API.
> Anything using `TuiPluginApi`, `api.state`, `api.event`, `api.keymap.registerLayer`,
> `api.theme.current`, `api.ui.dialog.replace/setSize`, `TuiPlugin` / `TuiPluginModule`,
> `tui.json`, or `@opencode-ai/plugin` is **v1-only** and must not be used in a v2 module.

---

# ⚠️ V1 LEGACY REFERENCE (deprecated)

Everything from **Step 1** onward documents the **v1** plugin API. Use it only to
maintain a deprecated v1 adapter; new / v2 work follows Step 0 and the
`opencode-dialogs` skill. Where the v2 equivalents are known they are cross-referenced;
v1-only identifiers are tagged **[v1-only]**.

---

## Step 1 — [v1-only] Server plugin structure

> v2 equivalent: `Plugin.define({ id, setup })` + domain hooks (`ctx.session.hook("context")` …). See Step 0.

```js
// .opencode/plugins/my-server-plugin.js
export const MyPlugin = async ({ project, client, $, directory, worktree }) => {
  // Initialization code runs once at plugin load

  return {
    // Event hooks
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        // React to session becoming idle
      }
    },

    // Tool interception
    "tool.execute.before": async (input, output) => {
      // input.tool = tool name, output.args = tool arguments
    },
    "tool.execute.after": async (input, output) => {
      // output.title, output.output, output.metadata
    },

    // Shell environment injection
    "shell.env": async (input, output) => {
      output.env.MY_VAR = "value"
    },

    // Custom tools
    tool: {
      mytool: tool({
        description: "Does something",
        args: { foo: tool.schema.string() },
        async execute(args, context) {
          return `Result: ${args.foo}`
        },
      }),
    },

    // Session compaction customization
    "experimental.session.compacting": async (input, output) => {
      output.context.push("## Custom context...")
    },
  }
}
```

### Available server events

```
session.created, session.updated, session.deleted, session.idle,
session.error, session.compacted, session.status, session.diff
permission.asked, permission.replied
message.updated, message.removed, message.part.updated, message.part.removed
tool.execute.before, tool.execute.after
file.edited, file.watcher.updated
shell.env, command.executed
server.connected, installation.updated
lsp.client.diagnostics, lsp.updated
todo.updated
```

### Server plugin context fields

| Field | Description |
|---|---|
| `project` | Current project information |
| `directory` | Current working directory |
| `worktree` | Git worktree path |
| `client` | OpenCode SDK client for API calls |
| `$` | Bun's shell API for executing commands |

### Config file loading (server plugins)

Use `new URL("./config.json", import.meta.url).pathname` to resolve paths relative to the plugin file. Use `Bun.file()` for reading.

```js
const configPath = new URL("./my-plugin.config.json", import.meta.url).pathname
const configFile = Bun.file(configPath)
const config = (await configFile.exists())
  ? JSON.parse(await configFile.text())
  : defaultConfig
```

---

## Step 2 — [v1-only] TUI plugin structure

> v2 equivalent: `Plugin.define({ id, setup })` with a `Plugin.Context` (`context.ui.slot`, `context.keymap.layer`, `context.ui.dialog`). See Step 0.

```tsx
// .opencode/plugins/my-tui-plugin.tsx
/** @jsxImportSource @opentui/solid */
import type { TuiPlugin, TuiPluginModule, TuiPluginApi } from "@opencode-ai/plugin/tui"
import { createSignal, createEffect, createMemo, onCleanup } from "solid-js"

function MyComponent(props: { api: TuiPluginApi; session_id: string }) {
  // SolidJS component with reactive state
  return <box flexDirection="column" gap={0}>
    <text fg="#ffffff">Hello from plugin!</text>
  </box>
}

const tui: TuiPlugin = async (api) => {
  api.slots.register({
    order: 175, // position in sidebar
    slots: {
      sidebar_content(_ctx, props) {
        return <MyComponent api={api} session_id={props.session_id} />
      },
    },
  })
}

const plugin: TuiPluginModule & { id: string } = {
  id: "my-tui-plugin",
  tui,
}

export default plugin
```

### TUI plugin registration in `tui.json` **[v1-only]**

> v2 replaces `tui.json` with `cli.json` (`{ "$schema": "https://opencode.ai/v2/cli.json", "plugins": [...] }`)
> for TUI plugins; a v2 plugin **directory** under `plugins/` is auto-discovered and
> needs no entry. Never point a v2 config at a loose `.tsx` file. See Step 0.

**Critical**: in v1, TUI plugins MUST be registered in `tui.json`. They are NOT auto-loaded.

**Location**: `~/.config/opencode/tui.json` (global) or `.opencode/tui.json` (project).

**Path resolution**: Relative paths resolve relative to the `tui.json` file location. Use `./` prefix:

```json
{
  "$schema": "https://opencode.ai/tui.json",
  "plugin": ["./plugins/my-tui-plugin.tsx"]
}
```

The opencode repo itself uses this pattern: `./plugins/tui-smoke.tsx` in `.opencode/tui.json`.

### Sidebar slots

| Slot | Mode | Props | Notes |
|---|---|---|---|
| `sidebar_title` | `single_winner` | `{ session_id, title, share_url? }` | Replaces title area |
| `sidebar_content` | additive (default) | `{ session_id }` | Main content, ordered by `order` |
| `sidebar_footer` | `single_winner` | `{ session_id }` | Replaces version info |

**Built-in order values**:

| Plugin | Order |
|---|---|
| `internal:sidebar-context` | 100 |
| `internal:sidebar-mcp` | 200 |
| `internal:sidebar-lsp` | 300 |
| `internal:sidebar-todo` | 400 |
| `internal:sidebar-files` | 500 |

Use `175` to insert between context and MCP. Use `> 500` to appear below all built-ins.

---

## Step 3 — [v1-only] `TuiPluginApi` reference

> v2 equivalent: the `Plugin.Context` surface (`ctx.data`, `ctx.client`, `ctx.keymap`, `ctx.ui`, `ctx.theme`, `ctx.storage`). See Step 0.

| Field | Description |
|---|---|
| `api.state` | Reactive SolidJS-backed state (sessions, messages, parts, providers, config) |
| `api.state.config` | `SdkConfig` — has `model?: string` in `"provider/model"` format |
| `api.state.provider` | `ReadonlyArray<Provider>` — each has `{ id, name, source, env, models }` |
| `api.state.session.messages(sessionID)` | `ReadonlyArray<Message>` — **capped at 100** |
| `api.state.part(messageID)` | `ReadonlyArray<Part>` for a message |
| `api.client` | Full SDK client — use for data beyond 100-msg cap |
| `api.event.on(type, handler)` | Subscribe to events; returns unsubscribe fn |
| `api.kv` | Persistent key-value store (`get`/`set`/`ready`) |
| `api.lifecycle.onDispose(fn)` | Plugin-level cleanup; returns unregister fn |
| `api.slots.register(plugin)` | Register UI slots; returns slot id |
| `api.ui` | `toast()`, `dialog`, `Dialog`, `DialogSelect`, `Slot`, `Prompt` |
| `api.theme` **[v1-only]** | Current theme colors (`foreground`, `muted`, etc.). v2 uses `context.theme` (`ResolvedTheme`) |
| `api.command.register(cb)` | Register slash commands; returns unregister fn |
| `api.route` | Register routes, navigate |

### Reading the active model/provider

```tsx
// Global default model (format: "providerID/modelID")
const model = api.state.config?.model // e.g. "github-copilot/claude-sonnet-4"

// Available providers
const providers = api.state.provider // ReadonlyArray<Provider>

// ✅ PREFERRED: Read model directly from UserMessage.model — it is a REQUIRED field in the
// OpenCode schema (never optional). Works for ALL user messages including compaction ones.
const messages = api.state.session.messages(sessionID) // NOTE: capped at 100
const userMsg = messages.find(m => m.role === "user")
if (userMsg) {
  const model = (userMsg as any).model as { providerID: string; modelID: string; variant?: string }
  // model.providerID e.g. "github-copilot", model.modelID e.g. "claude-sonnet-4"
}

// In a message.updated handler — read from the event payload directly:
api.event.on("message.updated", (event) => {
  const msg = event.properties.info
  if (msg.role !== "user") return
  const userModel = (msg as any).model as { providerID: string; modelID: string } | undefined
  // userModel?.providerID, userModel?.modelID — always set for user messages
})

// Fallback only: last assistant message's model (use only when UserMessage.model unavailable)
const lastAssistant = [...messages].reverse().find(m => m.role === "assistant")
if (lastAssistant) {
  // lastAssistant.providerID, lastAssistant.modelID (top-level on AssistantMessage, not nested)
}
```

### Fetching messages beyond the 100 cap

```tsx
const result = await api.client.session.messages({
  sessionID: props.session_id,
  limit: 10000,
})
// result.data: Array<{ info: Message, parts: Part[] }>
```

---

## Step 4 — SolidJS patterns for TUI plugins

> **Applies to v1 and v2.** The SolidJS rules below are generation-agnostic; only the
> host API differs. In v2 read events/state from `ctx.data` (`ctx.data.on`, `ctx.data.session…`)
> instead of `api.event` / `api.state`.

### Component lifecycle

```tsx
function MyComponent(props: { api: TuiPluginApi; session_id: string }) {
  const [data, setData] = createSignal(null)

  // Re-runs when session_id changes (reactive)
  createEffect(() => {
    const sessionID = props.session_id // MUST read synchronously for tracking
    fetchData(sessionID).then(setData) // Use .then(), NOT async/await

    const unsub = props.api.event.on("session.compacted", handler)
    onCleanup(unsub) // Cleanup on re-run or unmount
  })

  return <text>{data() ?? "Loading..."}</text>
}
```

### Critical: NEVER use async/await inside createEffect

Reactive tracking stops at the first `await`. Capture values synchronously, then use `.then()`:

```tsx
// WRONG — breaks reactivity
createEffect(async () => {
  const id = props.session_id
  const data = await fetchData(id) // ❌ reactivity lost after await
  setData(data)
})

// CORRECT — preserves reactivity
createEffect(() => {
  const id = props.session_id // tracked
  fetchData(id).then(setData) // ✅ closure captures id
})
```

### Component-level cleanup (NOT api.lifecycle)

Inside SolidJS components, use `onCleanup` — NOT `api.lifecycle.onDispose`:

```tsx
createEffect(() => {
  const unsub = props.api.event.on("message.updated", handler)
  onCleanup(unsub) // runs when effect re-runs or component unmounts
})
```

`api.lifecycle.onDispose` is for plugin-level cleanup only (in the `tui` function scope).

### Polling fallback for first-load race condition

If `api.state` isn't populated on first render, use a polling interval as fallback:

```tsx
const modelPoller = setInterval(() => {
  const detected = detectModel(props.api)
  if (detected !== currentModel()) {
    setCurrentModel(detected)
  }
}, 2000)

onCleanup(() => clearInterval(modelPoller))
```

### Never return null from the root component

Returning `null` causes the component to unmount. Use conditional rendering inside a persistent container:

```tsx
// WRONG — unmounts the component
if (!isActive()) return null
return <box>...</box>

// CORRECT — keeps component mounted
return (
  <box flexDirection="column" gap={0}>
    {isActive() ? <box>...</box> : null}
  </box>
)
```

---

## Step 5 — OpenTUI components

### Layout

- `<box>` — Container with `flexDirection`, `gap`, `border`, `padding`, `backgroundColor`
- `<scrollbox>` — Scrollable container
- `<text>` — Styled text with `fg`, `bg`, `content` props

### Text modifiers (inside `<text>`)

- `<strong>`, `<b>` — Bold
- `<em>`, `<i>` — Italic
- `<span>` — Inline styled text
- `<br>` — Line break

### Colors

Use hex strings: `"#ffffff"`, `"#22c55e"`, `"#eab308"`, `"#ef4444"`, `"#888888"`

Access theme colors via `api.theme.current` **[v1-only]** — v2 passes `context.theme`
(a resolved `ResolvedTheme`) and must **not** string-coerce the `RGBA` values:

```tsx
// v1-only
<text fg={api.theme.current?.foreground ?? "#ffffff"}>Text</text>
<text fg={api.theme.current?.muted ?? "#888888"}>Dim text</text>
```

---

## Step 6 — Debugging

### Logging (TUI plugin)

Use `fs.appendFileSync` — **NOT `Bun.write`** (append mode is broken):

```tsx
import { mkdirSync, appendFileSync } from "node:fs"

const DEBUG = process.env.MY_PLUGIN_DEBUG !== "false"
const logPath = new URL("./logs/my-plugin.log", import.meta.url).pathname
if (DEBUG) mkdirSync(new URL("./logs", import.meta.url).pathname, { recursive: true })

const log = (...args: unknown[]) => {
  if (!DEBUG) return
  const line = `[${new Date().toISOString()}] ${args.map(String).join(" ")}\n`
  appendFileSync(logPath, line)
}
```

### Logging (server plugin)

```js
await client.app.log({
  body: { service: "my-plugin", level: "info", message: "Something happened" },
})
```

### Common pitfalls

1. **`Bun.write` with `{ append: true }` overwrites** — use `fs.appendFileSync`
2. **`api.state.provider` is a `ReadonlyArray`**, not a single object — use `(api.state.provider ?? [])` for null safety
3. **`api.state.session.messages(sessionID)` requires `sessionID`** as argument
4. **`UserMessage.model` is `{ providerID, modelID }`** (object, not a string) and is **REQUIRED** — it is always present on every user message (including compaction messages)
5. **`AssistantMessage` has `modelID` and `providerID`** as separate top-level fields (not nested under `model`)
6. **TUI plugins are NOT auto-loaded** — must register in `tui.json`
7. **Paths in `tui.json` resolve relative to the config file** — use `./` prefix
8. **`EventMessageUpdated`** has `{ type, properties: { sessionID, info: Message } }`
9. **TUI plugins fail silently if not registered** — check `tui.json` if plugin doesn't appear
10. **Never store `0` in a deduplication map to mark a message as "counted but skipped"** — this permanently blacklists the ID and prevents any future retry (e.g. if the model becomes available on a later `message.updated` fire)
11. **Always read `UserMessage.model` directly** — do NOT scan adjacent `AssistantMessage`s to infer the model; for the first message in a session no assistant has responded yet, making the scan return null and silently skip counting
12. **`session.compacted` is a BusEvent (ephemeral)** — it fires in real-time but is NOT replayed when loading historical sessions. To detect past compactions, scan messages for parts with `type: "compaction"`
13. **`UserMessage.summary` is present on ALL user messages** (it is part of the `UserMessage` schema) — it is NOT specific to compaction messages. Never use it as a compaction indicator.
14. **`parts.some(p => p.synthetic)` is wrong for synthetic detection** — the first real user message in a session can have mixed parts (e.g. `text, file, text[synthetic]`). Using `.some()` incorrectly marks it as synthetic and skips counting it. The only reliable check is: a message is synthetic if and only if it has at least one `{type:"text", synthetic:true}` part AND has NO non-synthetic parts (i.e. `isSyntheticMessage()` with the "all parts must be synthetic" condition).

---

## Step 7 — Deployment

### File layout (v1 — loose files)

```
~/.config/opencode/
├── tui.json                    ← [v1-only] TUI plugin registration
├── plugins/
│   ├── my-tui-plugin.tsx       ← TUI plugin (SolidJS)
│   ├── my-tui-plugin.config.json
│   ├── my-server-plugin.js     ← Server plugin (hooks)
│   ├── my-server-plugin.config.jsonc
│   └── logs/                   ← .gitignore'd debug logs
```

### v2 layout (directory-per-plugin)

```
~/.config/opencode/               ← SEPARATE config root from v1
├── cli.json                      ← TUI-only list: { "plugins": [...] }
├── plugins/
│   └── my-plugin/                ← auto-discovered directory
│       ├── index.ts              ← server entry (or server.ts)
│       └── tui.tsx               ← TUI entry
```

Do **not** place v1 loose entry files in a v2 `plugins/` root. Core discovery admits
bare `.ts`/`.js` children and would try (and fail) to load them; a loose `.tsx` is not
auto-discovered at all. Keep v1 and v2 in **separate config roots**.

### Sync pattern (dotfiles repo)

When maintaining plugins in a dotfiles repo, copy files to `~/.config/opencode/`:

```bash
cp .config/opencode/plugins/*.tsx ~/.config/opencode/plugins/
cp .config/opencode/plugins/*.js ~/.config/opencode/plugins/
cp .config/opencode/plugins/*.json* ~/.config/opencode/plugins/
cp .config/opencode/plugins/tui.json ~/.config/opencode/tui.json
```

### .gitignore

Add to root `.gitignore`:

```
.config/opencode/plugins/logs/
```

---

## Step 8 — Event types reference

> **V2:** subscribe with `ctx.data.on(type, handler)` / `ctx.data.listen(handler)`; the
> legacy `api.event.on` below is **[v1-only]**. Payload names are namespaced in v2
> (`session.created`, `session.compacted`, `config.updated`, …).

> **SyncEvents** are persisted to the DB and replayed to clients when they connect or load a session.
> **BusEvents** are ephemeral — they fire in real-time only and are NOT replayed for historical sessions.

### Session events (SyncEvents unless noted)

```ts
EventSessionCreated   // { properties: { sessionID, info: Session } }
EventSessionUpdated   // { properties: { sessionID, info: Session } }
EventSessionDeleted   // { properties: { sessionID, info: Session } }
EventSessionIdle      // { properties: { sessionID } }          — BusEvent
EventSessionError     // { properties: { sessionID } }          — BusEvent
EventSessionCompacted // { properties: { sessionID } }          — BusEvent ⚠️ not replayed
EventSessionStatus    // { properties: { sessionID, status: SessionStatus } }
EventSessionDiff      // { properties: { sessionID, diff: FileDiff[] } } — BusEvent
```

`Session` has `parentID?: string` — sessions with `parentID` are subagent sessions.

### Message events (SyncEvents — all replayed on load)

```ts
EventMessageUpdated     // { properties: { sessionID, info: UserMessage | AssistantMessage } }
EventMessageRemoved     // { properties: { sessionID, messageID } }
EventMessagePartUpdated // { properties: { sessionID, part: Part, time: number } }
EventMessagePartRemoved // { properties: { sessionID, messageID, partID } }
```

**`EventMessageUpdated` fires for BOTH user and assistant message updates** (including streaming updates as the assistant responds). Filter by `msg.role`.

#### `UserMessage` shape (from OpenCode schema — all fields)

```ts
{
  id: string           // MessageID
  sessionID: string    // SessionID
  role: "user"
  model: {             // ⚠️ REQUIRED — never optional, always present
    providerID: string // e.g. "github-copilot"
    modelID: string    // e.g. "claude-sonnet-4"
    variant?: string
  }
  agent: string
  time: { created: number }
  // Optional fields:
  format?: OutputFormat
  summary?: { title?: string; body?: string; diffs: FileDiff[] }
  system?: string
  tools?: Record<string, boolean>
}
```

#### `AssistantMessage` shape (from OpenCode schema — key fields)

```ts
{
  id: string           // MessageID
  sessionID: string    // SessionID
  role: "assistant"
  parentID: string     // ← ID of the UserMessage that triggered this response
  modelID: string      // top-level (NOT nested under "model")
  providerID: string   // top-level (NOT nested under "model")
  mode: string         // "compaction" for compaction summary messages (deprecated)
  agent: string
  summary?: boolean    // true = this is a compaction summary assistant message
  cost: number         // 0 for compaction summaries
  tokens: {
    input: number; output: number; reasoning: number
    cache: { read: number; write: number }
    total?: number
  }
  time: { created: number; completed?: number }
  // Optional: error, path, structured, variant, finish
}
```

### Part types (`type` discriminator)

| `type` | Key fields | Notes |
|---|---|---|
| `"text"` | `text: string`, `synthetic?: boolean`, `ignored?: boolean` | `synthetic: true` = injected, not a real user input |
| `"reasoning"` | `text: string` | Model reasoning trace |
| `"tool"` | `callID`, `tool`, `state` | Tool call; state has `status: pending\|running\|completed\|error` |
| `"compaction"` | `auto: boolean`, `overflow?: boolean` | Marks a compaction user message |
| `"step-finish"` | `reason`, `cost`, `tokens` | End of an LLM response step |
| `"step-start"` | `snapshot?` | Start of an LLM response step |
| `"file"` | `mime`, `url`, `filename?`, `source?` | Attached file |
| `"patch"` | `hash`, `files: string[]` | Git patch |
| `"snapshot"` | `snapshot: string` | Filesystem snapshot |
| `"subtask"` | `prompt`, `description`, `agent`, `model?` | Subagent task |
| `"agent"` | `name`, `source?` | Agent invocation |
| `"retry"` | `attempt`, `error: APIError` | Retry event |

All parts share: `{ id: PartID, sessionID: SessionID, messageID: MessageID }`.

### Permission events

```ts
EventPermissionAsked   // { properties: { id, permission, metadata, ... } }
EventPermissionReplied // { properties: { ... } }
```

### Tool events

```ts
EventToolExecuteBefore // { properties: { tool, sessionID, callID } }
EventToolExecuteAfter  // { properties: { tool, sessionID, callID } }
```

---

## Step 9 — Session compaction internals

When a session is compacted, OpenCode performs these steps (source: `packages/opencode/src/session/compaction.ts`):

### What happens during compaction

1. **`create()`** — Creates a **user message** with a `compaction` part type and `model` set to the current model
2. **`process()`** — Makes a **real LLM API call** to generate a summary (consumes a premium request)
3. **`process()`** — Creates an **assistant message** with `summary: true`, `cost: 0`, and `modelID`/`providerID` set
4. If auto mode and `result === "continue"`:
   - Creates a **synthetic "continue" user message** with `model` set (copied from triggering user message) and a text part with `synthetic: true`
5. Publishes `session.compacted` event (**BusEvent — ephemeral, not replayed on load**)

### Message types created

| Message | Role | Part type | `model` field? | Consumes premium request? |
|---|---|---|---|---|
| Compaction user message | `user` | `compaction` | ✅ Yes (required) | Yes (via LLM call) |
| Assistant summary | `assistant` | `text` (summary) | N/A (`modelID`/`providerID` set) | No (`cost: 0`) |
| Synthetic "continue" | `user` | `text` (`synthetic: true`) | ✅ Yes (copied from triggering msg) | No |

### Detecting message types

**In `message.updated` handler** — use `api.state.part(messageID)`:

```tsx
const parts = api.state.part(msg.id!) ?? []

// Detect compaction user message (1 premium request was consumed)
const isCompaction = parts.some((p) => p.type === "compaction")

// Detect synthetic "continue" message (NOT a premium request)
// ⚠️ CRITICAL: Use isSyntheticMessage(), NOT parts.some(synthetic) — see pitfall #14 below
const isSynthetic = isSyntheticMessage(parts)

// Regular user message — neither
```

Where `isSyntheticMessage` must be defined as:

```tsx
function isSyntheticMessage(parts: Part[]): boolean {
  // Compaction user messages have a "compaction" part — never synthetic
  if (parts.some((p) => p.type === "compaction")) return false
  const syntheticTextParts = parts.filter((p) => p.type === "text" && (p as any).synthetic === true)
  if (syntheticTextParts.length === 0) return false
  // Only truly synthetic if ALL parts are synthetic text — mixed messages are real requests
  const nonSyntheticParts = parts.filter((p) => !(p.type === "text" && (p as any).synthetic === true))
  return nonSyntheticParts.length === 0
}
```

**In `fetchSessionUsage` (load path)** — use `item.parts` from `api.client.session.messages()`:

```tsx
// api.client.session.messages() returns { info: Message, parts: Part[] }[]
// parts are available directly — no separate api.state.part() call needed
const parts = (item as any).parts as Part[] ?? []
const isSynthetic = parts.some(p => p.type === "text" && (p as any).synthetic === true)
```

> **Note**: `api.state.part()` works for historically loaded sessions — parts are hydrated from the DB via the SyncEvent replay mechanism.

### Recommended handling for usage-counting plugins

```tsx
// ✅ CORRECT: Read model from UserMessage.model — always present, even for compaction messages
const userModel = (msg as any).model as { providerID: string; modelID: string } | undefined
const model = userModel?.providerID && userModel?.modelID
  ? `${userModel.providerID}/${userModel.modelID}` : null

// Skip synthetic messages — they are not real premium requests
// ⚠️ Use isSyntheticMessage() — parts.some(synthetic) wrongly skips mixed messages (see pitfall #14)
const parts = api.state.part(msg.id!) ?? []
if (isSyntheticMessage(parts)) {
  return  // ✅ Just return — do NOT store in deduplication map (storing 0 blacklists the ID)
}

// Only store in deduplication map when you have a real counted result
if (model && isCopilotModel(model)) {
  const multiplier = getMultiplier(model, config)
  if (!messageMultipliers.has(msg.id)) {
    messageMultipliers.set(msg.id, multiplier)  // store ONLY positive results
    setUsage(prev => prev + multiplier)
  }
}

// session.compacted: refresh quota only — messages are already counted via message.updated
// ⚠️ session.compacted is a BusEvent — NOT replayed for historical sessions
api.event.on("session.compacted", () => {
  fetchQuota()
})
```

> **Why NOT to scan adjacent AssistantMessages for the model**: For the first message in a session, no assistant has responded yet — the scan returns null and the message is silently skipped. `UserMessage.model` is a required schema field and is always populated at message creation time.

---

## Step 10 — Counting session usage (premium requests)

### Key design principles

1. **Read `UserMessage.model` directly** — it is always set, even for compaction messages
2. **Only store in the deduplication map when counting** — do NOT store `0` for skipped messages; that permanently blacklists the ID
3. **Use `api.client.session.messages()` for the load path** — it returns `{ info, parts }[]` for all messages; `api.state.session.messages()` is capped at 100
4. **`session.compacted` is ephemeral** — for historical sessions, detect compaction by scanning for parts with `type: "compaction"`

### Load path — counting from session history

```tsx
async function countSessionUsage(sessionID: string, api: TuiPluginApi, config: MyConfig) {
  const result = await api.client.session.messages({ sessionID, limit: 10000 })
  const messages = result.data ?? []

  let total = 0
  const counted = new Map<string, number>() // deduplication

  for (const item of messages) {
    if (item.info?.role !== "user") continue

    // Parts are returned inline — no separate api.state.part() call needed
    const parts = ((item as any).parts ?? []) as Part[]

    // Skip synthetic "continue" messages (injected after compaction, not a real request)
    // ⚠️ Must check ALL parts are synthetic — mixed messages (real + synthetic) are real requests
    if (isSyntheticMessage(parts)) continue

    // Read model directly from UserMessage.model (REQUIRED field, always present)
    const infoModel = (item.info as any)?.model
    const model = infoModel?.providerID && infoModel?.modelID
      ? `${infoModel.providerID}/${infoModel.modelID}` : null

    if (!model || !model.toLowerCase().includes("copilot")) continue

    const multiplier = getMultiplier(model, config)
    const msgId = item.id ?? (item.info as any)?.id
    if (msgId) {
      counted.set(msgId, multiplier)
      total += multiplier
    }
  }

  return { total, counted }
}
```

### Live path — counting from `message.updated` events

```tsx
// Deduplication map: message ID → multiplier (only populated for counted messages)
const messageMultipliers = new Map<string, number>()

api.event.on("message.updated", (event) => {
  const msg = event.properties.info
  if (msg.role !== "user") return

  // Deduplication — message.updated fires multiple times per message
  if (msg.id && messageMultipliers.has(msg.id)) return

  // Skip synthetic messages (⚠️ must check ALL parts are synthetic — see isSyntheticMessage)
  const parts = (api.state.part(msg.id!) ?? []) as Part[]
  if (isSyntheticMessage(parts)) return

  // Read model from UserMessage.model (tier 1 — always preferred)
  const userModel = (msg as any)?.model as { providerID: string; modelID: string } | undefined
  let model = userModel?.providerID && userModel?.modelID
    ? `${userModel.providerID}/${userModel.modelID}` : null

  // Fallback tier 2: last assistant in session state (≤100 msgs)
  if (!model) {
    const msgs = api.state.session.messages(sessionID)
    for (let i = msgs.length - 1; i >= 0; i--) {
      if (msgs[i].role === "assistant" && msgs[i].providerID && msgs[i].modelID) {
        model = `${msgs[i].providerID}/${msgs[i].modelID}`
        break
      }
    }
  }

  // Fallback tier 3: globally configured model
  if (!model) model = api.state.config?.model ?? null

  if (!model || !model.toLowerCase().includes("copilot")) return

  const multiplier = getMultiplier(model, config)
  messageMultipliers.set(msg.id!, multiplier)  // store ONLY when counting
  setUsage(prev => prev + multiplier)
})
```

### Compaction messages — are they counted automatically?

Yes. The compaction user message has `UserMessage.model` set just like any other user message. It will be picked up by both the load path and the live path above without any special handling. Its parts have `type: "compaction"` (not `type: "text" && synthetic: true`), so it passes the synthetic filter and gets counted as 1 premium request.

---

## References

- OpenCode plugins docs: <https://opencode.ai/docs/plugins/>
- OpenTUI SolidJS bindings: <https://opentui.com/docs/bindings/solid>
- OpenTUI slots: <https://opentui.com/docs/plugins/slots>
- SDK type definitions: `@opencode-ai/sdk/v2` types
- TUI plugin types: `@opencode-ai/plugin/tui` types
- Real examples: a deployed v2 plugin directory under `plugins/` (e.g.
  `plugins/<name>/` with `server.ts` + `tui.tsx`), or the deployed v1 flat file
  `.config/opencode/plugins/notifications.ts` (single-file `export const X = async () => ({ … })`).
- **`notifications` has no canonical source repository** — it exists only as a
  deployed copy (not present in the local workspace or the public `Wikijito7` org),
  so do not look for an upstream `opencode-notifications` repo to `git clone`.
