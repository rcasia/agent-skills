# ascii-ui Hook Reference

Hooks are called inside the body of a component function registered with
`ui.createComponent`. They must always be called **unconditionally** and
**in the same order** on every render — the same rule as React hooks.

```lua
local ui          = require("ascii-ui")
local useState    = ui.hooks.useState
local useEffect   = ui.hooks.useEffect
local useReducer  = ui.hooks.useReducer
local useInterval = ui.hooks.useInterval
local useTimeout  = ui.hooks.useTimeout
local useConfig   = ui.hooks.useConfig
```

---

## useState

Provides local reactive state. When the setter is called with a new value,
the component re-renders.

```lua
---@generic T
---@param initial T
---@return T value, fun(next: T | fun(prev: T): T) setter
local value, setValue = useState(initial)
```

**Returns:**
- `value` — a **deep copy** of the current state. Safe to read; mutating it
  has no effect.
- `setValue(next)` — schedules a re-render. Accepts a value or an updater
  function `fun(prev) -> next`.

```lua
local count, setCount = useState(0)

-- direct update
setCount(count + 1)

-- functional update (safe inside closures — avoids stale captures)
setCount(function(prev) return prev + 1 end)
```

### Gotchas

**Mutation is silently ignored.** `value` is a deep copy. Mutating it does
nothing — you must call `setValue`.

```lua
-- WRONG: mutation, no re-render
local list, setList = useState({ "a", "b" })
table.insert(list, "c")   -- mutates the copy, reconciler never sees it

-- RIGHT
setList(function(prev)
  local copy = vim.list_extend({}, prev)
  table.insert(copy, "c")
  return copy
end)
```

**No-op on equal values.** If `setValue` is called with the same primitive
value as the current state, the re-render is skipped. This is useful for
performance but can surprise you with table values — tables are compared by
reference, so a new table `{}` always triggers a re-render even if the
contents are identical.

---

## useEffect

Runs a side-effect function after the component mounts or when dependencies
change. Optionally returns a cleanup function that runs before the next
execution or on unmount.

```lua
---@param fn fun(): (fun() | nil)
---@param deps? any[]
useEffect(fn, deps)
```

| `deps` value | When `fn` runs |
|---|---|
| `{}` (empty table) | Once after first render (mount only) |
| `{ a, b }` | After first render, then whenever `a` or `b` change |
| `nil` (omitted) | After every render |

```lua
-- run once on mount
useEffect(function()
  vim.notify("window opened")
end, {})

-- run when query changes
useEffect(function()
  search(query)
end, { query })

-- run every render
useEffect(function()
  log("rendered")
end)
```

### Cleanup

Return a function to clean up resources. Called before the next run and on
unmount.

```lua
useEffect(function()
  local id = vim.api.nvim_create_autocmd("BufWritePost", {
    callback = function() reload() end,
  })
  return function()
    vim.api.nvim_del_autocmd(id)
  end
end, {})
```

### Gotchas

**Avoid `vim.api` in the effect without `vim.schedule`.** Effects run
synchronously after render in the Neovim event loop context. Heavy Neovim
API calls should be wrapped in `vim.schedule` if they may block.

**Dependencies are compared with `==`.** Tables are compared by reference.
If you pass a new table literal `{ a, b }` as a dep, it changes every render.
Pass primitive values (strings, numbers, booleans) as dependencies.

```lua
-- WRONG: new table every render → effect runs every render
useEffect(fn, { { a, b } })

-- RIGHT: pass primitives
useEffect(fn, { a, b })
```

---

## useReducer

Manages complex state with explicit transitions. Prefer over multiple
`useState` calls when state fields change together.

```lua
---@generic S, A
---@param reducer fun(state: S, action: A): S
---@param initial S
---@return S state, fun(action: A) dispatch
local state, dispatch = useReducer(reducer, initial)
```

```lua
local state, dispatch = useReducer(function(s, action)
  if action.type == "add" then
    local copy = vim.list_extend({}, s.items)
    table.insert(copy, action.item)
    return { items = copy, count = s.count + 1 }
  end
  if action.type == "clear" then
    return { items = {}, count = 0 }
  end
  return s
end, { items = {}, count = 0 })

-- in a callback:
dispatch({ type = "add", item = "new entry" })
dispatch({ type = "clear" })
```

The reducer must be a **pure function** — no side effects, always return a
new table (do not mutate `s`).

---

## useInterval

Runs `callback` repeatedly every `delay` milliseconds. The timer starts after
the first render and is automatically cleaned up on unmount or when `delay`
changes.

```lua
---@param callback fun()
---@param delay number | nil  milliseconds; nil or ≤0 disables the interval
useInterval(callback, delay)
```

```lua
local time, setTime = useState(os.time())

useInterval(function()
  setTime(os.time())
end, 1000)   -- tick every second
```

Pass `nil` or `0` as `delay` to disable the interval conditionally:

```lua
local running, setRunning = useState(true)

useInterval(function()
  tick()
end, running and 500 or nil)
```

### Gotchas

**Callback is a closure over the render-time scope.** If you read state
inside the callback, the value is captured at the time `useInterval` was
first called (when `delay` was first set). To always read the latest value,
use the functional setter form or combine with a `useEffect` that tracks the
value explicitly.

```lua
-- WRONG: `count` is stale inside the interval
local count, setCount = useState(0)
useInterval(function()
  setCount(count + 1)   -- count is always 0 here
end, 1000)

-- RIGHT: functional update reads fresh state
useInterval(function()
  setCount(function(prev) return prev + 1 end)
end, 1000)
```

---

## useTimeout

Runs `callback` once after `delay` milliseconds. Cleaned up on unmount.

```lua
---@param callback fun()
---@param delay number | nil  milliseconds; nil disables
useTimeout(callback, delay)
```

```lua
-- show a success message, then hide it after 2 seconds
local visible, setVisible = useState(false)

local function showMessage()
  setVisible(true)
end

useTimeout(function()
  setVisible(false)
end, visible and 2000 or nil)
```

---

## useConfig

Returns the user-supplied configuration table (the `opts` table from
`lazy.nvim` or `ui.setup(opts)`). Useful for reading user preferences inside
a component without passing config down as props.

```lua
local config = useConfig()
-- config is the table the user passed to setup()
```

---

## Hook Rules — Quick Checklist

```
[ ] Called at the top level of the component body
[ ] Not called inside if/else, loops, or pcall
[ ] Called in the same order on every render
[ ] No vim.api calls directly in the render body — wrap them in useEffect
[ ] Deps arrays contain primitives, not table literals
[ ] Interval/timeout callbacks use functional setters for state reads
[ ] Effects that allocate resources return a cleanup function
```
