---
title: Skeleton Template Style
model: claude-opus-4-6
reasoning: high
effort: high
input: full_diff
include:
  - "templates/skeleton/app/**/*.ts"
  - "templates/skeleton/app/**/*.tsx"
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/build/**"
  - "templates/skeleton/app/graphql/**/*.generated.ts"
  - "templates/skeleton/.shopify/**"
conclusion: neutral
---

Review changes to the skeleton template's route and component code for the conventions codified in `templates/TEMPLATE_GUIDELINES.md`. Flag missing `ErrorBoundary` exports, missing `errorElement` props on `<Await>`, `try/catch` inside `loader`/`action`/Component, non-Hydrogen npm dependencies, out-of-order route exports, and the codified GraphQL and loader-return conventions.

## What to flag

### ErrorBoundary In Every Route (🔴 Must fix)

- Any route file under `templates/skeleton/app/routes/**` that does not export an `ErrorBoundary` component. Hydrogen relies on `ErrorBoundary` to surface Storefront query failures, 500s, and other unexpected errors to the user — without it, errors are swallowed and the route renders blank. The `error` parameter should be typed as `unknown` since anything in JS can be thrown.

**Cite as:** ErrorBoundary In Every Route
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Error Handling](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#error-handling)
> "**Have an `ErrorBoundary` in every route template.** `ErrorBoundary` is used when an Error is thrown in a "loader", and is generally meant for unexpected errors, like 500, 503, etc. Any Storefront query or mutation error will be handled by the `ErrorBoundary`. Type the error as "unknown" since _anything_ in JS can be thrown 🙂"

### errorElement On Every Await (🔴 Must fix)

- Any `<Await ...>` JSX element that does not include an `errorElement` prop. Deferred promises that reject without an `errorElement` are silently swallowed and the user sees a blank UI. The fix is to add `errorElement={<SomeFallback />}` or render an inline error UI.

**Cite as:** errorElement On Every Await
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Error Handling](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#error-handling)
> "**Use the "errorElement" prop on every `<Await>` component.** When using "defer", some promises may be rejected at a later time. The only way to handle this is to use the "errorElement" on the associated <Await> component, otherwise the error is swallowed."

### No try/catch In loader/action/Component (🟡 Should fix)

- `try/catch` blocks inside `loader`, `action`, or the default-export route Component. These three Route Module APIs are already handled by `ErrorBoundary`/`CatchBoundary` — wrapping them defeats the framework's error routing. `try/catch` is fine and expected in `meta`, `links`, `handle`, and other route exports where a thrown error crashes the server.

**Cite as:** No try/catch In loader/action/Component
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Error Handling](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#error-handling)
> "**Use try/catch in "loader", "action", and the Component.** For templates it's easier to let the error be thrown and get handled by the `ErrorBoundary` than to handle it manually."

### No Non-Hydrogen NPM Deps In Templates (🟡 Should fix)

- New `import` statements in `templates/skeleton/**` that pull from packages outside the Hydrogen, React Router, React, or Remix-utility ecosystem (e.g. `tiny-invariant`, `clsx`, `zod`, `lodash`, `date-fns`). Templates should be self-contained learning material; helper imports abstract away patterns the template is meant to teach (especially error handling) and add installation friction for scaffolded projects.

**Cite as:** No Non-Hydrogen NPM Deps In Templates
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Template Dependencies](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#template-dependencies)
> "Use `npm` packages or dependencies in templates that aren't directly associated with Hydrogen or Remix. A package like `tiny-invariant` could be 1) confusing to developers unfamiliar with it, 2) abstract away things that we want to teach, such as correct error handling, and 3) require us to figure out how to correctly give direction on installation and updating the `package.json` upon template creation."

### Route API Top-Down Order (🟡 Should fix)

- Route files that declare exports out of the codified top-down order. The order is: HTTP-header tweaks (`shouldRevalidate`, `headers`, `meta`, `links`) → data manipulation (`loader`, `action`) → UI (default Component) → error handling (`ErrorBoundary`) → GraphQL query strings at the bottom. Also flag arrow-function expressions (`const loader = async () => ...`) where a function declaration (`export async function loader() ...`) is possible.

**Cite as:** Route API Top-Down Order
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Remix Route APIs](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#remix-route-apis)
> "Order these APIs following a top-down order of concerns:
>   1. Http header tweaks (`shouldRevalidate`, `headers`, `meta`, `links`)
>   1. Data manipulation (`loader`, `action`)
>   1. UI (`Component`)
>   1. Error handling (`ErrorBoundary`)
>   1. Storefront API GraphQL query strings"

### Prefer type Over interface (🟢 Nit)

- New `interface Foo { ... }` declarations in skeleton TypeScript files where `type Foo = { ... }` would work. The convention exists to keep skeleton code consistent so developers reading the scaffolded output have one mental model. Use `interface` only when declaration merging is genuinely required.

**Cite as:** Prefer type Over interface
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § TypeScript](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#typescript)
> "Use `type` over `interface` when possible"

### GraphQL Constants SCREAMING_SNAKE_CASE (🟡 Should fix)

- Top-level `const`s holding GraphQL query/mutation strings that aren't `SCREAMING_SNAKE_CASE` (`QUERY_PRODUCT`, `MUTATION_ADD_TO_CART`), GraphQL operation names embedded inside the query string that aren't globally unique and based on filename + content (e.g. `query product_shop { ... }` inside `product.tsx`), or query constants placed anywhere other than the bottom of the route file.

**Cite as:** GraphQL Constants SCREAMING_SNAKE_CASE
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Graphql Query Definitions](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#graphql-query-definitions)
> "- Declare query and mutation constant names in SCREAMING_SNAKE_CASE
> - Ensure that the query name itself is a (globally-unique) name based on the filename and the query contents
> - Place the query at the bottom of the route template."

### Loader Returns Use data() For Errors (🟡 Should fix)

- `loader`/`action` return statements that don't follow the convention: raw object for normal data, `redirect()` from `react-router` for redirects, `data(...)` for error responses with custom status/headers (like 404s), and `new Response(...)` for non-JSON document responses (`.xml`, `.txt`). Also flag response headers that aren't capitalized-kebab-case (`cache-control` instead of `Cache-Control`).

**Cite as:** Loader Returns Use data() For Errors
**Source:** [`templates/TEMPLATE_GUIDELINES.md` § Remix Loader Return Values](https://github.com/Shopify/hydrogen/blob/main/templates/TEMPLATE_GUIDELINES.md#remix-loader-return-values)
> "- Return raw json object by default
> - Use `await` if you want the data to be streamed in later
> - Use `redirect()` from `react-router` to redirect
> - Use `data()` for errors (like 404s)
> - Use `new Response()` for unique document responses like `.xml` and `.txt`
> - Use capitalized and kebab-cased headers in responses, like `Cache-Control`"

## What to ignore

- Do not duplicate the built-in Correctness check — it already covers runtime bugs and logic errors.
- Do not flag issues already caught by the repository's static linters (`.eslintrc`, `tsconfig.json`, `.prettierrc`).
- Ignore generated code (`*.generated.ts`, `*.generated.d.ts`), vendored dependencies, and lockfiles.
- Do not flag the `Prefer type Over interface` rule on declaration-merging cases (e.g. extending `Window` or third-party module augmentation).
- Component files under `templates/skeleton/app/components/**` are not full routes — only apply the route-specific rules (ErrorBoundary, route export order, loader returns) to files in `templates/skeleton/app/routes/**`.

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
