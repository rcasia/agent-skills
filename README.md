# rcasia/agent-skills

Agent skills for [ascii-ui.nvim](https://github.com/ascii-ui/ascii-ui.nvim) — a React-inspired UI framework for Neovim plugins.

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
- `SKILL.md` — context, design rules, change workflow, final checklist
- `components.md` — built-in components, layout (`Row`/`Column`), Color API, low-level blocks
- `hooks.md` — hook reference with gotchas and dependency semantics
- `patterns.md` — 13 recipes (clock, lists, conditional rendering, lifted state, useReducer, colored rows, Color API, layered architecture, autocmd cleanup, layouts, stdout, component testing, live reload)

**Install:**

```bash
npx skills add rcasia/agent-skills --skill ascii-ui-nvim
```

## Keeping the skill in sync

`SKILL.md` frontmatter records the upstream state it documents:

```yaml
metadata:
  source_repo: https://github.com/ascii-ui/ascii-ui.nvim
  source_commit: a88bbb033eeb70e058454797cb96ddf6f15aa611   # last synced commit
  source_commit_date: "2026-08-14"
  skill_synced: "2026-09-05"                                # date of the sync
```

To update: diff `lua/ascii-ui/` and `docs/` in [ascii-ui.nvim](https://github.com/ascii-ui/ascii-ui.nvim)
since `source_commit`, revise the reference files, then bump `source_commit`
and `skill_synced` in the same commit. See "Staying current" in `SKILL.md` for
the exact commands.

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
