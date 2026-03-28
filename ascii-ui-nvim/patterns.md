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
    Segment:new({ content = time, color = { fg = "#4ecdc4" } }):wrap(),
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
-- SearchInput is a child that fires onChange
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

When you need a row where each character or word has a different color, build
it manually with `Segment` and `BufferLine`.

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
    Segment:new({ content = "[" .. mode .. "]", color = { fg = mode_colors[mode] or "#ffffff" } }),
    Segment:new({ content = " " .. filename .. "  " }),
    Segment:new({ content = "col:" .. col .. " line:" .. line, color = { fg = "#8b949e" } })
  )
end

local Statusbar = ui.createComponent("Statusbar", function(props)
  return { make_statusbar(props.mode, props.filename, props.col, props.line) }
end)
```

---

## Pattern 7 — Layered architecture for complex UIs

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

## Pattern 8 — useEffect with cleanup (autocmd, LSP, external resource)

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

## Pattern 9 — Rendering to stdout (headless / script mode)

Use `StdoutViewport` to render a UI to the terminal instead of a floating
window. Useful for CLI scripts, CI output, or animated terminal art.

```lua
local ui       = require("ascii-ui")
local Stdout   = ui.viewports.StdoutViewport

local App = ui.createComponent("App", function()
  return {
    Paragraph({ content = "Hello from a headless script!" }),
  }
end)

ui.mount(App, Stdout.new())
```

ANSI truecolor codes are emitted automatically for any `color` fields on
`Segment` objects. The output resets color after each colored segment.

---

## Cheat Sheet

| Task | How |
|---|---|
| Display text | `Paragraph({ content = "..." })` |
| Colored text | `Segment:new({ content = "...", color = { fg = "#hex" } }):wrap()` |
| Theme-aware color | `Segment:new({ content = "...", highlight = "ErrorMsg" }):wrap()` |
| Click handler | `Button({ label = "x", on_press = fn })` |
| List of items | `ui.map(items, function(item) return ... end)` |
| Toggle visibility | `local x = cond and ComponentA() or ComponentB()` |
| Reactive value | `local v, setV = useState(init)` |
| Run on mount | `useEffect(fn, {})` |
| Run on change | `useEffect(fn, { dep })` |
| Timer | `useInterval(fn, ms)` |
| Complex state | `useReducer(reducer, init)` |
| Two siblings share state | Lift state to parent; pass as props |
| Read user config | `useConfig()` |
