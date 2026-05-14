---
title: Cookbook Recipe Integrity
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "cookbook/recipes/**"
  - "cookbook/llms/**"
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
conclusion: neutral
---

Review changes to cookbook recipes for schema-validation and doc-sync requirements. Recipes are the public cookbook surface on `shopify.dev` — the generated `README.md` is what merchants read, and `cookbook/llms/{name}.prompt.md` is what AI coding assistants consume. Flag unquoted step values, missing step descriptions, orphaned patch/ingredient files, missing LLM prompt files, and — most importantly — ingredient `.tsx` edits that don't regenerate the matching `README.md` and `llms/*.prompt.md`.

## What to flag

### Quote Step Values As Strings (🔴 Must fix)

- In `recipe.yaml`, any `step:` value that's a bare integer (`step: 1`) instead of a quoted string (`step: "1"`). YAML coerces unquoted integers to numbers, but the recipe Zod schema expects strings, so this fails validation with `expected string, received number`.

**Cite as:** Quote Step Values As Strings
**Source:** [`cookbook/CLAUDE.md` § IMPORTANT: Critical Rules](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#important-critical-rules)
> "Step values are quoted strings: `step: "1"` not `step: 1` (YAML coerces unquoted integers to numbers, but the Zod schema expects strings)"

### Every Step Has Description (🟡 Should fix)

- In `recipe.yaml`, any entry under `steps:` that omits the `description:` field or sets it to `null`/empty string. Empty descriptions break the generated `README.md` and the recipe validator surfaces them as `Step N has null description`.

**Cite as:** Every Step Has Description
**Source:** [`cookbook/CLAUDE.md` § IMPORTANT: Critical Rules](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#important-critical-rules)
> "Every step has a description (non-null, non-empty)"

### Step Names Unique Per Step (🟡 Should fix)

- Within a single `step:` number in `recipe.yaml`, any duplicate `name:` value. The validator's downstream addressing assumes uniqueness within a step.

**Cite as:** Step Names Unique Per Step
**Source:** [`cookbook/CLAUDE.md` § IMPORTANT: Critical Rules](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#important-critical-rules)
> "Step names are unique within the same step number"

### No Orphaned Patch Files (🟡 Should fix)

- Any file added under `cookbook/recipes/{name}/patches/` or `cookbook/recipes/{name}/ingredients/` that isn't referenced from `recipe.yaml`. Also flag PRs that remove a recipe step but leave the corresponding patch/ingredient file behind.

**Cite as:** No Orphaned Patch Files
**Source:** [`cookbook/CLAUDE.md` § Common Errors](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#common-errors)
> "❌ `patches/old-file.patch  validatePatchFiles: Orphaned patch file not referenced in recipe`"

### LLM Prompt File Must Exist (🟡 Should fix)

- Any PR that adds a new recipe directory at `cookbook/recipes/{name}/` without a matching `cookbook/llms/{name}.prompt.md`. AI coding assistants consume the LLM prompt file when merchants ask their tooling for help with that scenario — without it, the recipe is invisible to that surface.

**Cite as:** LLM Prompt File Must Exist
**Source:** [`cookbook/CLAUDE.md` § IMPORTANT: Critical Rules](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#important-critical-rules)
> "LLM prompt file exists at `llms/{recipe-name}.prompt.md`"

### Regenerate Docs After Ingredient Edits (🔴 Must fix)

- Any PR that modifies `.tsx`, `.ts`, `.jsx`, or `.js` files under `cookbook/recipes/{name}/ingredients/` but does not also include corresponding changes to both `cookbook/recipes/{name}/README.md` *and* `cookbook/llms/{name}.prompt.md`. Both generated files inline the ingredient source code — if they're not regenerated via `pnpm run cookbook -- render --recipe {name}`, the docs continue to teach the old pattern even after the fix lands.

**Cite as:** Regenerate Docs After Ingredient Edits
**Source:** [`.claude/skills/hydrogen-dev-workflow/SKILL.md` § Cookbook Architecture: Ingredients and Generated Docs](https://github.com/Shopify/hydrogen/blob/main/.claude/skills/hydrogen-dev-workflow/SKILL.md#cookbook-architecture-ingredients-and-generated-docs)
> "Both the `README.md` and `llms/{name}.prompt.md` must be committed as part of the same fix. Failing to regenerate means the docs teach the wrong pattern even after the ingredient files are correct."

### Recipe Names Kebab-Case (🟢 Nit)

- New recipe directory names or `name:` values under `steps:` that reference file paths but aren't kebab-case (e.g. `InfiniteScroll`, `infinite_scroll`, `infiniteScroll`). Match the existing recipes under `cookbook/recipes/` — they're all kebab-case.

**Cite as:** Recipe Names Kebab-Case
**Source:** [`cookbook/CLAUDE.md` § Code Style](https://github.com/Shopify/hydrogen/blob/main/cookbook/CLAUDE.md#code-style)
> "- Step names: Unique within same step number, kebab-case for file paths
> - …
> - Naming: Follow existing recipe patterns (kebab-case)"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (`.eslintrc`, `tsconfig.json`, `.prettierrc`).
- Ignore generated code, vendored dependencies, and lockfiles.
- Do not flag PRs that only edit `cookbook/recipes/{name}/recipe.yaml` (without ingredient changes) for the doc-regeneration rule — that rule is specifically about ingredient `.tsx`/`.ts` source edits.
- Do not flag the cookbook tooling itself (`cookbook/src/**`) under these rules — they apply to recipe content, not the cookbook CLI.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line.

Every inline comment must end with a collapsible citation block pointing back to the original rule in this repository's own docs. Identify which `###` rule above the finding violates, then build the footer from that rule's `Cite as` / `Source` / quote — verbatim, no paraphrasing. Insert a blank line between the comment body and the `<details>` block.

```
<details><summary><em>Violates</em>: {Cite as value}</summary>

**Source:** [{path} § {section}]({anchored_url})

> {verbatim quote}

</details>
```

After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
