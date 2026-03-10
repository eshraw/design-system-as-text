# design-system-as-text

A Claude plugin that represents your Figma design system as structured markdown and CSS files — making it easy for Claude to understand and apply your design system when building UI.

## What it does

Instead of Claude guessing at your design tokens and component rules, this plugin pulls your design system directly from Figma and stores it as local files your AI tools can read. Once initialized, you can query components, check tokens, and keep everything in sync as Figma evolves.

## Skills

### `/initialize-ds`
Imports a design system from Figma into structured local files. Walks through your Figma file, proposes a folder structure, generates component docs and token CSS files, then runs a short interview to capture usage rules your team cares about (breakpoints, composition rules, accessibility, anti-patterns, deprecated components).

Requires the Figma MCP server.

### `/update-ds`
Checks Figma for a newer version and syncs only what changed. Shows a diff before touching anything — added, modified, and removed components and tokens. Preserves any custom notes you've added to component files inside `<!-- custom -->` blocks.

Requires the Figma MCP server.

### `/ds [query]`
Searches the local design system files and returns matching components, tokens, or usage rules. Works without Figma — reads only from the files already on disk.

## Output structure

```
design-system.md          ← master index, Figma source + version stored here
ds-usage.md               ← usage rules (edit freely)
design-system/
  [part]/
    [part].md             ← part overview
    tokens.css            ← design tokens (do not edit manually)
    [Component].md        ← component docs
    [sub-part]/
      ...
```

## What you can edit

| File | Editable? |
|------|-----------|
| `ds-usage.md` | Yes — edit freely |
| `[Component].md` | Partially — add notes inside `<!-- custom --> ... <!-- /custom -->` blocks; they're preserved on sync |
| `tokens.css` | No — fully regenerated on every `/update-ds` |
| `design-system.md` | No — managed by the plugin |

## Installation

In Claude Code:

```
/plugin marketplace add eshraw/design-system-as-text
/plugin install design-system-as-text@design-system-as-text
```

## Requirements

- Claude with plugin support
- Figma MCP server (for `/initialize-ds` and `/update-ds`)
