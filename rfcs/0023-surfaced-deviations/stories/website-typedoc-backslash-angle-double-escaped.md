---
title: "escape-typedoc double-escapes typedoc's \\< so generics render as literal &lt;"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "website"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`escapeForVue` in `packages/website/scripts/escape-typedoc.mjs` escapes every
`<` outside fenced code blocks to `&lt;`. typedoc already emits backslash-escaped
angle brackets in signature lines (`**extractOptionsBang**\<`T`\>(...)`), so the
post-processing turns them into `\&lt;`, which markdown-it renders as the literal
text `&lt;` on the page:

    <p><strong>extractOptionsBang</strong>&amp;lt;<code>T</code>&gt;(...)</p>

i.e. every generic parameter list in the API docs reads `Foo&lt;T>` instead of
`Foo<T>`. Reproduce with `pnpm --filter @blazetrails/website run docs:build` and
open any generic function page (e.g.
`docs/api/@blazetrails/activesupport/functions/extractOptionsBang.md`).

Surfaced while fixing the brace/markdown-it-attrs build break in PR #6462; out
of scope there (that PR fixed a hard build failure, this is a rendering defect).

## Acceptance criteria

- A `<` typedoc already escaped as `\<` is not double-escaped; generic signatures
  render as `Foo<T>` on the built site.
- Existing `escapeForVue` unit tests keep passing; add cases for `\<` input.
- `pnpm --filter @blazetrails/website run docs:build` stays green.
