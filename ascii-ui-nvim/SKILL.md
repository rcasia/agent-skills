---
name: ascii-ui-nvim
description: >
  Building Neovim plugin UIs with ascii-ui.nvim — use when writing a Neovim
  plugin that needs a floating window UI, interactive components, or animated
  terminal output. Organized as context, design rules, a change workflow, and
  a final checklist, with deep references for components, hooks, and patterns.
  Load when the user mentions ascii-ui, asks how to build a Neovim UI, or is
  writing a plugin that mounts a floating window.
metadata:
  source_repo: https://github.com/ascii-ui/ascii-ui.nvim
  source_commit: a88bbb033eeb70e058454797cb96ddf6f15aa611
  source_commit_date: "2026-08-14"
  skill_synced: "2026-09-05"
---

# ascii-ui.nvim

A skill for building Neovim plugin UIs with
[ascii-ui.nvim](https://github.com/ascii-ui/ascii-ui.nvim), a React-inspired
UI framework. It is organized in four parts:

```
1. Context   → what ascii-ui is and the building blocks
2. Rules     → design principles and coding rules that never bend
3. Workflow  → the steps to approach any change
4. Checklist → simple yes/no questions to close the change
```

**Docs:** https://ascii-ui.github.io/ascii-ui-docs/
**Repo:** https://github.com/ascii-ui/ascii-ui.nvim

---

## 1. Context

### When to use this skill

- User is building a Neovim plugin that needs a UI
- User mentions `ascii-ui`, `ascii-ui.nvim`, or `ui.mount`
- User wants interactive widgets (buttons, inputs, sliders) in Neovim
- User asks how to animate or update a floating window at runtime
- User asks about components, hooks, `useState`, or `useInterval` in a
  Neovim context

### Mental model

ascii-ui is **not** a DOM — it is a **line-oriented** renderer. A component
returns a flat table of `BufferLine` objects (rows). Each `BufferLine` is a
list of `Segment` objects (columns). The framework reconciles this tree on
every state change and writes only the dirty lines to the Neovim buffer.

```
App (component)
 └─ BufferLine   ← one row
     ├─ Segment  ← "Hello, "
     └─ Segment  ← "world"  (colored #ff0000)
```

Hooks (`useState`, `useEffect`, …) are called inside the component body
**in the same order on every render** — identical rule to React.

### Quick Start

```lua
local ui = require("ascii-ui")
local Paragraph = ui.components.Paragraph
local Button    = ui.components.Button
local useState  = ui.hooks.useState

local App = ui.createComponent("App", function()
  local count, setCount = useState(0)

  return {
    Paragraph({ content = "Count: " .. count }),
    Button({
      label    = "Increment",
      on_press = function() setCount(count + 1) end,
    }),
  }
end)

ui.mount(App)
```

Install (lazy.nvim):

```lua
{ "ascii-ui/ascii-ui.nvim", opts = {} }
```

Also available on LuaRocks (`luarocks install ascii-ui`) and lux
(`lux install ascii-ui`).

### Key Building Blocks

| Concept | What it is |
|---|---|
| `ui.createComponent(name, fn)` | Registers a component; `fn` is the render function |
| `ui.mount(App)` | Opens a floating window and starts the event loop |
| `BufferLine` | One rendered row — a horizontal list of `Segment`s |
| `Segment` | Smallest unit: a string with optional color / highlight |
| `Segment:wrap()` | Shorthand — wraps a lone segment in a `BufferLine` |
| Hooks | `useState`, `useEffect`, `useReducer`, `useConfig`, `useInterval`, `useTimeout` |
| `ui.map(items, fn)` | Safe list rendering — returns a flat array of nodes |
| `ui.layout.Row` / `ui.layout.Column` | Side-by-side / stacked child layout |
| `ui.Color` | Unified truecolor API (hex, `{fg,bg}` tables, HSL) |
| `ui.viewports.StdoutViewport` | Render to terminal stdout instead of a window |
| `ui.debug(file)` | Live-reload development: mount a file, reload on save |
| `require("ascii-ui.testing")` | Component test harness (render → query → interact) |

### Deep references

- [components.md](components.md) — built-in component props and examples
- [hooks.md](hooks.md) — hook signatures, return values, and gotchas
- [patterns.md](patterns.md) — full recipes for common UI patterns

### Staying current

This skill mirrors upstream commit `a88bbb0` (2026-08-14) — recorded in
`metadata.source_commit` in the frontmatter. To update it, diff upstream
since that commit and revise the reference files:

```sh
cd path/to/ascii-ui.nvim
git fetch origin
git log --oneline a88bbb0..HEAD -- lua/ascii-ui docs/ README.md
git diff a88bbb0..HEAD -- lua/ascii-ui/init.lua lua/ascii-ui/components lua/ascii-ui/hooks
```

After editing, bump `metadata.source_commit` and `metadata.skill_synced`.

---

## 2. Rules

### Design principles

Seven rules come before any implementation decision:

1. **Simplicity is the top priority.** When choosing between alternatives,
   pick the one with the lowest cognitive complexity for a person reading
   the code for the first time — clear over clever.

2. **One responsibility per component.** A component must not mix
   presentation, state, and layout at once. It should have a single reason
   to change. If you cannot describe what it does in one sentence, split
   it: hoist state into a parent, delegate arrangement to `Row`/`Column`,
   and extract pure rendering into its own component.

3. **The public API must be declarative — designed for whoever uses it.**
   Whoever uses a component should express *what* they want, not *how* to
   implement it. Prefer props that describe intent (`value`, `label`,
   `on_change`) over callbacks that dictate mechanics; hide rendering,
   timing, and cleanup details inside the component.

4. **A component must be understandable in under two minutes.** If a
   first-time reader cannot grasp what it renders and when it re-renders
   in that time, simplify it: shorten the render body, flatten the
   conditionals, and extract what doesn't belong.

5. **Prefer composition over specialization.** Combine simpler pieces
   instead of creating complex variants. Don't grow a component with flag
   props (`show_header`, `variant = "compact"`) that branch into different
   behaviors; build small components and put them together —
   `Row`/`Column` for arrangement, `Tree` children for embedded nodes,
   children props for slots. A new use case should mean a new combination,
   not a new prop.

6. **Prefer composability.** Design every piece so it fits inside any
   other: components accept and return the standard types
   (`FiberNode[]` / `BufferLine[]`), take `children` instead of hard-coding
   content, and never assume a fixed size, position, or parent. A component
   that only works in one place is a design smell.

7. **Errors must be explicit and handleable — never silent.** Failures are
   surfaced, not swallowed: invalid props raise at call time, and render
   errors report the component-tree path and reason instead of quietly
   rendering nothing. In your own code, fail with a message that says what
   failed and why, and give the caller a way to handle it — don't hide
   failures behind bare `pcall`.

### Coding rules

The concrete habits that keep ascii-ui code correct.

**1. Keep component bodies pure.** No Neovim API calls (`vim.api.*`,
`vim.cmd`, `vim.fn.*`) during render. They belong in `useEffect` or event
callbacks.

```lua
-- WRONG: side effect in render body
local App = ui.createComponent("App", function()
  vim.cmd("echo 'rendered'")          -- crashes or causes flicker
  return { Paragraph({ content = "hi" }) }
end)

-- RIGHT: side effect in effect
local App = ui.createComponent("App", function()
  useEffect(function()
    vim.cmd("echo 'mounted'")
  end, {})                             -- {} = run once on mount
  return { Paragraph({ content = "hi" }) }
end)
```

**2. Return a flat table — never nested arrays.** The return value must be
`BufferLine[]` or `FiberNode[]`. Nested tables of tables confuse the
reconciler. When you have a dynamic sub-list, flatten it with `ui.map`.

```lua
-- WRONG
return { { Paragraph({ content = "a" }), Paragraph({ content = "b" }) } }

-- RIGHT
return {
  ui.map(items, function(item)
    return Paragraph({ content = item })
  end),
}
```

**3. Conditional rendering with plain `if/else`.** Assign to a local
variable, include the variable in the return table. Do **not** use
`and`/`or` short-circuit tricks — they return the wrong type when the
condition is false.

```lua
local body
if visible then
  body = Paragraph({ content = "Visible!" })
else
  body = Paragraph({ content = "(hidden)" })
end
return { body }
```

**4. Lift state up when siblings share data.** If two components need the
same value, own the state in their parent and pass it as props.

**5. Prefer `useReducer` over multiple related `useState` calls.** When
state transitions are coupled (e.g. a list with an active index), group
them in a reducer — it prevents stale-closure bugs and makes transitions
explicit.

**6. Use the functional setter inside closures.** Callbacks capture state
at render time. Read fresh values with `setCount(function(prev) ... end)`.

**7. Color: hex strings, `{fg, bg}` tables, or `ui.Color`.** For
hardcoded brand values; `highlight = "DiagnosticError"` defers to the
user's colorscheme — prefer it for semantic colors.

```lua
Segment:new({ content = "●", color = "#ff6b6b" }):wrap()
Segment:new({ content = " OK ", color = { fg = "#000000", bg = "#4caf50" } }):wrap()
Segment:new({ content = "✖ error", highlight = "DiagnosticError" }):wrap()
```

**8. Separate data, logic, and rendering into layers.** For non-trivial
UIs keep three layers — **data** (reads external state), **logic** (pure
computation), **render** (values → `BufferLine[]`, no hooks) — with the
component layer owning state on top. The pure layers are testable without
mounting anything. See pattern 8 in [patterns.md](patterns.md).

**9. Clean up timers and effects explicitly.** `useInterval` and
`useTimeout` tear down their timers automatically on unmount and when
`delay` changes. For manual `useEffect` resources, return the cleanup
function.

```lua
useEffect(function()
  local timer = vim.uv.new_timer()
  timer:start(0, 500, vim.schedule_wrap(function() ... end))
  return function()           -- cleanup runs on unmount or dep change
    timer:stop()
    timer:close()
  end
end, {})
```

### Anti-patterns

| Anti-pattern | Why it breaks | Fix |
|---|---|---|
| Mutating the value from `useState` | Returns a deep copy — mutations are silently lost | Call the setter: `setList(newList)` |
| Calling hooks conditionally | Hooks must run in identical order every render | Move the condition inside the hook callback |
| Calling hooks inside loops | Same order rule — loop count must be fixed | Use `useReducer` or pre-allocate fixed hooks |
| Defining a component inside another | A new component type is registered on every render, leaking memory | Define components at module level |
| Calling `ui.mount()` inside a component | Creates infinite window recursion | Call `ui.mount` at the top level of your plugin command |
| Reusing a `Segment` object across renders | Segments carry auto-incrementing IDs; sharing IDs corrupts focus tracking | Create a fresh `Segment:new(...)` each render |
| Returning `nil` or a bare string | The reconciler expects `FiberNode[]` | Wrap strings in `Paragraph` or `Segment:wrap()` |
| Reading `ui.components.Tree/Box/Checkbox` | Only `Paragraph`, `Button`, `Input`, `Select`, `Slider` are on `ui.components` | `require("ascii-ui.components.tree")` (also `box`, `checkbox`) |

---

## 3. Workflow — approaching a change

The order to do things in, for any feature or fix:

1. **Frame it in one sentence.** "This component renders X and changes
   when Y." If the sentence needs "and", split the component (Rule 2).

2. **Write the pure layers first.** Data + logic + render as plain Lua
   functions — values in, `BufferLine[]` out. No ascii-ui state yet.
   (Coding rule 8; pattern 8 in patterns.md.)

3. **Pick building blocks.** Built-in components before custom ones;
   `Row`/`Column` for arrangement; raw `Segment`/`BufferLine` only when
   nothing fits. Check components.md for the exact props.

4. **Decide state ownership.** One owner per value. Shared state lives in
   the parent; coupled transitions go in a single `useReducer`.

5. **Write the component body.** Hooks at top level in stable order,
   flat return table, `ui.map` for lists, plain `if/else` for branches,
   fresh segments each render (coding rules 1–3, 6).

6. **Wire interactivity and effects.** `on_press` / `on_change` callbacks;
   `useInterval` / `useTimeout` for time; autocmds and subscriptions in
   `useEffect` **with cleanup** (coding rule 9).

7. **Iterate live.** Develop with `require("ascii-ui").debug("file.lua")`
   — save and the window re-mounts; errors show as notifications.

8. **Verify.** Run the checklist below. Test the pure layers directly;
   test component behavior with `require("ascii-ui.testing").render(...)`
   (pattern 12 in patterns.md).

---

## 4. Final checklist

Answer every question yes/no before calling the change done. A "no" sends
you back to the workflow step that owns it.

**Design (rules 1–6):**
- [ ] Can I describe each component's responsibility in one sentence?
- [ ] Would a first-time reader understand each component in two minutes?
- [ ] Does the public API say *what* the caller wants, not *how* it works?
- [ ] Did any flag/variant props appear that should be separate pieces?
- [ ] Could any component be reused elsewhere — or did it assume a fixed
      parent, size, or position?

**Code (coding rules):**
- [ ] No `vim.api` / `vim.cmd` / timers in any render body?
- [ ] Every component returns a flat `BufferLine[]` / `FiberNode[]`?
- [ ] Hooks unconditional, same order every render?
- [ ] Closures that read state use the functional setter?
- [ ] Every effect that allocates a resource returns a cleanup?
- [ ] Fresh `Segment` instances each render (none shared across renders)?

**Robustness (rule 7):**
- [ ] Failures raise with a clear message — no bare `pcall` swallowing
      errors?
- [ ] Prop types declared where validation matters?

**Verification (workflow step 8):**
- [ ] Pure layers tested with plain assertions?
- [ ] Interactive behavior tested with `ui.testing.render`?
- [ ] Mounted once via `ui.debug`: renders, responds, quits cleanly?
