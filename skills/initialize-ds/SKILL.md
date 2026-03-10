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
2. Call `get_variable_defs(fileKey)` to retrieve design tokens (colors, spacing, typography, shadow variables).
3. Call `get_design_context` for top-level component frames to understand naming and grouping.

Map out the design system parts and sub-parts from the structure discovered.

## Phase 3 — Propose architecture

Present the proposed folder/file structure to the user. Always follow this standard layout:

```
[project-root]/
├── design-system.md
├── ds-usage.md
└── design-system/
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
| [part-name] | [tokens.css](design-system/part-name/tokens.css) |

## Usage Rules

See [ds-usage.md](ds-usage.md) for usage rules, breakpoints, anti-patterns, and accessibility guidelines.
```

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
```

Explain which files are safe to edit manually:
- `ds-usage.md` — fully manual, edit freely
- Component `.md` files — add notes inside `<!-- custom --> … <!-- /custom -->` blocks; these are preserved on `/update-ds`
- `tokens.css` files — **do not edit manually**, regenerated in full on every `/update-ds`
- `design-system.md` — **do not edit manually**, managed by the plugin

Ask the user to review and confirm the generated structure looks correct before continuing.

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
