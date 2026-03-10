---
name: ds
description: Searches through the local design system files to find component information, tokens, or usage rules. Use when user asks about a design system component, token, or pattern, or runs /ds.
---

Search the local design system files and return relevant component, token, or usage information.

## Phase 1 — Check DS exists

Read `design-system.md` from the project root.

If not found, stop and tell the user:
> "No initialized design system found. Run `/initialize-ds` first."

Parse the index to build a map of:
- All part folders and their component files
- All `tokens.css` file paths
- The `ds-usage.md` path

## Phase 2 — Search

The user's query is the text after `/ds`. For example: `/ds Button` → query is `Button`.

Search strategy (in order):

1. **Component name match**: Check all component `.md` filenames and headings for the query (case-insensitive).
2. **Token name match**: Search all `tokens.css` files for variable names containing the query.
3. **Full-text match**: Search component `.md` file content for the query string.
4. **Usage rules match**: Search `ds-usage.md` for the query string.

For each match, record:
- File path (relative to project root)
- Matched section or line
- A short excerpt (2–5 lines of context)

## Phase 3 — Return results

If a single clear match is found, return the full content of the matched file or section.

If multiple matches are found, list them:

```
Found [N] results for "[query]":

1. [ComponentName] — design-system/[part]/[ComponentName].md
   > [excerpt]

2. --[token-name] — design-system/[part]/tokens.css:L[N]
   > [excerpt]

3. Usage rule — ds-usage.md
   > [excerpt]
```

Then ask: "Which result would you like to see in full?"

If no matches are found, respond:
> "No results found for '[query]' in the design system. Check the index at `design-system.md` for available parts and components."

Always include the file path in results so the user can navigate directly to the source.
