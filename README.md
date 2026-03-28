# rcasia/agent-skills

Agent skills for [ascii-ui.nvim](https://github.com/rcasia/ascii-ui.nvim) — a React-inspired UI framework for Neovim plugins.

Skills follow the [skills.sh](https://skills.sh/) format and work with any compatible AI agent (OpenCode, Claude Code, Cursor, Copilot, Gemini CLI, and more).

## Available Skills

### ascii-ui-nvim

Building Neovim plugin UIs with ascii-ui.nvim. Covers the component model,
hooks, best practices, and common patterns for plugin authors.

**Use when:**
- Writing a Neovim plugin that needs a floating window UI
- Adding interactive widgets (buttons, inputs, sliders) to a plugin
- Animating or live-updating a terminal window
- Asking how to use `useState`, `useInterval`, `ui.mount`, or any ascii-ui API

**Contents:**
- `SKILL.md` — mental model, quick start, best practices, anti-patterns
- `components.md` — built-in component reference with props and examples
- `hooks.md` — hook reference with gotchas and dependency semantics
- `patterns.md` — 9 recipes (timer/clock, list rendering, conditional rendering, lifted state, useReducer, colored rows, layered architecture, autocmd cleanup, stdout mode)

**Install:**

```bash
npx skills add rcasia/agent-skills --skill ascii-ui-nvim
```

## Installation

Install all skills in this repo:

```bash
npx skills add rcasia/agent-skills
```

Install a specific skill:

```bash
npx skills add rcasia/agent-skills --skill ascii-ui-nvim
```

## License

MIT
