# ascii-ui Component Reference

All built-in components live on `ui.components`. Most accept a single props
table. Components are focusable / interactive unless noted otherwise.

```lua
local ui        = require("ascii-ui")
local Paragraph = ui.components.Paragraph
local Button    = ui.components.Button
local Select    = ui.components.Select
local Checkbox  = ui.components.Checkbox
local Input     = ui.components.Input
local Slider    = ui.components.Slider
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

Clickable, focusable label. Triggers `on_press` on `<CR>` or mouse click.

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

## Select

Keyboard-navigable options list. Calls `on_select` with the chosen item.

```lua
Select({
  options   = { "Apple", "Banana", "Cherry" },
  on_select = function(item) print("picked: " .. item) end,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `options` | `string[]` | yes | List of option labels |
| `on_select` | `fun(item: string)` | no | Called when user confirms |

**Navigation:** `j`/`k` or arrow keys move the cursor; `<CR>` confirms.

---

## Checkbox

Toggle with an optional label. Renders `[x]` when active, `[ ]` when not.

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
or mouse click on the checkbox itself.

---

## Input

Single-line editable text field. Calls `on_change` on every keystroke.

```lua
local text, setText = useState("")

Input({
  value     = text,
  on_change = setText,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `value` | `string` | no | Current text content |
| `on_change` | `fun(value: string)` | no | Called on every character change |

---

## Slider

Horizontal 0–100 range control. Draggable with mouse; `h`/`l` with keyboard.

```lua
local volume, setVolume = useState(50)

Slider({
  value     = volume,
  on_change = setVolume,
})
```

| Prop | Type | Required | Notes |
|---|---|---|---|
| `value` | `number` | no | 0–100, default `0` |
| `on_change` | `fun(value: number)` | no | Called as the slider moves |

---

## Low-level blocks — Segment and BufferLine

Use these when no built-in component fits, or when you need precise control
over color and layout within a single row.

### Segment

```lua
local Segment = require("ascii-ui.buffer.segment")

-- plain text
Segment:new({ content = "hello" })

-- hex foreground color
Segment:new({ content = "●", color = { fg = "#ff6b6b" } })

-- hex background
Segment:new({ content = " OK ", color = { fg = "#000000", bg = "#4caf50" } })

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
| `color` | `{ fg?: string, bg?: string }` | Hex strings e.g. `"#rrggbb"` |
| `highlight` | `string` | Neovim highlight group name |
| `is_focusable` | `boolean` | Keyboard-focusable when `true` |
| `interactions` | `table` | Map of `interaction_type` → `fun()` |

`color` and `highlight` are mutually exclusive per segment — use one or the other.

### BufferLine

A horizontal row of segments.

```lua
local BufferLine = require("ascii-ui.buffer.bufferline")

-- from one or more segments
BufferLine.new(
  Segment:new({ content = "Name: " }),
  Segment:new({ content = "Alice", color = { fg = "#4ecdc4" } })
)

-- shorthand when you have a single segment
Segment:new({ content = "Hello" }):wrap()
```

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

The name string must be unique across the plugin — it is used as the fiber
node type for reconciliation. Duplicate names produce a log error and the
second registration is ignored.
