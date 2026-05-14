---
title: Changeset Discipline
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "packages/*/src/**"
  - "packages/*/package.json"
  - "templates/skeleton/**"
  - "packages/cli/assets/**"
  - ".changeset/**"
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/build/**"
  - "pnpm-lock.yaml"
conclusion: neutral
---

Review PRs that touch published package source for changeset completeness. Verify that a changeset file exists when required, that its package list satisfies Hydrogen's bundling-chain rules (skeleton ↔ cli-hydrogen ↔ create-hydrogen, hydrogen-react → hydrogen), and that the body reads like merchant-facing release notes rather than an internal commit message.

## What to flag

### Changeset Required For Package Changes (🔴 Must fix)

- Any PR that modifies files under `packages/*/src/**` or any `packages/*/package.json` and does not include at least one new `.md` file under `.changeset/`. Without it, the release pipeline either skips the version bump entirely or ships unannounced changes.

**Cite as:** Changeset Required For Package Changes
**Source:** [`CLAUDE.md` § Changeset Rules](https://github.com/Shopify/hydrogen/blob/main/CLAUDE.md#changeset-rules)
> "If changes affect `packages/*/src/**` or `packages/*/package.json`, a changeset is required."

### Skeleton Changes Bump All Three Packages (🔴 Must fix)

- Any PR that modifies `templates/skeleton/**` whose changeset bumps `skeleton` but omits `@shopify/cli-hydrogen` or `@shopify/create-hydrogen`. The `pnpm run changeset add` UI hides these two packages because they have no code changes, so contributors must add them manually. Without all three bumps, `npm create @shopify/hydrogen@latest` continues to scaffold the stale skeleton.

**Cite as:** Skeleton Changes Bump All Three Packages
**Source:** [`CLAUDE.md` § Rule 1: Skeleton Template Changes](https://github.com/Shopify/hydrogen/blob/main/CLAUDE.md#rule-1-skeleton-template-changes)
> "Any change to `templates/skeleton` must include a changeset specifying a **patch** bump for **both** `@shopify/cli-hydrogen` and `@shopify/create-hydrogen` (in addition to `skeleton` itself)."

### hydrogen-react Bumps Hydrogen (🔴 Must fix)

- Any changeset that lists `@shopify/hydrogen-react` but does not also include a corresponding bump for `@shopify/hydrogen`. Hydrogen re-exports the entire surface of `hydrogen-react`, so a `hydrogen-react`-only bump means consumers never receive the update.

**Cite as:** hydrogen-react Bumps Hydrogen
**Source:** [`CLAUDE.md` § Rule 2: hydrogen-react Changes](https://github.com/Shopify/hydrogen/blob/main/CLAUDE.md#rule-2-hydrogen-react-changes)
> "The entire contents of `hydrogen-react` are re-exported in Hydrogen. Any changeset for `hydrogen-react` must also specify a corresponding bump for `hydrogen`."

### CLI Init Changes Bump create-hydrogen (🔴 Must fix)

- Any PR that modifies the `hydrogen init` code path in `packages/cli-hydrogen`, anything under `packages/cli/assets/`, or virtual-route templates, whose changeset bumps `@shopify/cli-hydrogen` but omits `@shopify/create-hydrogen`. Both packages must be bumped because `create-hydrogen` bundles the `init` entry point and the full `dist/assets` directory at build time. Non-init CLI changes (bug fixes to `dev`/`build`/`check`) only need `@shopify/cli-hydrogen`.

**Cite as:** CLI Init Changes Bump create-hydrogen
**Source:** [`CLAUDE.md` § I'm Updating the CLI (cli-hydrogen)](https://github.com/Shopify/hydrogen/blob/main/CLAUDE.md#im-updating-the-cli-cli-hydrogen)
> "**Scenario B: Change affects the `init` command, scaffolding process, or CLI assets**
> - Bump both `@shopify/cli-hydrogen` and `@shopify/create-hydrogen`
> - Examples: changes to how `hydrogen init` (which powers `npm create @shopify/hydrogen`) works; changes to virtual-route templates or other files in `packages/cli/assets/`"

### Write Changesets For Merchants (🟡 Should fix)

- Flag changeset bodies that read like internal commit messages ("refactor cart hook", "fix typo per review", "address review comments") instead of merchant-facing release notes. The body feeds both the published CHANGELOG and the `h2 upgrade` command's generated upgrade instructions — merchants read it, not maintainers. The fix is to rewrite as a user-visible description of what changed and how to adopt it.

**Cite as:** Write Changesets For Merchants
**Source:** [`CLAUDE.md` § Changeset Rules](https://github.com/Shopify/hydrogen/blob/main/CLAUDE.md#changeset-rules)
> "These changesets go into Hydrogen's changelog, and are also used to generate upgrade instructions. **Write the changeset as if the audience is a merchant building with Hydrogen.**"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (`.eslintrc`, `tsconfig.json`, `.prettierrc`).
- Ignore generated code, vendored dependencies, lockfiles (`pnpm-lock.yaml`), and `dist/` output.
- Do not flag missing changesets on PRs that only touch tests (`**/*.test.ts`, `**/*.spec.ts`), docs (`**/*.md`), or repo tooling outside `packages/`.

## Output format

Group findings by severity (🔴 Must fix, 🟡 Should fix, 🟢 Nit). For each finding, post an inline review comment on the offending line — for changeset issues, that's typically the `.changeset/*.md` file itself or the `package.json` whose bump is missing.

Every inline comment must end with a collapsible citation block pointing back to the original rule in this repository's own docs. Identify which `###` rule above the finding violates, then build the footer from that rule's `Cite as` / `Source` / quote — verbatim, no paraphrasing. Insert a blank line between the comment body and the `<details>` block.

```
<details><summary><em>Violates</em>: {Cite as value}</summary>

**Source:** [{path} § {section}]({anchored_url})

> {verbatim quote}

</details>
```

After inline comments, post a top-level PR comment with a one-line summary per finding. If no issues are found in the changed code, post a single top-level comment: **"All clear."**

If no issues are found, report **"All clear."** Do not invent issues to fill space.
