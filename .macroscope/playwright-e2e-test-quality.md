---
title: Playwright E2E Test Quality
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "e2e/**/*.spec.ts"
  - "e2e/**/*.test.ts"
  - "e2e/fixtures/**"
  - "e2e/helpers/**"
  - "e2e/playwright.config.ts"
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/build/**"
conclusion: neutral
---

Review changes to Playwright E2E tests and fixtures for the anti-patterns the team has explicitly banned. Flag arbitrary timeouts, `networkidle`/`waitForResponse` waits, CSS-class selectors, hardcoded dynamic store data, redundant ARIA/`data-testid`, count-comparison absence assertions, and headed-mode configuration.

## What to flag

### No waitForTimeout (🔴 Must fix)

- Any call to `page.waitForTimeout(...)` (or aliased `await page.waitForTimeout(...)`) in test or fixture code. Arbitrary waits are the single most common source of flake in this codebase. The fix is to wait for the actual user-visible effect with `await expect(locator).toBeVisible()`, `await expect.poll(() => ...)`, or `await expect(locator).toHaveValue(...)`.

**Cite as:** No waitForTimeout
**Source:** [`e2e/CLAUDE.md` § Anti-Patterns to Avoid](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#anti-patterns-to-avoid)
> "❌ `page.waitForTimeout(1000)` - Arbitrary waits (flaky)"

### No networkidle Or waitForResponse (🔴 Must fix)

- Any call to `page.waitForLoadState('networkidle')` or `page.waitForResponse(...)`. Network idle is unreliable on pages with analytics/polling/websockets, and waiting on a specific response couples the test to implementation details. The fix is to wait for the visible effect — the rendered data, the cleared input, the enabled button.

**Cite as:** No networkidle Or waitForResponse
**Source:** [`e2e/CLAUDE.md` § Wait for Visible Effects, Not Mechanisms](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#wait-for-visible-effects-not-mechanisms)
> "**NEVER use**: `page.waitForTimeout()`, `page.waitForLoadState('networkidle')`, or arbitrary waits."

### No CSS Class Selectors (🟡 Should fix)

- Any locator that targets CSS classes or styles, e.g. `page.locator('.cart-summary')`, `page.locator('div.product-card')`, or `:visible`-suffixed CSS selectors. The codified locator priority order is: role + accessible name → role + landmark → text content → `getByTestId` → CSS as last resort. CSS selectors break when markup is restyled and don't reflect how users perceive the page.

**Cite as:** No CSS Class Selectors
**Source:** [`e2e/CLAUDE.md` § DOM Elements over CSS](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#dom-elements-over-css)
> "Always choose selectors based on **DOM elements and semantic structure**, NOT CSS classes or styles. Tests should reflect how a user perceives and interacts with the page."

### No Hardcoded Dynamic Store Data (🔴 Must fix)

- Locators or assertions that match against literal collection titles, product names, or other values driven by `sortKey: UPDATED_AT`-style queries against the benchmark store (e.g. `getByRole('heading', {name: 'Winter Collection'})` on the homepage). The fix is to assert on structure ("an `h1` exists in the featured section") rather than the specific name. Specific names are only safe when navigating to a known entity by its stable `handle`.

**Cite as:** No Hardcoded Dynamic Store Data
**Source:** [`e2e/CLAUDE.md` § Dynamic Store Data](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#dynamic-store-data)
> "When testing homepage sections that query dynamic store data (e.g., "most recently updated collection"), assert **structure** rather than specific names"

### No Redundant ARIA (🟡 Should fix)

- New `role="..."` attributes that duplicate a native element's semantic (`role="list"` on `<ul>`, `role="button"` on `<button>`, `role="link"` on `<a>`), or `aria-label` added purely to enable a locator on an element that already has an accessible name. These changes don't help screen readers — they pollute the markup. If the only motivation is the test, restructure the markup or use `getByTestId` instead.

**Cite as:** No Redundant ARIA
**Source:** [`e2e/CLAUDE.md` § Accessibility Improvements During Test Writing](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#accessibility-improvements-during-test-writing)
> "**Not acceptable:**
> - Adding `aria-label` purely for test purposes when it provides no user benefit
> - Over-labeling elements that are already accessible
> - Adding explicit ARIA roles that duplicate native semantics (e.g., `role="list"` on `<ul>`, `role="button"` on `<button>`)
> - Using `data-testid` when role-based selectors would work with proper markup"

### Absence Broad, Presence Specific (🟡 Should fix)

- Removal-style assertions that check whether a count "decreased" or "is not equal" to a prior count (`expect(newCount).not.toBe(initialCount)`). These pass even when one item remains. The codified pattern is `.toHaveCount(0)` to prove absence anywhere on the page, paired with a scoped `.toBeVisible()` assertion that the empty-state UI rendered in the correct container.

**Cite as:** Absence Broad, Presence Specific
**Source:** [`e2e/CLAUDE.md` § Presence vs Absence Assertions](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#presence-vs-absence-assertions)
> "**Granularity Rule**: Assert **absence broadly** and **presence specifically**."

### Headless Tests Only (🔴 Must fix)

- Any change to `playwright.config.ts` (or any project config it imports) that sets `headless: false`, or any `package.json` script or CI workflow that passes `--headed`. Headed mode is for local debugging only — never commit it. The Playwright config relies on the default headless behavior.

**Cite as:** Headless Tests Only
**Source:** [`e2e/CLAUDE.md` § Always Use Headless Mode](https://github.com/Shopify/hydrogen/blob/main/e2e/CLAUDE.md#always-use-headless-mode)
> "Tests should ALWAYS be run in headless mode, both in development and in CI."

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (`.eslintrc`, `tsconfig.json`, `.prettierrc`).
- Ignore generated code, vendored dependencies, and lockfiles.
- Do not flag tests that use `page.waitForURL()` — that's an explicit navigation wait, not an arbitrary timeout, and is allowed.
- Do not flag `expect.poll()` with a timeout option — that's the recommended replacement for arbitrary waits.
- When the same `waitForTimeout` (or other anti-pattern) call appears across multiple sister test files with near-identical context (e.g., `old-cookies/` ↔ `new-cookies/` parallel specs, or `*-accept.spec.ts` ↔ `*-decline.spec.ts` pairs), comment **once** on the most representative instance and list the other occurrences inline in the comment body (e.g., "Same pattern also appears in `path/to/sibling-a.spec.ts:103` and `path/to/sibling-b.spec.ts:122`"). Do not post the same finding on each sibling file — it reads as a loud agent rather than a thorough one.

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
