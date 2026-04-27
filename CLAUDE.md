# CLAUDE.md — Figma Design System Automation

## What This Is

A Python + JavaScript bridge that automates Figma via the **Scripter** community plugin.
Playwright drives a persistent Firefox profile; user code is injected into Monaco, executed
in Scripter's sandbox, and results are returned over the `console.log` sentinel protocol.

The goal is a **project-agnostic pipeline**: given any codebase (Flutter/Dart, Tailwind,
CSS variables, token JSON), extract design tokens at 3 levels (primitives → semantic →
component) plus typography, then push them into Figma as Variables and Text Styles with
full alias chains.

---

## Skills

### When to load

| Task | Skills to load |
|------|---------------|
| Writing any sync `*.js` script | `figma-use` + `figma-generate-library` |
| Designing token architecture / naming | `figma-create-design-system-rules` |
| Implementing parsers (`parsers/*.py`) | `figma-create-design-system-rules` |
| Any Plugin API JS at all | `figma-use` (always, no exceptions) |

### Skill locations

```
.claude/skills/figma-use/SKILL.md
.claude/skills/figma-use/references/variable-patterns.md
.claude/skills/figma-use/references/working-with-design-systems/wwds-variables--creating.md
.claude/skills/figma-use/references/working-with-design-systems/wwds-text-styles.md
.claude/skills/figma-use/references/gotchas.md

.claude/skills/figma-generate-library/SKILL.md
.claude/skills/figma-generate-library/references/token-creation.md
.claude/skills/figma-generate-library/scripts/createVariableCollection.js
.claude/skills/figma-generate-library/scripts/createSemanticTokens.js
.claude/skills/figma-generate-library/scripts/validateCreation.js
.claude/skills/figma-generate-library/scripts/rehydrateState.js

.claude/skills/figma-create-design-system-rules/SKILL.md
```

### Critical Plugin API rules (from figma-use skill)

These apply to every `.js` file in `scripts/variables/`:

- Colors are **0–1 range**, not 0–255: `{r: 1, g: 0, b: 0}` = red
- Use `return` for output — `console.log` is never returned to the agent
- **Never use** `figma.notify()` — throws "not implemented"
- **Never use** `getPluginData()` / `setPluginData()` — use `getSharedPluginData()` instead
- Font MUST be loaded before any text write: `await figma.loadFontAsync({family, style})`
- `setBoundVariableForPaint` returns a **NEW** paint — must capture and reassign
- Always set `variable.scopes` explicitly — never leave as `ALL_SCOPES`
- Web code syntax MUST use `var()` wrapper: `var(--color-bg-primary)`, not `--color-bg-primary`
- **Return ALL created/mutated node IDs** in every script that touches the canvas
- **Never parallelize** — all `use_figma` / Scripter calls must be strictly sequential

### Helper scripts

When implementing sync JS scripts, **embed and adapt** these rather than writing from scratch:

```
.claude/skills/figma-generate-library/scripts/createVariableCollection.js  → basis for collection creation
.claude/skills/figma-generate-library/scripts/createSemanticTokens.js      → basis for sync_semantic_colors.js
.claude/skills/figma-generate-library/scripts/validateCreation.js          → basis for post-sync validation
.claude/skills/figma-generate-library/scripts/rehydrateState.js            → basis for resume/idempotency
```

---

## Architecture

```
run.py                 CLI entry (Typer); top-level commands + re-exports siblings for test stability
protocol.py            v2 wire format: Pydantic models, sentinel constants, _reassemble_chunks
transport.py           Playwright + Scripter: launch Firefox, find iframe, inject wrapper, collect sentinels
host_io.py             Code resolution (--code / --code-file / stdin), atomic writes, logging
wrapper.js             Static JS template injected into Scripter sandbox; includes polyfills
read_handlers.py       8 read-only Figma query commands
sync_handlers.py       Write-side commands: dry-run gate, validation, token sync (extend here)

parsers/               ← NEW: one parser per source type, all output standard TokenGraph
  base.py              Abstract TokenParser class + shared token JSON schema
  flutter_material.py  Reads Dart files, Material 3 ThemeData + ColorScheme
  tailwind.py          Reads tailwind.config.js / tailwind.config.ts
  css_variables.py     Reads .css / .scss files with custom properties
  token_json.py        Reads design-tokens JSON / Style Dictionary format

scripts/variables/     JS scripts injected into Scripter via the bridge
  sync_primitive_colors_normalized.js   ← existing, do not modify
  validate_runtime_context.js           ← existing, do not modify
  sync_semantic_colors.js               ← NEW
  sync_component_tokens.js              ← NEW
  sync_text_styles.js                   ← NEW

tokens/                Working token files (gitignored)
  primitives.json
  semantic.json
  components.json
  text_styles.json
  primitives.normalized.json

profile/               Firefox persistent profile — gitignored
```

---

## Standard Token Schema

All parsers output this exact shape. The sync scripts consume it.

```json
// tokens/primitives.json
{
  "color": { "purple": { "500": { "value": "#673AB7", "type": "color" } } },
  "spacing": { "sm": { "value": 8, "type": "spacing" } },
  "radius": { "md": { "value": 8, "type": "radius" } },
  "font": { "size": { "base": { "value": 16, "type": "fontSize" } } }
}

// tokens/semantic.json
{
  "color/background/primary": { "value": "{color.purple.500}", "type": "color" },
  "color/text/primary":       { "value": "{color.neutral.900}", "type": "color" }
}

// tokens/components.json
{
  "button/background/default": { "value": "{color/background/primary}", "type": "color" },
  "button/label/default":      { "value": "{color/text/on-primary}", "type": "color" }
}

// tokens/text_styles.json
[
  {
    "name": "heading/h1",
    "fontFamily": "Inter",
    "fontSize": 32,
    "fontWeight": 700,
    "lineHeight": 40,
    "letterSpacing": -0.5
  }
]
```

---

## CLI Commands

```bash
# Setup (one-time)
python -m venv .venv && source .venv/bin/activate
pip install -e '.[dev]'
playwright install firefox
cp .env.example .env        # set FIGMA_FILE_URL

# Login (one-time)
python run.py login

# Smoke test
python run.py hello -m "bridge alive"

# Parse any repo → tokens/
python run.py parse --source /path/to/any/repo --out ./tokens/

# Sync (always dry-run first)
python run.py sync primitive-colors-normalized --dry-run
python run.py sync primitive-colors-normalized
python run.py sync semantic-colors --tokens tokens/semantic.json --dry-run
python run.py sync semantic-colors --tokens tokens/semantic.json
python run.py sync component-tokens --tokens tokens/components.json --dry-run
python run.py sync component-tokens --tokens tokens/components.json
python run.py sync text-styles --tokens tokens/text_styles.json --dry-run
python run.py sync text-styles --tokens tokens/text_styles.json

# Full pipeline (parse → validate → dry-run → confirm → sync all)
python run.py pipeline --source /path/to/any/repo

# Tests
pytest tests/ -v
```

---

## Command Conventions

| Prefix | Meaning |
|--------|---------|
| `read_*`     | Read-only, no side effects |
| `sync_*`     | Create or update, must be idempotent |
| `validate_*` | Safety and consistency checks |
| `parse_*`    | Source code → token JSON |

- Prefer these prefixes over free-form instructions
- Do not invent new prefixes
- Validate before sync when possible
- Every sync command must support `--dry-run`

---

## Token-Saving Rules

- Read only the files needed for the task. Prefer `Grep` over `Read` for locating symbols.
- Use `offset`/`limit` when reading large files.
- Default model: **Sonnet 4.6**. No extended thinking unless explicitly requested.
- Avoid re-reading files unless necessary (file changed or context may be stale).
- Avoid creating docs unless they provide long-term value.
- Keep CLI behavior stable unless explicitly changing it.
- Prefer small, incremental changes over large rewrites.
- Do not perform any Git operations. Do not create branches, commits, or pull requests.
- Focus only on code and runtime behavior.
- Avoid workflow or process suggestions unless explicitly asked.

---

## Critical Patterns

### Transport (do not modify)
`wrapper.js` uses marker substitution at runtime (`RID`, sentinels, inline cap, chunk bytes).
User code goes at `/*__USER_JS__*/`. Scripter sandbox lacks `TextEncoder`, `btoa`,
`crypto.subtle` — wrapper includes polyfills. **Never remove them.**

### Chunked transport
Wrapper emits `BEGIN` + N base64 chunks via `console.log`:
```
__FS::RID:C:i:<base64>::SF__
```
`_reassemble_chunks` validates transport SHA256 before decoding.

### Inline vs file mode
Determined by `inline_cap` in `_bridge_exec`. 500 B → inline, 65536 B → file.

### Parser auto-detection priority
When multiple parsers match a repo, use this order (most explicit wins):
```
token_json > tailwind > css_variables > flutter_material
```

### Idempotency in sync scripts
Before creating any variable or style, check if it already exists by name.
Skip if exists and value unchanged. Update if value changed. Never duplicate.
Use `getSharedPluginData()` / `setSharedPluginData()` for tagging — NOT `getPluginData()`.

### Graceful degradation
If a parser finds no typography data → `text_styles.json` is `[]` → sync skips silently.
If semantic.json is empty → component sync still runs (may produce orphaned aliases — warn user).

---

## Environment

```
FIGMA_FILE_URL=https://www.figma.com/file/...   # required
PROFILE_DIR=./profile                            # default
```

---

## What NOT to Do

- Do not modify `transport.py`, `protocol.py`, `wrapper.js` — they are complete
- Do not use `figma.notify()` in any JS script — it throws
- Do not use `ALL_SCOPES` on any variable — always set specific scopes
- Do not write large sync scripts from scratch — use helper scripts from `.claude/skills/`
- Do not run multiple sync commands in parallel — always sequential
- Do not perform any Git operations
- Do not create docs unless they provide long-term value
