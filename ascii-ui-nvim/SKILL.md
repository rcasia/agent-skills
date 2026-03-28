---
name: ascii-ui-nvim
description: >
  Building Neovim plugin UIs with ascii-ui.nvim — use when writing a Neovim
  plugin that needs a floating window UI, interactive components, or animated
  terminal output. Covers the component model, hooks, best practices, and
  common patterns. Load when the user mentions ascii-ui, asks how to build a
  Neovim UI, or is writing a plugin that mounts a floating window.
---

# ascii-ui.nvim

Plugin UI framework for Neovim with a React-inspired component model.
Components are pure Lua functions that return renderable lines. The framework
diffs and re-renders only what changed.

**Docs:** https://rcasia.github.io/ascii-ui-docs/
**Repo:** https://github.com/rcasia/ascii-ui.nvim

## When to Load This Skill

- User is building a Neovim plugin that needs a UI
- User mentions `ascii-ui`, `ascii-ui.nvim`, or `ui.mount`
- User wants interactive widgets (buttons, inputs, sliders) in Neovim
- User asks how to animate or update a floating window at runtime
- User asks about components, hooks, `useState`, or `useInterval` in a Neovim context

## Mental Model

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

---

## Quick Start

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
{ "rcasia/ascii-ui.nvim", opts = {} }
```

---

## Key Building Blocks

| Concept | What it is |
|---|---|
| `ui.createComponent(name, fn)` | Registers a component; `fn` is the render function |
| `ui.mount(App)` | Opens a floating window and starts the event loop |
| `BufferLine` | One rendered row — a horizontal list of `Segment`s |
| `Segment` | Smallest unit: a string with optional color / highlight |
| `Segment:wrap()` | Shorthand — wraps a lone segment in a `BufferLine` |
| Hooks | `useState`, `useEffect`, `useInterval`, `useReducer`, … |
| `ui.map(items, fn)` | Safe list rendering — returns a flat array of nodes |

See [components.md](components.md) for all built-in components.
See [hooks.md](hooks.md) for all hooks with gotchas.
See [patterns.md](patterns.md) for recipes and best practices.

---

## Best Practices

### 1. Keep component bodies pure

No Neovim API calls (`vim.api.*`, `vim.cmd`, `vim.fn.*`) during render.
They belong in `useEffect` or event callbacks.

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

### 2. Return a flat table — never nested arrays

The return value must be `BufferLine[]` or `FiberNode[]`. Nested tables of
tables confuse the reconciler.

```lua
-- WRONG
return { { Paragraph({ content = "a" }), Paragraph({ content = "b" }) } }

-- RIGHT
return {
  Paragraph({ content = "a" }),
  Paragraph({ content = "b" }),
}
```

When you have a dynamic sub-list, flatten it with `ui.map` or
`vim.list_extend`.

### 3. Use `ui.map` for lists, not manual loops

`ui.map` returns a flat array and reads like intent.

```lua
-- WRONG: manual loop silently nests the result
local rows = {}
for _, item in ipairs(items) do
  table.insert(rows, Paragraph({ content = item }))
end
return { rows }   -- oops — nested

-- RIGHT
return {
  ui.map(items, function(item)
    return Paragraph({ content = item })
  end),
}
```

### 4. Conditional rendering with plain `if/else`

Assign to a local variable, include the variable in the return table.

```lua
local App = ui.createComponent("App", function()
  local visible, setVisible = useState(true)

  local body
  if visible then
    body = Paragraph({ content = "Visible!" })
  else
    body = Paragraph({ content = "(hidden)" })
  end

  return {
    body,
    Button({ label = "Toggle", on_press = function() setVisible(not visible) end }),
  }
end)
```

Do **not** use `and`/`or` short-circuit tricks — they return the wrong type
when the condition is false.

### 5. Lift state up when siblings share data

If two components need the same value, own the state in their parent and pass
it as props.

```lua
local Parent = ui.createComponent("Parent", function()
  local query, setQuery = useState("")

  return {
    SearchInput({ value = query, on_change = setQuery }),
    ResultsList({ query = query }),
  }
end)
```

### 6. Prefer `useReducer` over multiple related `useState` calls

When state transitions are coupled (e.g. a list with an active index), group
them in a reducer — it prevents stale-closure bugs and makes transitions
explicit.

```lua
-- Prefer this for related state
local state, dispatch = useReducer(function(s, action)
  if action.type == "select" then
    return { items = s.items, selected = action.index }
  end
  if action.type == "add" then
    return { items = vim.list_extend({}, s.items, { action.item }), selected = s.selected }
  end
  return s
end, { items = {}, selected = 1 })
```

### 7. Color segments with `color`, use `highlight` for theme-aware groups

`color = { fg = "#rrggbb", bg = "#rrggbb" }` on a `Segment` applies a
truecolor hex directly. Use it for hardcoded brand colors.
`highlight = "DiagnosticError"` defers to the user's colorscheme — prefer
it for semantic colors.

```lua
-- hardcoded hex
Segment:new({ content = "●", color = { fg = "#ff6b6b" } }):wrap()

-- theme-aware
Segment:new({ content = "✖ error", highlight = "DiagnosticError" }):wrap()
```

### 8. Separate data, logic, and rendering into layers

For non-trivial UIs (clocks, charts, boards) keep three distinct layers:
- **data layer** — fetches / computes raw values (pure Lua, no ascii-ui)
- **render layer** — converts values to `BufferLine[]` tables (pure, no hooks)
- **component layer** — owns state and hooks, calls the render layer

This makes the render logic independently testable without mounting a window.

### 9. Clean up timers and effects explicitly

`useInterval` cleans up its timer automatically on unmount. For manual
`useEffect` timers, return the cleanup function.

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

---

## Anti-Patterns

| Anti-pattern | Why it breaks | Fix |
|---|---|---|
| Mutating the value from `useState` | Returns a deep copy — mutations are silently lost | Call the setter: `setList(newList)` |
| Calling hooks conditionally | Hooks must run in identical order every render | Move the condition inside the hook callback |
| Calling hooks inside loops | Same order rule — loop count must be fixed | Use `useReducer` or pre-allocate fixed hooks |
| Defining a component inside another | A new component type is registered on every render, leaking memory | Define components at module level |
| Calling `ui.mount()` inside a component | Creates infinite window recursion | Call `ui.mount` at the top level of your plugin command |
| Reusing a `Segment` object across renders | Segments carry auto-incrementing IDs; sharing IDs corrupts focus tracking | Create a fresh `Segment:new(...)` each render |
| Returning `nil` or a bare string | The reconciler expects `FiberNode[]` | Wrap strings in `Paragraph` or `Segment:wrap()` |

---

## Reference Files

- [components.md](components.md) — built-in component props and examples
- [hooks.md](hooks.md) — hook signatures, return values, and gotchas
- [patterns.md](patterns.md) — full recipes for common UI patterns
