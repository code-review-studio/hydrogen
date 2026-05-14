---
title: React Router Import Hygiene
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "templates/skeleton/**/*.ts"
  - "templates/skeleton/**/*.tsx"
  - "templates/skeleton/**/*.js"
  - "templates/skeleton/**/*.jsx"
  - "cookbook/recipes/*/ingredients/**/*.ts"
  - "cookbook/recipes/*/ingredients/**/*.tsx"
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/build/**"
  - "templates/skeleton/.shopify/**"
conclusion: neutral
---

Review changes to the skeleton template and cookbook ingredient code for banned import sources. Hydrogen migrated to React Router v7; any new code that pulls from the old `@remix-run/*` packages or from `react-router-dom` must be rewritten. This is one of the two `.cursor/rules/*.mdc` files the team wired into their AI tooling — they care about it enough to enforce it on every Cursor session.

## What to flag

### No @remix-run Imports (🔴 Must fix)

- Any `import ... from '@remix-run/<anything>'` statement. Replace with the equivalent React Router v7 package per the codified mapping: `@remix-run/react` → `react-router`, `@remix-run/dev` → `@react-router/dev`, `@remix-run/cloudflare` → `@react-router/cloudflare`, `@remix-run/node` → `@react-router/node`, `@remix-run/server-runtime` → `react-router`, `@remix-run/testing` → `react-router`. The skeleton is React Router v7, not Remix.

**Cite as:** No @remix-run Imports
**Source:** [`templates/skeleton/.cursor/rules/hydrogen-react-router.mdc`](https://github.com/Shopify/hydrogen/blob/main/templates/skeleton/.cursor/rules/hydrogen-react-router.mdc)
> "When you see imports from Remix packages, replace them with their equivalent React Router v7 packages."

### No react-router-dom Imports (🔴 Must fix)

- Any `import ... from 'react-router-dom'` statement. Hydrogen uses `react-router` directly — the `-dom` suffix package is a historical artifact and must never appear in this codebase. Replace with `react-router`.

**Cite as:** No react-router-dom Imports
**Source:** [`templates/skeleton/.cursor/rules/hydrogen-react-router.mdc`](https://github.com/Shopify/hydrogen/blob/main/templates/skeleton/.cursor/rules/hydrogen-react-router.mdc)
> "NEVER USE 'react-router-dom' imports!"

### Use react-router For Redirects (🟡 Should fix)

- `redirect()` imports that resolve from anywhere other than `react-router` (e.g. `import {redirect} from '@remix-run/server-runtime'` or `import {redirect} from 'react-router-dom'`). This is the most common Remix-era pattern that slips through migration. The fix is `import {redirect} from 'react-router'`.

**Cite as:** Use react-router For Redirects
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Remix Loader Return Values](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#remix-loader-return-values)
> "Use `redirect()` from `react-router` to redirect"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (`.eslintrc`, `tsconfig.json`, `.prettierrc`).
- Ignore generated code, vendored dependencies, and lockfiles.
- Do not flag imports from `@react-router/*` packages — those are the correct destinations.
- Do not flag imports from `react-router` (without the `-dom` suffix) — that's the canonical package for this codebase.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line — typically the `import` statement itself.

Every inline comment must end with a collapsible citation block pointing back to the original rule in this repository's own docs. Identify which `###` rule above the finding violates, then build the footer from that rule's `Cite as` / `Source` / quote — verbatim, no paraphrasing. Insert a blank line between the comment body and the `<details>` block.

```
<details><summary><em>Violates</em>: {Cite as value}</summary>

**Source:** [{path} § {section}]({anchored_url})

> {verbatim quote}

</details>
```

After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
