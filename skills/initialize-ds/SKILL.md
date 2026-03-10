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

Then use the `AskUserQuestion` tool with this question and these choices:

- question: "How would you like to provide your design tokens?"
- choices:
  - "A) Import an existing file — I have a tokens.json, Token Studio export, Style Dictionary output, or similar"
  - "B) Define them here — I'll type my tokens and we'll build the file together"
  - "C) Skip for now — create a placeholder I can fill in later"

Wait for the user to pick A, B, or C before continuing.

### Step B — Branch on user response

**If the user picks A (import file):**

Use the `AskUserQuestion` tool:

- question: "What format is your token file?"
- choices:
  - "1) Token Studio / Figma Tokens plugin"
  - "2) Style Dictionary"
  - "3) W3C Design Tokens (DTCG format)"
  - "4) Raw / flat JSON (simple key-value pairs)"
  - "5) Not sure — paste it and I'll detect the format"

Then use the `AskUserQuestion` tool to ask the user to provide the file path or paste the JSON content (free text, no choices).

1. Read or accept the token data.
2. Normalize to the standard `tokens.json` shape based on the detected/chosen format:
   - **Token Studio / Figma Tokens** — groups at top level, values under `$value` / `value` keys.
   - **Style Dictionary** — nested category → type → item with `value` keys.
   - **W3C / DTCG** — `$value` and `$type` on each token.
   - **Raw / flat** — flat key-value pairs, infer types from values.
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
3. Write `design-system/tokens.json`.
4. Confirm: "Imported [N] tokens across [M] collections from your file."

**If the user picks B (define interactively):**

Ask the following questions one at a time using the `AskUserQuestion` tool. Wait for confirmation after each before moving on.

**Question 1 — Colors**

- question: "Which color token categories do you want to define? (select all that apply)"
- choices:
  - "A) Brand / primary palette"
  - "B) Neutral / grey scale"
  - "C) Semantic (success, warning, error, info)"
  - "D) Surface / background colors"
  - "E) Other / all at once"

Then use `AskUserQuestion` (free text) to ask: "List your color tokens as name: value pairs (e.g. primary: #0052CC)."

**Question 2 — Spacing**

- question: "Do you use a spacing scale?"
- choices:
  - "A) Yes, a fixed scale (e.g. 4px, 8px, 16px…) — I'll list the steps"
  - "B) Yes, named tokens (e.g. spacing-sm, spacing-lg) — I'll define each"
  - "C) No dedicated spacing tokens"

If A or B, follow up with `AskUserQuestion` (free text): "List your spacing tokens."

**Question 3 — Typography**

- question: "Which typography tokens do you want to define?"
- choices:
  - "A) Font families only"
  - "B) Font sizes only"
  - "C) Font weights only"
  - "D) All of the above (families, sizes, weights, line heights)"
  - "E) No typography tokens"

If A–D, follow up with `AskUserQuestion` (free text): "List your typography tokens."

**Question 4 — Other categories**

- question: "Any additional token categories?"
- choices:
  - "A) Border radius"
  - "B) Shadows / elevation"
  - "C) Z-index"
  - "D) Motion / duration / easing"
  - "E) Multiple of the above"
  - "F) None"

If A–E, follow up with `AskUserQuestion` (free text): "List the tokens for those categories."

**Question 5 — Modes / themes**

- question: "Do you use multiple modes or themes?"
- choices:
  - "A) Yes — light and dark"
  - "B) Yes — more than two themes (I'll describe them)"
  - "C) No, single mode only"

If A or B, follow up with `AskUserQuestion` (free text): "Describe the modes and list any token values that differ between them."

After all questions are answered, summarize the collected tokens back to the user using `AskUserQuestion`: "Here's what I collected — does this look correct?" with choices "Yes, write the file" / "No, let me correct something". Then write `design-system/tokens.json` using the standard shape above.

**If the user picks C (skip):**

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

Ask these questions one at a time using the `AskUserQuestion` tool. Wait for the user's answer before moving to the next. After all answers are collected, generate `ds-usage.md`.

**Question 1 — Breakpoints**

- question: "Do you have defined breakpoints or responsive behavior?"
- choices:
  - "A) Yes — standard (mobile / tablet / desktop)"
  - "B) Yes — custom breakpoints (I'll describe them)"
  - "C) No responsive rules defined yet"

If A or B, follow up with `AskUserQuestion` (free text): "List your breakpoints and any layout behavior that changes at each (e.g. column count, spacing, component variants)."

**Question 2 — Component composition rules**

- question: "Are there rules for combining components?"
- choices:
  - "A) Yes — there are explicit restrictions (e.g. never use X inside Y)"
  - "B) Yes — there are preferred patterns but no hard rules"
  - "C) No specific composition rules"

If A or B, follow up with `AskUserQuestion` (free text): "Describe the rules or patterns."

**Question 3 — Accessibility**

- question: "What accessibility requirements apply to this design system?"
- choices:
  - "A) WCAG AA compliance"
  - "B) WCAG AAA compliance"
  - "C) Internal a11y guidelines (I'll describe them)"
  - "D) No formal requirements defined"
  - "E) Multiple of the above"

If A, B, C, or E, follow up with `AskUserQuestion` (free text): "Are there specific patterns to call out — ARIA roles, focus order, colour contrast rules, or keyboard navigation?"

**Question 4 — Anti-patterns**

- question: "Are there known anti-patterns or misuses to document?"
- choices:
  - "A) Yes — I know specific ones to call out"
  - "B) Not sure — help me think through common ones for this type of system"
  - "C) No anti-patterns to document yet"

If A, follow up with `AskUserQuestion` (free text): "List the anti-patterns."
If B, suggest common ones based on the components discovered in Phase 2 and use `AskUserQuestion` with each as a selectable choice for the user to confirm which apply.

**Question 5 — Brand & copy guidelines**

- question: "Are there guidelines for copy inside components (labels, placeholders, error messages)?"
- choices:
  - "A) Yes — formal tone/voice guidelines exist"
  - "B) Yes — informal rules the team follows"
  - "C) No copy guidelines"

If A or B, follow up with `AskUserQuestion` (free text): "Describe the guidelines (e.g. sentence case for labels, avoid 'click here', max character counts)."

**Question 6 — Deprecated components**

- question: "Are there deprecated components still present in Figma that should not be used in code?"
- choices:
  - "A) Yes — I'll list them"
  - "B) Not sure — I'd like to flag some as review needed"
  - "C) No deprecated components"

If A, follow up with `AskUserQuestion` (free text): "List the deprecated component names."
If B, follow up with `AskUserQuestion` (free text): "List the components you're unsure about and I'll flag them."

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
