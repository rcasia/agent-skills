# ascii-ui Patterns

Common recipes for building Neovim plugin UIs with ascii-ui.nvim.

---

## Pattern 1 — Timer / Clock (live-updating display)

Use `useInterval` + `useState` to drive a value that changes over time.
Lift the time state into the component that owns the display — don't put it
in a child that re-creates the interval every parent render.

```lua
local ui          = require("ascii-ui")
local BufferLine  = require("ascii-ui.buffer.bufferline")
local Segment     = require("ascii-ui.buffer.segment")
local useState    = ui.hooks.useState
local useInterval = ui.hooks.useInterval

local Clock = ui.createComponent("Clock", function()
  -- Own the time state here, at the top level of the component that renders it.
  local time, setTime = useState(os.date("%H:%M:%S"))

  useInterval(function()
    setTime(os.date("%H:%M:%S"))
  end, 1000)

  return {
    Segment:new({ content = time, color = "#4ecdc4" }):wrap(),
  }
end)
```

**Rules:**
- Always use the functional setter (`setTime(fn)`) if you read old state
  inside the interval — captured values go stale.
- Pass `nil` delay to pause: `useInterval(fn, paused and nil or 1000)`.

---

## Pattern 2 — List rendering

Use `ui.map` for any dynamic list. It returns a flat `FiberNode[]` array
that the reconciler understands directly.

```lua
local Paragraph = ui.components.Paragraph
local Button    = ui.components.Button

local List = ui.createComponent("List", function(props)
  -- props.items is a string[]
  return {
    ui.map(props.items, function(item, index)
      return Paragraph({ content = index .. ". " .. item })
    end),
    Button({
      label    = "Add",
      on_press = props.on_add,
    }),
  }
end, { on_add = "function" })
```

**Do not** build the list with `table.insert` and then return it wrapped in
an extra `{}` — that nests the array one level too deep.

```lua
-- WRONG
local rows = {}
for i, item in ipairs(items) do
  table.insert(rows, Paragraph({ content = item }))
end
return { rows }     -- rows is nested inside another array

-- RIGHT
return {
  ui.map(items, function(item) return Paragraph({ content = item }) end),
}
```

---

## Pattern 3 — Conditional rendering

Assign to a local variable in the component body; include the variable in
the return table. Plain `if/else` — no special syntax needed.

```lua
local App = ui.createComponent("App", function()
  local loggedIn, setLoggedIn = useState(false)

  local body
  if loggedIn then
    body = Paragraph({ content = "Welcome back!" })
  else
    body = Paragraph({ content = "Please log in." })
  end

  return {
    body,
    Button({
      label    = loggedIn and "Log out" or "Log in",
      on_press = function() setLoggedIn(not loggedIn) end,
    }),
  }
end)
```

**Avoid `and`/`or` short-circuit:** `false and x or y` evaluates to `y` when
`x` is falsy. With a component call that returns a `FiberNode`, `x` is always
truthy, so it works — but it is fragile and unreadable. Use explicit `if/else`.

---

## Pattern 4 — Lifting state up

When two sibling components need the same value, own the state in their common
parent and pass it down as props.

```lua
-- SearchInput is a child that fires on_change
local SearchInput = ui.createComponent("SearchInput", function(props)
  return {
    Input({
      value     = props.value,
      on_change = props.on_change,
    }),
  }
end)

-- ResultsList reads the query
local ResultsList = ui.createComponent("ResultsList", function(props)
  local results = search_index(props.query)   -- pure function, no hooks
  return {
    ui.map(results, function(r)
      return Paragraph({ content = r.title })
    end),
  }
end)

-- Parent owns the shared state
local SearchUI = ui.createComponent("SearchUI", function()
  local query, setQuery = useState("")

  return {
    SearchInput({ value = query, on_change = setQuery }),
    ResultsList({ query = query }),
  }
end)
```

---

## Pattern 5 — Complex state with useReducer

When multiple fields change together (e.g. a list with a selected index),
collect them in one reducer to keep transitions atomic and explicit.

```lua
local useReducer = ui.hooks.useReducer

local function list_reducer(state, action)
  if action.type == "add" then
    local items = vim.list_extend({}, state.items)
    table.insert(items, action.item)
    return { items = items, selected = #items }
  end
  if action.type == "remove" then
    local items = vim.iter(state.items):enumerate():filter(function(i)
      return i ~= state.selected
    end):totable()
    local selected = math.max(1, state.selected - 1)
    return { items = items, selected = selected }
  end
  if action.type == "select" then
    return { items = state.items, selected = action.index }
  end
  return state
end

local ListManager = ui.createComponent("ListManager", function()
  local state, dispatch = useReducer(list_reducer, { items = { "item 1" }, selected = 1 })

  return {
    ui.map(state.items, function(item, i)
      local prefix = i == state.selected and "> " or "  "
      return Button({
        label    = prefix .. item,
        on_press = function() dispatch({ type = "select", index = i }) end,
      })
    end),
    Button({
      label    = "+ Add",
      on_press = function()
        dispatch({ type = "add", item = "item " .. (#state.items + 1) })
      end,
    }),
    Button({
      label    = "- Remove",
      on_press = function() dispatch({ type = "remove" }) end,
    }),
  }
end)
```

---

## Pattern 6 — Custom colored row (low-level Segment + BufferLine)

When you need a row where each word has a different color, build it manually
with `Segment` and `BufferLine`. Colors accept a hex string, a `{fg, bg}`
table, or a `ui.Color` instance.

```lua
local BufferLine = require("ascii-ui.buffer.bufferline")
local Segment    = require("ascii-ui.buffer.segment")

-- A status bar: "[NORMAL] filename  col:1 line:1"
local function make_statusbar(mode, filename, col, line)
  local mode_colors = {
    NORMAL  = "#4ecdc4",
    INSERT  = "#ff6b6b",
    VISUAL  = "#ffe66d",
  }

  return BufferLine.new(
    Segment:new({ content = "[" .. mode .. "]", color = mode_colors[mode] or "#ffffff" }),
    Segment:new({ content = " " .. filename .. "  " }),
    Segment:new({ content = "col:" .. col .. " line:" .. line, color = "#8b949e" })
  )
end

local Statusbar = ui.createComponent("Statusbar", function(props)
  return { make_statusbar(props.mode, props.filename, props.col, props.line) }
end)
```

---

## Pattern 7 — Animated / computed colors with the Color API

`ui.Color` derives palettes and shades at runtime — ideal for charts, heat
maps, and smooth animations without hand-picking hex values.

```lua
local ui = require("ascii-ui")

local base = ui.Color.from_hsl(174, 72, 56)        -- teal

base:lighten(0.2)     -- brighter variant
base:darken(0.2)      -- shadow variant
base:complement()     -- opposite hue
base:saturate(0.1)

-- A bar chart whose bars tint from dark (low) to light (high):
local function bar(value, max)
  local shade = base:lighten(value / max * 0.3)
  return ui.blocks.Segment({
    content = ("▮"):rep(math.floor(value / max * 10)),
    color = shade,
  }):wrap()
end
```

Colors are cached: identical `(fg, bg)` pairs return the same instance.

---

## Pattern 8 — Layered architecture for complex UIs

For UIs with real logic (clocks, charts, file trees), separate concerns into
four layers. Each layer can be tested independently.

```
┌─────────────────────────────┐
│  Component layer            │  owns state & hooks; calls render layer
├─────────────────────────────┤
│  Render layer               │  pure: value → BufferLine[]; no hooks
├─────────────────────────────┤
│  Logic layer                │  pure: raw data → computed values
├─────────────────────────────┤
│  Data layer                 │  reads external state (os.date, vim.fn, …)
└─────────────────────────────┘
```

```lua
-- data layer
local function get_time()
  local t = os.date("*t")
  return t.hour, t.min, t.sec
end

-- logic layer (pure — no ascii-ui, no Neovim API)
local function format_time(h, m, s)
  return string.format("%02d:%02d:%02d", h, m, s)
end

local function time_to_segments(h, m, s)
  -- returns a list of { char, color } cells for a visual clock face
  -- ...pure computation, fully unit-testable without mounting a window...
  return cells
end

-- render layer (pure — builds BufferLine[] from computed values)
local function render_clock(cells, width)
  -- convert cells table → BufferLine[]
  local lines = {}
  -- ... build BufferLine objects from cells ...
  return lines
end

-- component layer (owns state + hooks)
local ClockWidget = ui.createComponent("ClockWidget", function()
  local h, m, s   = get_time()
  local time, set = useState({ h = h, m = m, s = s })

  useInterval(function()
    local nh, nm, ns = get_time()
    set({ h = nh, m = nm, s = ns })
  end, 1000)

  return render_clock(time_to_segments(time.h, time.m, time.s), 40)
end)
```

**Benefit:** `time_to_segments` and `render_clock` are pure functions — write
unit tests for them with no Neovim instance needed.

---

## Pattern 9 — useEffect with cleanup (autocmd, LSP, external resource)

```lua
local useEffect = ui.hooks.useEffect

local AuditPanel = ui.createComponent("AuditPanel", function()
  local events, setEvents = useState({})

  -- Register an autocmd on mount; remove it on unmount.
  useEffect(function()
    local group = vim.api.nvim_create_augroup("AuditPanel", { clear = true })
    local id    = vim.api.nvim_create_autocmd("BufWritePost", {
      group    = group,
      callback = function(ev)
        setEvents(function(prev)
          local copy = vim.list_extend({}, prev)
          table.insert(copy, "saved: " .. ev.file)
          if #copy > 20 then table.remove(copy, 1) end  -- keep last 20
          return copy
        end)
      end,
    })

    return function()
      -- cleanup: remove the autocmd when the window closes
      pcall(vim.api.nvim_del_autocmd, id)
    end
  end, {})  -- {} = run once on mount

  return {
    Paragraph({ content = "Recent saves:" }),
    ui.map(events, function(e) return Paragraph({ content = e }) end),
  }
end)
```

---

## Pattern 10 — Side-by-side layouts with Row / Column

Compose `Row` and `Column` for dashboards, forms, and sidebar + main
layouts. `gap` spaces children without manual padding.

```lua
local ui       = require("ascii-ui")
local Row      = ui.layout.Row
local Column   = ui.layout.Column
local Box      = require("ascii-ui.components.box")
local Paragraph = ui.components.Paragraph
local Slider   = ui.components.Slider
local useState = ui.hooks.useState

local Dashboard = ui.createComponent("Dashboard", function()
  local volume, setVolume = useState(50)

  return {
    Paragraph({ content = "=== Dashboard ===" }),
    Row({
      children = {
        Column(
          Box({ width = 20, content = "CPU: 45%" }),
          Box({ width = 20, content = "RAM: 62%" })
        ),
        Column(
          Slider({ title = "Volume", value = volume, on_change = setVolume })
        ),
      },
      gap = 4,
    }),
  }
end)
```

Rules of thumb:
- `Row` is top-aligned: rows of different height start at the same line.
- `Column` is left-aligned.
- Nest freely; for dynamic child lists use `ui.map` as `children`.

---

## Pattern 11 — Rendering to stdout (headless / script mode)

Use `StdoutViewport` to render a UI to the terminal instead of a floating
window. Useful for CLI scripts, CI output, or animated terminal art.

```lua
local ui     = require("ascii-ui")
local Stdout = ui.viewports.StdoutViewport

local App = ui.createComponent("App", function()
  return {
    Paragraph({ content = "Hello from a headless script!" }),
  }
end)

ui.mount(App, Stdout.new())
```

ANSI truecolor codes are emitted automatically for any `color` fields on
`Segment` objects. The output resets color after each colored segment.

For tests or piped output, inject a custom writer:

```lua
local lines = {}
local viewport = Stdout.new(function(s) table.insert(lines, s) end)
ui.mount(App, viewport)
```

---

## Pattern 12 — Component testing with `ui.testing`

ascii-ui ships a React Testing Library-style harness. `render()` mounts a
component into an in-memory screen — no window, no Neovim UI — and you query
and interact with it by visible text.

```lua
-- tests/unit/components/counter_spec.lua
pcall(require, "luacov")

local ui      = require("ascii-ui")
local testing = require("ascii-ui.testing")

describe("Counter", function()
  it("increments on button press", function()
    local screen = testing.render(Counter)          -- no window opened

    assert.is_true(screen:hasText("Count: 0"))
    screen:select("+1")                              -- dispatch SELECT on the label
    assert.is_true(screen:hasText("Count: 1"))
  end)

  it("renders the expected frame", function()
    local screen = testing.render(Counter)
    assert.are.same({ "Count: 0", "[ +1 ]" }, screen:toLines())
  end)
end)
```

`screen` query / interaction API:

| Method | Purpose |
|---|---|
| `getByText(t)` / `getAllByText(t)` / `queryByText(t)` | Find segments by content |
| `hasText(t)` / `hasLine(l)` / `hasLines(ls)` | Assertions on rendered output |
| `hasHighlight(hl)` / `getByHighlight(hl)` | Assert highlight groups |
| `getFocusable()` / `getAllFocusable()` / `hasFocusable(t)` | Focus queries |
| `select(t)` | Fire `SELECT` on the segment with text `t` |
| `focus(t)` | Move focus to segment `t` |
| `trigger(t, interaction)` | Fire any interaction type (`"CURSOR_MOVE_RIGHT"`, …) |
| `toLines()` / `toSnapshot()` | Raw frame / snapshot string |

For real input (keypresses, cursor position), use the e2e helper:
`require("ascii-ui.testing.e2e").mount(Component)` with `screen:press("jj<CR>")`
and `screen:waitForText(...)`; unmount with `screen:unmount()`.

Keep pure render/logic layers separate (Pattern 8) so most tests don't even
need the harness; use `testing.render` for component-level behavior.

---

## Pattern 13 — Live-reload development loop

Iterate on a component without restarting Neovim: write the component in its
own file returning `createComponent(...)`, then mount it in debug mode with
an auto-reload watcher.

```lua
-- lua/myplugin/MyComp.lua
local ui = require("ascii-ui")
return ui.createComponent("MyComp", function()
  -- ...
end)
```

```lua
-- from any Neovim session (or a debug.lua + `make debug` in the plugin repo)
require("ascii-ui").debug("lua/myplugin/MyComp.lua")
```

Every save (`BufWritePost`) closes the current window and re-mounts the file.
Load/mount errors surface as notifications instead of crashing the session.
`ui.debug` accepts optional injected `loader` / `mounter` / `notifier` /
`watcher` overrides (dependency injection) for custom setups and tests.

---

## Cheat Sheet

| Task | How |
|---|---|
| Display text | `Paragraph({ content = "..." })` |
| Colored text | `Segment:new({ content = "...", color = "#hex" }):wrap()` |
| fg + bg | `Segment:new({ content = "...", color = { fg = "#000", bg = "#fff" } }):wrap()` |
| Computed colors | `ui.Color.from_hsl(h, s, l)`, `:lighten()`, `:complement()` |
| Theme-aware color | `Segment:new({ content = "...", highlight = "ErrorMsg" }):wrap()` |
| Click handler | `Button({ label = "x", on_press = fn })` |
| List of items | `ui.map(items, function(item) return ... end)` |
| Side-by-side | `Row(child1, child2)` / `Row({ children = ..., gap = 2 })` |
| Stacked | `Column(child1, child2)` |
| Boxed panel | `require("ascii-ui.components.box")({ width = 20, content = "hi" })` |
| Collapsible tree | `require("ascii-ui.components.tree")({ tree = node })` |
| Text field | `Input({ value = v, on_change = fn, on_submit = fn })` |
| Toggle visibility | `local x = cond and ComponentA() or ComponentB()` |
| Reactive value | `local v, setV = useState(init)` |
| Run on mount | `useEffect(fn, {})` |
| Run on change | `useEffect(fn, { dep })` |
| Timer | `useInterval(fn, ms)` |
| One-shot / auto-dismiss | `useTimeout(fn, ms)` (nil delay cancels) |
| Complex state | `useReducer(reducer, init)` |
| Two siblings share state | Lift state to parent; pass as props |
| Read user config | `useConfig()` |
| Headless render | `ui.mount(App, ui.viewports.StdoutViewport.new())` |
| Test a component | `require("ascii-ui.testing").render(Comp)` |
| Dev live-reload | `require("ascii-ui").debug("path/to/Comp.lua")` |
