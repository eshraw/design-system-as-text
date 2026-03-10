---
name: update-ds
description: Checks Figma for a new design system version and updates the local markdown/CSS files with changes. Use when user says "update design system", "sync DS", "check for DS changes", or runs /update-ds.
metadata:
  mcp-server: figma
---

Sync local design system files with the latest Figma version. Follow these phases exactly.

## Phase 1 — Load local state

Read `design-system.md` from the project root and extract:
- `figma_sources`: list of Figma URLs, file keys, and last-synced versions
- `last_synced_at`: timestamp of last sync

If `design-system.md` is not found or is missing frontmatter, stop and tell the user:
> "No initialized design system found. Run `/initialize-ds` first."

## Phase 2 — Check for updates

For each fileKey in `figma_sources`:

1. Call `get_metadata(fileKey)` to retrieve the current Figma file version.
2. Compare with the stored `last_synced_version`.

If all versions match: report "Design system is already up to date." and stop.

If any version differs: continue to Phase 3.

## Phase 3 — Identify changes

For each updated fileKey:

1. Re-fetch `get_variable_defs(fileKey)` and compare token values against existing `tokens.css` files.
2. Re-fetch `get_design_context` for top-level component frames and compare against existing component `.md` files.
3. Re-fetch `get_metadata(fileKey)` to detect added or removed pages/frames/components.

Categorize all differences:
- **Added**: components or tokens present in Figma but not locally
- **Modified**: components or tokens that exist locally but have changed values or structure
- **Removed**: components or tokens present locally but no longer in Figma

Present the diff summary to the user before making any changes:

```
Changes detected in [file name] (v[old] → v[new]):

Added:
  + [ComponentName]
  + --color-new-token

Modified:
  ~ [ComponentName] (variants changed)
  ~ --color-primary (#old → #new)

Removed:
  - [DeprecatedComponent]
```

Ask the user to confirm before applying changes.

## Phase 4 — Update files

Apply only the changes identified in Phase 3.

### Token files (`tokens.css`)
Regenerate fully from Figma variable definitions. These files contain no manual content.

```css
/* Auto-generated from Figma. Do not edit manually. */
/* Source: [FIGMA_URL] | Version: [VERSION] | Synced: [ISO_DATE] */

:root {
  /* [Collection Name] */
  --[token-name]: [value];
}
```

### Component `.md` files
Preserve content between `<!-- custom -->` and `<!-- /custom -->` markers exactly as-is.

For the rest of the file, regenerate from the updated Figma context.

Example of preserved custom section:
```markdown
## Overview

[Regenerated from Figma]

<!-- custom -->
Our team uses this component exclusively for primary CTAs. Never use the ghost variant in forms.
<!-- /custom -->

## Variants
[Regenerated from Figma]
```

For **removed** components: do not delete the file automatically. Instead, add a notice at the top:

```markdown
> **Deprecated**: This component no longer exists in Figma as of [DATE]. It may still exist in code but should not be used in new work. See [ds-usage.md](../../ds-usage.md) for the deprecated components list.
```

For **added** components: create new `.md` files using the same template as `/initialize-ds` Phase 4.

### `design-system.md` index
Update the `last_synced_version` and `last_synced_at` fields in frontmatter. Add or remove rows in the parts/token tables to reflect structural changes.

## Phase 5 — Report

Show a final summary:

```
Design system updated to version [VERSION].

Files updated: [N]
Files added: [N]
Files deprecated: [N]
Tokens changed: [N]
```

Ask: "Would you also like to re-run the usage interview to update `ds-usage.md`?"

If yes, run the Phase 6 interview from `/initialize-ds`.
