---
name: opencode-dialogs
description: "IMPORTANT: Load when creating or modifying a dialog in any OpenCode TUI plugin (v1 or v2). Enforces consistent spacing, layout, color, and interaction patterns plus the v1-vs-v2 dialog/keymap/theme lifecycle rules. Missing this = dialogs with inconsistent UX and controllers the render callback throws away."
---

## When to use me
- Adding a new dialog to any OpenCode TUI plugin
- Modifying the layout or spacing of an existing dialog
- Reviewing a dialog PR for UX consistency
- Fixing a spacing or alignment bug in a dialog

## Not intended for
- Sidebar components — those follow different layout rules
- Non-dialog UI (toasts, prompts) — OpenCode's built-in patterns
- Plugin loading / hooks / discovery → use the `opencode-plugin` skill

---

## Generation: V1 vs V2 (READ FIRST)

The dialog **design rules below apply to both generations**; the **host lifecycle
API does not**. Pick per module, and never mix.

| Concern | V1 (legacy) | V2 (current) |
|---|---|---|
| Open / replace | `api.ui.dialog.replace(render)` | `context.ui.dialog.show(render, onClose?)` |
| Size | `api.ui.dialog.setSize("large")` | `context.ui.dialog.set({ size, centered? })` |
| Close | `api.ui.dialog.clear()` | `context.ui.dialog.clear()` |
| Key layer | `api.keymap.registerLayer({ commands, bindings })` | `context.keymap.layer(() => KeymapLayer)` |
| Theme | `api.theme.current` (normalized by wlib `resolveThemeColors`) | `context.theme` (`ResolvedTheme`, semantic tokens) |

**Confirmed v2 facts (2.0.27):**

- **`KeymapLayer.mode`**: omitted ⇒ `base`. Opening a dialog pushes the active mode to
  `modal`, so a layer left at the `base` default is **inactive exactly while the dialog
  is visible**. Dialog and plugin keymap layers must set **`mode: "global"`**.
- **Theme**: `context.theme` is **resolved**, so action states drop the `$` sigil —
  `base`, `focused`, `selected` (raw theme definitions use `$focused` / `$selected`).
  Colors are **`RGBA` objects — pass them through, never string-coerce**.
- **Dialog render callbacks can re-run** whenever a signal read in the body changes.
  A controller created inside the callback is therefore replaced on every re-run (its
  effects dispose; an overlay renders against a fresh, closed instance). See
  "V2 Dialog Lifecycle" below.
- **`.tsx` vs `.ts` — the OpenCode local plugin-loader transform boundary (2.0.27).**
  OpenCode 2.0.27's local plugin loader applies its Solid runtime transform to
  **`.tsx`/`.jsx`** plugin files, so a **plugin-local module** that relies on that host
  rewrite must be `.tsx`: one that **returns JSX**, **calls Solid/OpenTUI runtime APIs**
  (`createRoot`, `createEffect`, `onMount`, signals, components), or **has a runtime
  `solid-js` / `@opentui/solid` import whose code depends on the host transform**. A
  module that is **pure `.ts` helpers** — or whose only `solid-js` import is **type-only**
  (`import type { JSX } from "solid-js"`) — stays **`.ts`**. This is the OpenCode loader's
  transform boundary, **not** a universal rule that every `solid-js` runtime import
  requires `.tsx` (a `.ts` module is fine when its Solid code is transformed elsewhere,
  e.g. a prebuilt/bundled dependency). The same code in a `.ts` file silently fails to
  transform.

---

## Command Registration

- **v2**: register through `context.keymap.layer(() => ({ mode: "global", commands: [...] }))`.
  Commands use `id`, `title`, `group`, `palette: true`, optional `slash: { name }`, and
  an optional `bind`. `mode: "global"` is **required** (see above).
- **v1 [v1-only]**: `api.keymap.registerLayer({ commands, bindings })`. See
  `examples/command-registration.tsx` (v1 skeleton).

Rules (both generations):
- The command id/name and its slash alias must match (e.g. `analyze.show` / `"analyze"`)
- Keyboard shortcut must not conflict with existing bindings
- Group/category `"Plugin"`; v1 also sets `namespace: "palette"` (v2 uses `palette: true`)

---

## Dialog Shell

Every dialog uses this outer structure. `examples/dialog-shell.tsx` is an
**illustrative snippet** (NOT a copy-run skeleton) — its imports are placeholders
and surrounding plumbing is omitted; treat it as the shape, not runnable code.

| Rule | Correct | Wrong |
|---|---|---|
| Outer padding | `paddingLeft={2} paddingRight={2} paddingBottom={1}` | `padding={2}` |
| Title bar | Always present, even in error/empty/guard states | Missing "esc" or title |
| Content wrapper | `<box paddingBottom={1}>` — **no `gap` prop** | `<box gap={1}>` — causes double-spacing |
| Between sections | `<text> </text>` — single blank line | `─` rules, double blank lines |
| Between entries | `gap={1}` on a wrapper `<box>` | `<text> </text>` spacers between entries |
| Scroll indicators | Always reserve space with `" "` when hidden | Conditional rendering that shifts layout |
| Bars | `buildBar(pct, 50)` — width 50, on own line | Custom widths, bars inline sharing a line |

---

## Title Bar

```tsx
<box flexDirection="row" justifyContent="space-between">
  <box flexDirection="row" gap={1}>
    <text fg={fg}><b>Title</b></text>
    <text fg={muted}>— Subtitle</text>
  </box>
  <text fg={muted}>esc</text>
</box>
```

- Bold title in `fg` color
- Subtitle in `muted` color, prefixed with `—` (em dash U+2014, not hyphen `-`)
- `esc` in `muted` on the far right
- **Must appear in every dialog state**: data, loading, error, empty, and pre-dialog guard states

---

## Scroll Indicators

```tsx
// Above scrollbox
<text fg={muted}>{hasData() && scroll.isScrolled() ? "▲ more above" : " "}</text>

// Below scrollbox
<text fg={muted}>{hasData() && !scroll.isAtBottom() ? "▼ more below" : " "}</text>
```

- Always render a `<text>` — use `" "` as placeholder when hidden to prevent layout shift
- Show only when content overflows (hide during loading/error/empty states)

### Scroll state tracking

Use `makeScrollState()` from the shared library (`./wlib/src/core/scroll` —
host-agnostic, used by both generations). Import it directly from the module
(plugin-relative placeholder path — substitute your own plugin's location):

```typescript
import { makeScrollState } from "./wlib/src/core/scroll" // placeholder path
```

```tsx
const scroll = makeScrollState(createSignal)

function handleKey(key: string) {
  if (key === "up") return scroll.handleUp()
  if (key === "down") return scroll.handleDown()
  return false
}
```

The factory returns:
| Property | Type | Description |
|---|---|---|
| `scrollRef` | getter/setter | Attach to scrollbox via `ref={(el) => scroll.scrollRef = el}` |
| `isScrolled()` | `() => boolean` | Whether scrolled away from top |
| `isAtBottom()` | `() => boolean` | Whether scrolled to bottom |
| `hasOverflow()` | `() => boolean` | Whether content overflows viewport |
| `handleUp()` | `() => boolean` | Scrolls up 10px, updates signals, returns true |
| `handleDown()` | `() => boolean` | Scrolls down 10px, updates signals, returns true |
| `checkOverflow()` | `() => void` | Recalculates overflow after content changes |
| `scrollToTop()` | `() => void` | Resets scroll to top, clears scroll signals |

---

## Dialog Key Layer

Use the shared `registerDialogKeyLayer(host, config)` helper
(`./wlib/src/core/keys` — plugin-relative placeholder path). It maps
`{ bindings, commands }` onto whichever keymap API the generation exposes.

> **V2 — `mode: "global"` is mandatory.** The helper defaults `mode` to `"global"`,
> but if you register a layer by hand it must set it too: on 2.0.27 the active mode is
> `modal` while a dialog is open, so a layer left at the host default (`base`) is
> inactive exactly when the dialog is visible.

### V2 (current)

Register inside the dialog **render callback** via `onMount`; the returned cleanup is a
**no-op on v2** (the reactive computation that created the layer owns its disposal):

The config below is wlib's **host-agnostic `KeyLayerConfig` wrapper shape**, so its
commands carry **`name`** (they identify the command and are the `bindings[].cmd` target).
The v2 adapter maps that wrapper onto the native `KeymapLayer`: **`name → id`** (and each
`bindings[].cmd → bind`). A `KeymapLayer` passed **directly** to `context.keymap.layer(...)`
instead uses **`id`** — see the "Complete V2 Dialog & Keymap Example" below, which writes
that native shape.

```tsx
onMount(() => {
  context.ui.dialog.set({ size: "large" })
  // wlib KeyLayerConfig wrapper shape: commands use `name`; the v2 adapter maps name→id.
  const cleanup = registerDialogKeyLayer(host, {
    priority: 1,
    mode: "global", // explicit; default is already "global"
    bindings: [
      { key: "up",   cmd: "xxx.scrollUp",   desc: "Scroll up" },
      { key: "down", cmd: "xxx.scrollDown", desc: "Scroll down" },
    ],
    commands: [
      { name: "xxx.scrollUp",   title: "Scroll Up",   run: async () => { handleKey("up") } },
      { name: "xxx.scrollDown", title: "Scroll Down", run: async () => { handleKey("down") } },
    ],
  })
  onCleanup(() => { try { cleanup() } catch { /* ignore */ } })
  loadData() // start async data fetch
})
```

### V1 (legacy)

```tsx
// v1-only
onMount(() => {
  api.ui.dialog.setSize("large")
  cleanupKeyLayer = registerDialogKeyLayer(api, { bindings: [...], commands: [...] })
  loadData()
})
onCleanup(() => { if (cleanupKeyLayer) { try { cleanupKeyLayer() } catch {} ; cleanupKeyLayer = null } })
```

- Binding namespaced to the dialog: `xxx.scrollUp`, `xxx.scrollDown`
- Always use `try/catch` on cleanup — the layer may already be disposed
- Dialog size: `"large"` for data-heavy dialogs, `"medium"` for simple messages

### Navigation dialogs

If the dialog has left/right navigation in addition to scroll, add those bindings too:

```tsx
{ key: "left",  cmd: "xxx.navLeft",  desc: "Previous" },
{ key: "right", cmd: "xxx.navRight", desc: "Next" },
```

---

## V2 Dialog Lifecycle & Controller Ownership

V2 dialogs open with `context.ui.dialog.show(render, onClose?)`, size with
`context.ui.dialog.set({ size, centered? })`, and close with
`context.ui.dialog.clear()`.

**The render callback can re-run** whenever a signal read directly in its body changes.
So a controller (export, polling, effects) created *inside* the callback is thrown away
and replaced on every re-run — its effects dispose and overlays render against a fresh,
closed instance (the confirmed export-popup bug).

Use `createOwnedInstance(factory)` (`./wlib/src/ui/owned-instance` — plugin-relative
placeholder path) to survive re-runs:

```tsx
// Constructed once, but kept INERT until .get().
const controllerOwner = createOwnedInstance(() => createExportController(host, exportable, { name: "usage" }))
let controller!: ExportController

context.ui.dialog.show(
  () => {
    // FIRST render: materialize here so the standalone createRoot this creates
    // inherits the dialog's Keymap.Provider context; later re-runs reuse the SAME
    // instance. Creating it eagerly (outside the callback) captures an owner above
    // the provider → "Keymap.Provider is missing" and the layer never registers.
    controller = controllerOwner.get()
    /* …render dialog JSX… */
    return <box>…</box>
  },
  () => {
    // onClose fires for escape, replacement (opening another dialog), and clear.
    controllerOwner.dispose() // idempotent; tears down effects + key layer exactly once
  },
)
```

Rules:
- Construct the owner wherever you like; the root + factory run on the **first `get()`,
  which must be inside the render callback** (before the controller's overlays render).
- `get()` is idempotent across re-runs; `get()` after `dispose()` throws.
- `dispose()` in `onClose` is mandatory — it covers escape, replacement, and clear.

---

## Complete V2 Dialog & Keymap Example

A single, real (illustrative) v2 TUI module — entrypoint + global keymap + a dialog
that survives render-callback re-runs — using `@opencode/plugin/tui` on OpenCode
2.0.27. It is one `.tsx` file (it returns JSX); imports into the shared library are
placeholders. Everything runs through `Plugin.Context`.

```tsx
/** @jsxImportSource @opentui/solid */
import { Plugin } from "@opencode/plugin/tui"
import { onMount } from "solid-js"
import { createOwnedInstance } from "./wlib/src/ui/owned-instance" // placeholder path

export default Plugin.define({
  id: "example",
  setup(context) {
    // GLOBAL key layer. `mode: "global"` is REQUIRED: it keeps this command and
    // its binding active in every input mode. Omitting `mode` defaults to `base`,
    // which OpenCode 2.0.27 deactivates exactly while a dialog pushes `modal`.
    // `keymap.layer` is reactive — the layer belongs to the computation that
    // created it and is disposed automatically (no manual cleanup handle).
    context.keymap.layer(() => ({
      mode: "global",
      commands: [
        {
          id: "example.show",
          title: "Example Dialog",
          group: "Plugin",
          palette: true,
          slash: { name: "example" },
          bind: "ctrl+shift+e",
          run: () => openDialog(context),
        },
      ],
    }))
  },
})

function openDialog(context: Plugin.Context) {
  // Long-lived controller (effects / its own key layer): construct the owner now,
  // but it stays INERT until `.get()`.
  const owner = createOwnedInstance(() => ({ dispose() { /* tear down effects */ } }))

  context.ui.dialog.show(
    () => {
      // Lazy `.get()` as the FIRST operation in the (re-runnable) render callback:
      // the standalone createRoot it creates inherits the dialog's Keymap.Provider
      // context, and later re-runs reuse the SAME controller.
      const controller = owner.get()
      void controller
      // Size the dialog. Idempotent, so it is safe on every re-run.
      context.ui.dialog.set({ size: "large" })
      // Dialog-scoped keys are ALSO global, so they stay active in `modal` while
      // the dialog is open. `context.ui.dialog.clear()` closes the dialog.
      onMount(() => {
        context.keymap.layer(() => ({
          mode: "global",
          commands: [
            {
              id: "example.close",
              title: "Close",
              group: "Plugin",
              bind: "escape",
              run: () => context.ui.dialog.clear(),
            },
          ],
        }))
      })
      return <box paddingLeft={2} paddingRight={2} paddingBottom={1}>{/* … content … */}</box>
    },
    () => {
      // onClose fires for escape, replacement, and clear() — dispose exactly once.
      owner.dispose()
    },
  )
}
```

**What this demonstrates**

- `context.keymap.layer(() => ({ mode: "global", … }))` for both the entry command
  and the dialog's own keys — global mode keeps keybindings live while the dialog
  is open (the active mode is `modal`, so a `base` layer would be dead).
- `context.ui.dialog.show(render, onClose?)` opens; `context.ui.dialog.set({ size })`
  sizes (idempotent, safe inside the re-runnable callback);
  `context.ui.dialog.clear()` closes.
- `createOwnedInstance` + lazy `.get()` **inside** the render callback, with
  `dispose()` in `onClose`.

---

## Stale-load Guard

Use `createLoadGuard()` from the shared library (`./wlib/src/core/reload`) to
prevent superseded async fetches from overwriting newer data. Import it directly
from the module (plugin-relative placeholder path — substitute your own plugin's location):

```typescript
import { createLoadGuard } from "./wlib/src/core/reload" // placeholder path
```

```tsx
const loadGuard = createLoadGuard()

async function loadData() {
  const gen = loadGuard.invalidate() // increments generation, returns new number
  // ... fetch data ...
  if (!loadGuard.isCurrent(gen)) return // discard — newer fetch superseded this one
  // ... update state ...
}

function reload() {
  loadGuard.invalidate() // invalidate current generation before starting fresh fetch
  loadData()
}
```

The factory returns:
| Method | Description |
|---|---|
| `invalidate(): number` | Increments generation, returns new number. Call before each fetch. |
| `isCurrent(gen: number): boolean` | Returns true if `gen` is still the latest generation. Use to check after async fetch. |

---

## Content Wrapper

Everything inside the scrollbox data state:

```tsx
<box paddingBottom={1}>
  {/* sections */}
</box>
```

- **`paddingBottom={1}` only — NO `gap` prop**
- All spacing is explicit via `<text> </text>` elements
- Never use `flexDirection="column" gap={1}` on the content wrapper

---

## Entry Patterns

See `examples/entry-patterns.tsx` for illustrative presentation snippets (values are
intentionally omitted — they show layout shape, not runnable code).

### Entry with bar (categories, tools, models)

- Wrap entries in `<box flexDirection="column" gap={1}>` for between-entry spacing
- Each entry is a `<box flexDirection="column" gap={1}>` with name, stats line, bar
- NO `<text> </text>` spacers between entries — `gap={1}` on the wrapper handles it
- Bar is always on its own line, width 50

### Single-line entry (numbered lists)

- Wrap in `<box flexDirection="column" gap={0}>` — entries are single-line
- Numbers use `padStart(2)` for fixed-width alignment
- Label uses `padEnd(24)` for column alignment

---

## Section Headers and Separators

- Headers in `fg` bold
- One blank line after header, one blank line before next section header
- **Never use horizontal rules** — blank lines only
- Conditional sections carry their own spacing inside the fragment

---

## Summary / Total Line

```tsx
<text fg={fg}>Total: {value} units ({context})</text>
```

- In `fg` (not bold, not muted)
- Context in parentheses: e.g. `(90 msgs)` or `($12.34)`
- Surrounded by `<text> </text>` above and below for section separation

---

## Footer

```tsx
{/* Scroll-only dialog */}
<text fg={muted}>↑↓ scroll  ·  esc close</text>

{/* Navigation dialog */}
<text fg={muted}>← → navigate  ·  ↑↓ scroll  ·  esc close</text>
```

- In `muted` color
- Separators use `·` (middle dot U+00B7) with spaces: `  ·  `
- Only shown when data is loaded
- **List only keys the dialog actually binds** — the footer advertises the real key
  layer (`esc` close; `↑↓` scroll; `← →` navigation when present), never a key the
  dialog does not register

---

## States

See `examples/states.tsx` for illustrative presentation snippets (values are
intentionally omitted — they show layout shape, not runnable code).

### Loading

```tsx
<text fg={muted}>Loading description…</text>
```

### Error

```tsx
<box flexDirection="column" gap={1}>
  <text fg={red}><b>Error Title</b></text>
  <text fg={muted}>{errorMsg()}</text>
</box>
```

### Empty

```tsx
<text fg={muted}>No data for this view.</text>
```

### Pre-dialog guard

Must include the title bar with "esc". Dialog size: `"medium"`.

---

## Color Conventions

| Element | Color |
|---|---|
| Bold titles, entry names, bars, values, totals | `fg` (foreground, `#ffffff`) |
| Stats, subtitles, hints, muted text, "esc" | `muted` (`#888888`) |
| Errors | `red` (`#ef4444`) |

```tsx
// v1-only shape (api.theme.current)
const fg = theme?.foreground ?? "#ffffff"
const muted = theme?.muted ?? "#888888"
const red = theme?.red ?? "#ef4444"
```

**v2 theme:** resolve `context.theme` (`ResolvedTheme`) through the shared theme helper
(`resolveSemanticTheme`) into `fg` / `muted` / `red` / `primary` / `selectedText` /
`panel`. Resolved action states are `base` / `focused` / `selected` (no `$`; raw theme
definitions use `$focused` / `$selected`). The values are `RGBA` objects — **pass them
through, never string-coerce**. `resolveSemanticTheme` **always returns a required
`selectedText`**: when the focused/selected/base states are missing or transparent it
applies a **resolved `fg` fallback**, so v2 `selectedText` is never `undefined`. (The v1
loose resolver — `resolveThemeColors` — may still yield `undefined`; apply the fallback
yourself there.) The selection convention (below) is shared across v1 and v2.

```tsx
// v2: resolve `context.theme` (a ResolvedTheme) once, then read semantic tokens.
// Resolved keys carry NO `$` sigil — read `focused`/`selected`/`base`, not the raw
// `$focused`/`$selected` definition keys — and every value is an RGBA object.
const theme = resolveSemanticTheme(context.theme)

// Selected chip: background ← the resolved `focused` action surface (`primary`),
// text ← the resolved `focused` action text (`selectedText`). `resolveSemanticTheme`
// falls back `selectedText` to the resolved `fg` when the focused/selected/base text
// states are missing or transparent, so it is never undefined in v2.
<box backgroundColor={theme.primary}>
  <text fg={theme.selectedText}>{chipLabel}</text>
</box>
```

---

## Bar Conventions

| Property | Value |
|---|---|
| Function | `buildBar(percentage, 50)` from `./helpers/format.ts` (plugin-relative example) |
| Width | 50 characters |
| Filled char | `\u2588` (full block) |
| Empty char | `\u2591` (light shade) |
| Line color | `fg` |
| Position | Own line, after stats line, inside entry box |

Each bar is proportional to the entry's percentage of the **total** (not relative to the largest entry).

---

## Export Flow & Controller Ownership

Export is a three-step overlay owned by a long-lived controller
(`./wlib/src/ui/export-controller` — plugin-relative placeholder path), never re-created
per render:

1. **format** — `↑↓` pick a format (`EXPORT_FORMATS`: Markdown / CSV / JSON / plain text),
   `enter` → **destination**.
2. **destination** — `↑↓` choose `Clipboard` or `File`, `enter` runs it, `esc` → back.
   Clipboard success flashes `copied!` (2 s); File writes then advances to **result**.
3. **result** — on **success**, shows the saved path; `←` / `→` move focus between the
   **Close** and **Open** buttons, `enter` activates the focused one (focused **Open**
   opens the directory then closes, focused **Close** closes), `o` opens the folder
   directly then closes, and `esc` closes. On **error** (the file write failed) the
   overlay omits `Open` and shows only `enter close · esc close`: only `Enter` and
   `Escape` close the dialog. The remaining registered dialog/export keys — the
   arrows, `o`, and the rest — are **consumed** without closing or opening anything,
   so Open/open-folder can never fire; `o` deliberately does **not** close the error
   popup (unlike the success state). (The keymap only invokes handlers for registered
   bindings, so truly unbound keys are never dispatched/swallowed. The Close/Open
   focus navigation applies to the success state only, not to the error state.)

Ownership:
- `createExportController(host, exportable, { name })` registers its own `priority: 2`
  key layer (via `createEffect` + `onCleanup`) and the flash timeout, so it **must live
  inside a stable Solid owner** — create it through `createOwnedInstance` and `.get()` it
  inside the dialog render callback (see "V2 Dialog Lifecycle & Controller Ownership").
- The host dialog includes `renderOverlay()` / `renderResultOverlay()` **once** in its
  JSX and renders the footer flash from `copiedFlash()`; the callbacks return a
  persistently-mounted `Show` so they stay reactive even if the host evaluates the JSX
  expression only once.
- `dispose()` the owner in the dialog's `onClose`.
- Feature-specific serializers (`Exportable.build(format)`) live in the consuming plugin,
  not in wlib; the controller owns only the flow.

---

## References

Paths below are **plugin-relative examples** (relative to a plugin root; shown as laid
out in the reference `model-usage` plugin — substitute your own plugin's layout). They
use the same `./…` convention as the inline import placeholders above.

- `./usage.tsx` — reference: navigation + scroll dialog (`/usage`)
- `./analyze.tsx` — reference: tabbed dialog with scroll, tabs, and reload (`/analyze`)
- `./v2/usage.tsx`, `./v2/analyze.tsx` — v2 dialog + export-controller owner examples
- `./helpers/format.ts` — shared utilities (`buildBar`, `fmt`, `buildProgressBar`, `getUsageColor`, `formatDuration`, `getQuotaLabel`)
- `./helpers/debug.ts` — debug logging (`DEBUG`, `log`)
- `./wlib/src/ui/dialog.tsx` — `useDialogSizing()`, `DialogShell`
- `./wlib/src/core/keys.ts` — `registerDialogKeyLayer(host, config)` (v2 `mode:"global"`)
- `./wlib/src/core/scroll.ts` — `makeScrollState` factory for scroll signals + handlers
- `./wlib/src/core/reload.ts` — `createLoadGuard()` factory for stale-fetch prevention
- `./wlib/src/ui/owned-instance.tsx` — `createOwnedInstance()` stable owner
- `./wlib/src/ui/export-controller.tsx` — the export flow controller
- `opencode-plugin` skill — plugin generation selection + API patterns
