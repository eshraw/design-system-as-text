---
name: initialize-ds
description: Initializes a design system from Figma into structured markdown and CSS files. Use when user says "initialize design system", "set up DS", "import from Figma", or runs /initialize-ds.
metadata:
  mcp-server: figma
---

Initialize a design system from Figma into structured markdown and CSS files. Follow these phases exactly.

## Phase 1 — Gather Figma source

Ask the user for Figma file URL(s). They may provide multiple (e.g. one for tokens, one for components).

For each URL, parse:
- `fileKey`: the segment after `/design/` (e.g. `figma.com/design/ABC123/...` → `ABC123`)
- `nodeId`: the `node-id` query param, converting `-` to `:` (e.g. `1-2` → `1:2`)

Ask: "Would you like to scope to a specific frame/page, or import the whole file?"

## Phase 2 — Explore the Figma file

For each fileKey provided:

1. Call `get_metadata(fileKey)` to discover pages, top-level frames, and component structure.
2. Call `get_variable_defs(fileKey)` to retrieve design tokens (colors, spacing, typography, shadow variables). **If the result is empty or an error, note this — Phase 5.5 will handle token recovery.**
3. Call `get_design_context` for top-level component frames to understand naming and grouping.

Map out the design system parts and sub-parts from the structure discovered.

## Phase 3 — Propose architecture

Present the proposed folder/file structure to the user. Always follow this standard layout:

```
[project-root]/
├── design-system.md
├── ds-usage.md
└── design-system/
    ├── tokens.json
    ├── [part-name]/
    │   ├── [part-name].md
    │   ├── [component].md
    │   ├── tokens.css
    │   └── [sub-part]/
    │       ├── [sub-part].md
    │       ├── [component].md
    │       └── tokens.css
```

Store the Figma source URL(s) and current file version in `design-system.md` frontmatter for future `/update-ds` use.

Ask the user to confirm or adjust naming before proceeding.

## Phase 4 — Generate files

Create all files in the confirmed structure.

### `design-system.md` (root index)

```markdown
---
figma_sources:
  - url: [FIGMA_URL]
    file_key: [FILE_KEY]
    last_synced_version: [VERSION]
    last_synced_at: [ISO_DATE]
generated_at: [ISO_DATE]
---

# Design System

> Auto-generated index. Do not edit manually — run `/update-ds` to sync changes.

## Parts

| Part | Description | Files |
|------|-------------|-------|
| [part-name] | [description] | [part-name/part-name.md](design-system/part-name/part-name.md) |

## Token Files

| Scope | File |
|-------|------|
| All (raw JSON export) | [tokens.json](design-system/tokens.json) |
| [part-name] | [tokens.css](design-system/part-name/tokens.css) |

## Usage Rules

See [ds-usage.md](ds-usage.md) for usage rules, breakpoints, anti-patterns, and accessibility guidelines.
```

### `tokens.json` (root token export)

Write a single `design-system/tokens.json` containing the raw token data from `get_variable_defs`. Structure it as:

```json
{
  "_meta": {
    "source": "[FIGMA_URL]",
    "version": "[VERSION]",
    "synced_at": "[ISO_DATE]"
  },
  "[collection-name]": {
    "[mode-name]": {
      "[token-name]": {
        "value": "[resolved-value]",
        "type": "color | spacing | typography | shadow | number | string",
        "description": "[description if present]"
      }
    }
  }
}
```

Rules:
- Use camelCase for collection and token names to keep JSON idiomatic.
- Include all modes (e.g. `light`, `dark`) as sibling keys under each collection.
- Resolve aliases to their final values where possible; if a token references another token, include both the `value` (resolved) and an `alias` field pointing to the source token path.
- This file is **not edited manually** — it is regenerated in full on every `/update-ds`.

### `tokens.css` files

Populate with CSS custom properties from Figma variable definitions:

```css
/* Auto-generated from Figma. Do not edit manually. */
/* Source: [FIGMA_URL] | Version: [VERSION] | Synced: [ISO_DATE] */

:root {
  /* [Collection Name] */
  --[token-name]: [value];
}
```

Group tokens by their Figma collection. Use kebab-case for variable names, prefixed by collection name where appropriate.

### Component `.md` files

```markdown
# [Component Name]

<!-- custom -->
<!-- /custom -->

## Overview

[Description from Figma]

## Variants

| Variant | Description |
|---------|-------------|
| [variant] | [description] |

## Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| [prop] | [type] | [default] | [description] |

## Usage

[Usage guidance from Figma annotations or inferred from component structure]

## Do / Don't

- Do: [positive pattern]
- Don't: [anti-pattern]
```

### Part `.md` files

```markdown
# [Part Name]

[Description of this design system part]

## Components

| Component | File |
|-----------|------|
| [Component] | [[Component].md]([Component].md) |

## Sub-parts

| Sub-part | File |
|----------|------|
| [Sub-part] | [[sub-part]/[sub-part].md]([sub-part]/[sub-part].md) |

## Tokens

See [tokens.css](tokens.css) for design tokens scoped to this part.
```

Show progress to the user as each file is created.

## Phase 5 — User validation

Present a full file tree of everything created, using the actual resolved paths relative to the project root. Format it exactly like this example — the user must be able to copy any path and open it directly:

```
Design system initialized. Here's what was created:

design-system.md                              ← master index (Figma source + version stored here)
ds-usage.md                                   ← usage rules (edit freely)
design-system/
  tokens.json                                 ← raw token export (all collections + modes, do not edit)
  [part-name]/
    [part-name].md                            ← part overview
    tokens.css                                ← design tokens for this part
    [ComponentName].md                        ← component docs (add notes inside <!-- custom --> blocks)
    [sub-part]/
      [sub-part].md
      tokens.css
      [ComponentName].md

Files created: [N total]
  [N] part/sub-part index files (.md)
  [N] component files (.md)
  [N] token files (.css)
  1 token export (tokens.json)
```

Explain which files are safe to edit manually:
- `ds-usage.md` — fully manual, edit freely
- Component `.md` files — add notes inside `<!-- custom --> … <!-- /custom -->` blocks; these are preserved on `/update-ds`
- `tokens.css` files — **do not edit manually**, regenerated in full on every `/update-ds`
- `design-system/tokens.json` — **do not edit manually**, regenerated in full on every `/update-ds`
- `design-system.md` — **do not edit manually**, managed by the plugin

Ask the user to review and confirm the generated structure looks correct before continuing.

## Phase 5.5 — Token recovery (run only if `tokens.json` was NOT created)

If `get_variable_defs` returned no data, or the user's Figma plan does not expose variables, `tokens.json` will be missing. In that case, do not silently skip it — enter the following recovery conversation instead.

### Step A — Diagnose and inform

Tell the user:

> "I wasn't able to export design tokens from Figma automatically (variables may not be published, or your plan may not include the Variables API). Let's get your tokens in another way so `tokens.json` can be created."

Then ask:

> "Do you have an existing token file I can import? For example: a `tokens.json` from Style Dictionary, Token Studio, Figma Tokens plugin, or any JSON export. Or would you prefer to define them together now?"

### Step B — Branch on user response

**If the user provides a file path or pastes JSON:**

1. Read or accept the token data.
2. Detect the format:
   - **Token Studio / Figma Tokens** — groups at the top level, values under `$value` / `value` keys.
   - **Style Dictionary** — nested category → type → item with `value` keys.
   - **Raw / flat** — flat key-value pairs.
   - **W3C Design Tokens** — `$value` and `$type` on each token.
3. Normalize to the standard `tokens.json` shape:
   ```json
   {
     "_meta": {
       "source": "user-provided",
       "imported_at": "[ISO_DATE]",
       "original_format": "[detected-format]"
     },
     "[collection-name]": {
       "default": {
         "[token-name]": {
           "value": "[resolved-value]",
           "type": "color | spacing | typography | shadow | number | string"
         }
       }
     }
   }
   ```
4. Write `design-system/tokens.json`.
5. Confirm: "Imported [N] tokens across [M] collections from your file."

**If the user wants to define tokens interactively:**

Ask the following questions one at a time, building the token map as you go:

1. "What color tokens do you use? List them as `name: value` pairs (e.g. `primary: #0052CC`)."
2. "What spacing tokens do you use? (e.g. `spacing-sm: 4px, spacing-md: 8px`)"
3. "What typography tokens do you use? (e.g. font families, sizes, weights)"
4. "Any other token categories — shadows, border radii, z-index, motion?"
5. "Do you use multiple themes or modes (e.g. light/dark)? If so, give values per mode."

After each answer, confirm the tokens back to the user before moving on. When all categories are collected, write `design-system/tokens.json` using the standard shape above.

**If the user wants to skip tokens entirely:**

Acknowledge and write a placeholder file so downstream tooling doesn't break:

```json
{
  "_meta": {
    "source": "placeholder",
    "note": "No tokens were imported. Run /update-ds or replace this file manually.",
    "generated_at": "[ISO_DATE]"
  }
}
```

Tell the user: "A placeholder `tokens.json` has been created. You can populate it later by running `/update-ds` once variables are published in Figma, or by replacing the file with your own token export."

## Phase 6 — Create ds-usage.md via interview

Ask these questions one at a time, waiting for answers:

1. "What are the primary breakpoints and responsive behaviors?"
2. "Are there rules for combining components — e.g. 'never use X inside Y'?"
3. "Are there accessibility requirements or patterns (ARIA roles, focus order, color contrast)?"
4. "What are common anti-patterns to avoid when using this design system?"
5. "Are there brand or tone guidelines for copy within components (labels, placeholders, error messages)?"
6. "Are there any deprecated components still present in Figma that should not be used?"

Generate `ds-usage.md` from the answers:

```markdown
# Design System Usage Rules

> Maintained manually. Run `/initialize-ds` to regenerate from scratch, or edit directly.

## Breakpoints & Responsive Behavior

[Answer 1]

## Component Composition Rules

[Answer 2]

## Accessibility

[Answer 3]

## Anti-patterns

[Answer 4]

## Brand & Copy Guidelines

[Answer 5]

## Deprecated Components

[Answer 6]
```
