# ascii-ui Component Reference

## Where components live

`ui.components` exposes the five most common components:

```lua
local ui        = require("ascii-ui")
local Paragraph = ui.components.Paragraph
local Button    = ui.components.Button
local Input     = ui.components.Input
local Select    = ui.components.Select
local Slider    = ui.components.Slider
```

`Tree`, `Box`, and `Checkbox` are built-in but must be required directly:

```lua
local Tree     = require("ascii-ui.components.tree")
local Box      = require("ascii-ui.components.box")
local Checkbox = require("ascii-ui.components.checkbox")
```

Layout primitives live on `ui.layout`:

```lua
local Row    = ui.layout.Row
local Column = ui.layout.Column
```

---

## Paragraph

Plain text block. The only component that accepts `\n` for multiple lines.
Not interactive.

```lua
Paragraph({ content = "Line one\nLine two" })
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `content` | `string` | yes | Supports `\n` for newlines |

**When to use:** labels, help text, any non-interactive display text.
**When NOT to use:** a single colored span — use a raw `Segment` instead.

---

## Button

Clickable, focusable label. Triggers `on_press` on the select key (`<CR>`
by default) or mouse click.

```lua
Button({
  label    = "Save",
  on_press = function() save_file() end,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `label` | `string` | yes | Displayed text |
| `on_press` | `fun()` | no | Called on activation |

**Gotcha:** `on_press` captures the closure at render time. If you read state
inside it, use the functional setter form (`setCount(function(prev) return prev + 1 end)`)
to avoid stale values.

---

## Input

Single-line editable text field. Supports **controlled** and **uncontrolled**
modes. Enter insert mode over it with `i`; `<CR>` submits; leaving insert
mode blurs.

```lua
local text, setText = useState("")

-- Controlled (parent owns the value)
Input({
  value       = text,
  placeholder = "Type something...",
  on_change   = setText,
  on_submit   = function(v) print("submitted: " .. v) end,
})

-- Uncontrolled (Input owns its state)
Input({
  initial_value = "prefill",
  on_change     = function(v) print("changed: " .. v) end,
  on_blur       = function(v) save(v) end,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `value` | `string?` | no | Controlled value; syncs when it changes |
| `initial_value` | `string?` | no | Seed for uncontrolled mode |
| `placeholder` | `string?` | no | Shown when empty (dimmed as content) |
| `on_change` | `fun(value: string)` | no | Fires on every text change |
| `on_submit` | `fun(value: string)` | no | Fires on `<CR>` in insert mode |
| `on_blur` | `fun(value: string)` | no | Fires when insert mode exits |
| `password` | `boolean?` | no | `true` masks text with `*` |

**Controlled vs uncontrolled:** pass `value` to drive it from parent state
(use `on_change` to sync back); pass `initial_value` and let the Input own
its state for simple forms.

---

## Select

Keyboard-navigable options list. Calls `on_select` with the chosen item.

```lua
Select({
  title     = "Choose a fruit:",
  options   = { "Apple", "Banana", "Cherry" },
  on_select = function(item) print("picked: " .. item) end,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `options` | `string[]` | yes | List of option labels |
| `title` | `string?` | no | Rendered as the first line |
| `on_select` | `fun(item: string)` | no | Called when user confirms |

**Behavior:** options render as `[x]` (selected) / `[ ]`; navigate with
hjkl/arrows, confirm with `<CR>`. First option is selected by default.
Options are snapshotted at mount — changing `options` later does not
re-render the list (TODO upstream).

---

## Slider

Horizontal 0–100 range control, steps of 10. Draggable with mouse; moves
with arrow keys (`CURSOR_MOVE_LEFT/RIGHT` interactions) once focused.

```lua
local volume, setVolume = useState(50)

Slider({
  title     = "Volume",
  value     = volume,
  on_change = setVolume,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `title` | `string?` | no | Rendered above the slider |
| `value` | `number?` | no | 0–100, default `0` |
| `on_change` | `fun(value: number)` | no | Called as the slider moves |

Visual: `────●───── 50%`.

---

## Checkbox

Toggle display. Renders `[x]` when active, `[ ]` when not.

```lua
local checked, setChecked = useState(false)

Checkbox({
  active = checked,
  label  = "Enable notifications",
})
-- toggling is done via a Button or interaction wired to setChecked
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `active` | `boolean` | no | Default `false` |
| `label` | `string` | no | Text shown beside the box |

**Note:** `Checkbox` is a display component — it does not own its state.
Drive `active` from a `useState` in the parent and toggle it with a `Button`
or a custom `Segment` interaction.

---

## Tree

Recursive, collapsible tree view. Children may be plain `TreeNode` tables
**or** already-constructed components (`FiberNode`s are rendered in place).

```lua
local Tree = require("ascii-ui.components.tree")

Tree({
  tree = {
    text = "./",
    expanded = true,
    children = {
      { text = "src", expanded = false, children = { { text = "main.lua" } } },
      { text = "README.md" },
    },
  },
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `tree` | `TreeNode` | yes | Root node |

| TreeNode field | Type | Notes |
|---|---|---|
| `text` | `string` | Label for this node |
| `children` | `TreeNode[] \| FiberNode[]?` | Absent/empty = leaf node |
| `expanded` | `boolean?` | Default `true`; `▸` collapsed, `▾` expanded |

**Behavior:** click or `<CR>` on a node toggles expand/collapse; branches
draw `├─` / `│` / `╰─` connectors. A child entry that is a component call
(e.g. `Button({ label = "add" })`) is rendered inline at that position —
useful for file explorers with actions.

---

## Box

Rounded box with centered text. Borders use `config.characters`.

```lua
local Box = require("ascii-ui.components.box")

Box({ width = 20, height = 5, content = "Hello!" })
-- ╭──────────────────╮
-- │                  │
-- │      Hello!      │
-- │                  │
-- ╰──────────────────╯
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `width` | `integer?` | no | Total width incl. borders, default `15` |
| `height` | `integer?` | no | Total height incl. borders, default `3` |
| `content` | `string?` | no | Centered text, default `""` |

---

## Layout — Row and Column

`Row` arranges children horizontally; `Column` stacks them vertically.
Both accept varargs or a props table with `gap`.

```lua
local Row    = ui.layout.Row
local Column = ui.layout.Column

-- varargs
Row(Button({ label = "OK" }), Button({ label = "Cancel" }))

-- props table with spacing
Row({
  children = {
    Paragraph({ content = "Name:" }),
    Input({ placeholder = "Enter name..." }),
  },
  gap = 2,
})
```

| Prop | Type | Default | Notes |
|---|---|---|---|
| `children` | `FiberNode[]` | `{}` | Child nodes to arrange |
| `gap` | `integer?` | `0` | Spaces (Row) or blank lines (Column) between children |

Rows are top-aligned, columns left-aligned. Nest freely for sidebar/main or
dashboard layouts; combine with `ui.map` for dynamic child lists.

---

## Keyboard model (floating window)

- Move focus with standard cursor keys: `h`/`j`/`k`/`l`, arrows
- `<CR>` — select/activate the focused segment (configurable)
- `i` — enter insert mode on input-able segments
- `q` — quit/close the window (configurable)
- Mouse: click to focus/select, `<LeftDrag>` moves the window

Defaults come from `ui.setup({ keymaps = { quit = "q", select = "<CR>" } })`.

---

## Low-level blocks — Segment and BufferLine

Use these when no built-in component fits, or when you need precise control
over color and layout within a single row.

### Segment

```lua
local Segment = require("ascii-ui.buffer.segment")

-- plain text
Segment:new({ content = "hello" })

-- hex color shorthand (foreground)
Segment:new({ content = "●", color = "#ff6b6b" })

-- fg + bg table
Segment:new({ content = " OK ", color = { fg = "#000000", bg = "#4caf50" } })

-- Color instance (see Color API below)
Segment:new({ content = "✨", color = ui.Color.from_hsl(280, 70, 60) })

-- theme highlight group
Segment:new({ content = "warning", highlight = "WarningMsg" })

-- focusable with a click handler
local interaction_type = require("ascii-ui.interaction_type")
Segment:new({
  content      = "[click me]",
  is_focusable = true,
  interactions = {
    [interaction_type.SELECT] = function() print("clicked") end,
  },
})
```

| Field | Type | Notes |
|---|---|---|
| `content` | `string` | No newlines allowed |
| `color` | `string \| {fg?, bg?} \| Color` | Truecolor; hex e.g. `"#ff0000"` |
| `highlight` | `string` | Neovim highlight group name |
| `is_focusable` | `boolean` | Keyboard-focusable when `true` |
| `interactions` | `table` | Map of `interaction_type` → `fun()` |

`color` and `highlight` are mutually exclusive per segment — use one or the other.

Available interaction types (`require("ascii-ui.interaction_type")`):
`SELECT`, `HOVER`, `CURSOR_MOVE_LEFT`, `CURSOR_MOVE_RIGHT`,
`CURSOR_MOVE_UP`, `CURSOR_MOVE_DOWN`, `INPUT`.

### BufferLine

A horizontal row of segments.

```lua
local BufferLine = require("ascii-ui.buffer.bufferline")

-- from one or more segments
BufferLine.new(
  Segment:new({ content = "Name: " }),
  Segment:new({ content = "Alice", color = "#4ecdc4" })
)

-- shorthand when you have a single segment
Segment:new({ content = "Hello" }):wrap()
```

`ui.blocks` offers constructor shortcuts for both: `ui.blocks.Segment(opts)`
and `ui.blocks.Bufferline(...)`.

### Color API

`ui.Color` unifies hex strings, `{fg, bg}` tables, and HSL:

```lua
local red    = ui.Color.new("#ff0000")               -- fg from hex string
local styled = ui.Color.new({ fg = "#ffffff", bg = "#333333" })
local hsl    = ui.Color.from_hsl(174, 72, 56)        -- h 0-360, s/l 0-100

hsl:lighten(0.1)   -- also :darken, :saturate, :desaturate, :complement
hsl:to_hsl()       -- round-trip back to h, s, l
```

Pass a `Color` (or raw hex/table) anywhere a segment accepts `color`.
Methods `:to_ansi()` and `:to_highlight_group()` are used internally by the
stdout viewport and the window renderer.

### `ui.map` — safe list rendering

Always use `ui.map` to render dynamic lists. It returns a flat array, which
is what the reconciler expects.

```lua
return {
  ui.map(items, function(item, index)
    return Paragraph({ content = index .. ". " .. item })
  end),
}
```

---

## Creating custom components

```lua
local MyWidget = ui.createComponent("MyWidget", function(props)
  -- props is the table passed at call site: MyWidget({ title = "hi" })
  return {
    Paragraph({ content = props.title }),
  }
end, {
  -- optional prop type declarations (validated at call time)
  title = "string",
})

-- use it like any built-in:
MyWidget({ title = "Hello" })
```

Supported forms:

```lua
-- named (recommended) + flat prop types
ui.createComponent("Name", fn, { title = "string" })

-- anonymous (name defaults to "anonymous")
ui.createComponent(fn)

-- extended format — prop types + layout
ui.createComponent("Name", fn, {
  props  = { text = "string" },
  layout = { direction = "row" },
})
```

Valid prop type strings: `"string"`, `"number"`, `"boolean"`, `"function"`,
`"table"`, `"nil"`. Invalid props raise an error at call time.

The name string must be unique across the plugin — it is used as the fiber
node type for reconciliation. Duplicate names produce a log error and the
second registration is ignored.
