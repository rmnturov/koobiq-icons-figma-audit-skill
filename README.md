# koobiq-icons-figma-audit

A Claude Code skill that audits a Figma icon component in the Koobiq icons file before it is exported via `figma:sync`.

## What it checks

| Rule | What is verified |
|------|-----------------|
| **A. Node type & name** | `COMPONENT` node, name matches the required format, size suffix equals actual geometry |
| **B. Geometry & layout** | Square, pixel-grid aligned, no auto-layout, clip content OFF |
| **C. Root structure** | One `shape` child (single-color) or `shape` + `shape-2` (two-color), no stray layers, `SCALE` constraints on both axes |
| **D. Fill color** | Primary shape bound to `Icon/Contrast` variable (same as sibling icons); secondary shape filled `#E21D03` |
| **E. Description metadata** | `kbq-{name}` line present and correct; tags sorted alphabetically; no extra blank lines or trailing whitespace; `<11:` interop entry matches `mapping-interop.json` |
| **F. Cross-variant consistency** | All size variants agree on structure, tags, description, and `<11:` old name |

The audit is **read-only by default** — it reports a pass/fail checklist. Fixes are only applied when explicitly requested.

## When to use

- Before running `yarn nx run @koobiq/icons:figma:sync`
- After drawing or editing an icon in Figma
- When you want to verify an icon is ready to export

## Usage

Invoke by name or describe what you want in natural language:

```
/koobiq-icons-figma-audit
```

```
проверь иконку floating-window
```

```
готова ли иконка к экспорту? https://www.figma.com/design/...?node-id=44731-6643
```

You can pass a Figma node URL, a node id, or just an icon name — the skill resolves all of them.

## Installation

Copy `SKILL.md` into your project's `.claude/skills/koobiq-icons-figma-audit/` directory:

```sh
mkdir -p .claude/skills/koobiq-icons-figma-audit
cp SKILL.md .claude/skills/koobiq-icons-figma-audit/
```

## Requirements

- [Claude Code](https://claude.ai/code) with the Figma MCP connected and authenticated
- `FIGMA_TOKEN` in `.env` with read access to the Koobiq icons file
