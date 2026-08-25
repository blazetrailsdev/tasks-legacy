---
title: "trailties' generator classify/dasherize shadow the ActiveSupport inflector with different semantics"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/trailties/src/generators/base.ts` exports a `classify` that is NOT
ActiveSupport's:

```ts
export function classify(name: string): string {
  return _camelize(name.replace(/-/g, "_"));
}
```

ActiveSupport's `classify` singularizes the last word before camelizing —
`camelize(singularize(table_name.sub(/.*\./, "")))`
(`vendor/rails/activesupport/lib/active_support/inflector/methods.rb`, `def classify`).
trails' version camelizes only, so `classify("posts")` yields `Posts` where
Rails yields `Post`. Callers work around it: `model-generator.ts` writes
`classify(tableize(className))` and `scaffold-controller-generator.ts` writes
`tableize(classify(singularLeaf))`, round-tripping through `tableize` to get the
inflection the name promises.

Surfaced while removing the sibling `tableize`/`underscore` re-exports from the
same file (PR #6103, `triage-newly-visible-object-literal-accessors`): those two
were plain aliases and were deleted in favour of importing from
`@blazetrails/activesupport`. `classify` could not follow, because it does not
mean what its name means, and swapping it in place would change every generated
file name.

`dasherize` in the same file has the same shape problem in miniature —
`_dasherize(_underscore(name))` is a two-step composition Rails does not have
under that name (`ActiveSupport::Inflector#dasherize` is `underscore.tr("_", "-")`
applied to an already-underscored string).

Rails' generators reach these through `String#classify` / `#dasherize` on the
model name (`railties/lib/rails/generators/named_base.rb`), not through
generator-local redefinitions.

## Converged shape

Delete `classify` (and `dasherize`) from `generators/base.ts` and import
`classify` / `dasherize` from `@blazetrails/activesupport`, fixing each call site
to pass what Rails passes — the singular model name — instead of pre-mangling the
argument through `tableize` to compensate. Generator golden-file expectations
move with it where the emitted names were wrong.

## Acceptance criteria

- `generators/base.ts` declares no `classify` or `dasherize` of its own; call
  sites use the ActiveSupport inflector.
- No call site round-trips through `tableize` purely to undo `classify`'s missing
  singularization.
- Generator tests under `packages/trailties/src/generators/` stay green, with any
  changed expectation shown to be the Rails-correct name.
- `pnpm parity:api:extra --package trailties` drops the corresponding novel names; no
  allowlist rows added.
