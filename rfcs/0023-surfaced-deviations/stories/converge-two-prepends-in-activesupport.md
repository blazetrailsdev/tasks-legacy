---
title: "Converge the two activesupport prepend() functions onto Ruby's Module#prepend"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`@blazetrails/activesupport` now has two exported functions named `prepend`, both
claiming to mirror Ruby's `Module#prepend`:

- `packages/activesupport/src/prepend.ts` — wraps each existing method on the
  target so the module's version receives the original as an explicit `super_`
  first argument. It validates that every module entry is a function and that
  the target already defines a same-named method, and it is the one re-exported
  from the package index.
- `packages/activesupport/src/include.ts` (added by PR #6461) — the actual
  ancestry splice that pairs with `include()`: copies the module's property
  descriptors onto `klass.prototype` so the module's method wins over the class
  body. This is what a module's `prepend_features` hook installs, and it is what
  `Deprecation::DeprecatedConstantProxy#prepend_features` needs
  (`vendor/rails/activesupport/lib/active_support/deprecation/proxy_wrappers.rb:161-164`,
  whose body is `base.prepend(target)`). It is deliberately NOT re-exported from
  the index to avoid colliding with the other one.

Ruby has exactly one `Module#prepend`. `prepend.ts`'s explicit-`super_` shape is
a trails invention for the case where the module wants to call the original; in
Ruby that is just `super` inside a prepended module, resolved by the ancestry
the splice creates.

## Converged shape

One `prepend(klass, mod)`, the ancestry splice, exported from the package index.
Callers that need the original method reach it the way Ruby does — through the
prototype chain the splice leaves intact — rather than through an injected
`super_` parameter. Migrate `prepend.ts`'s call sites onto it and delete the
duplicate, so `include`/`prepend`/`extend` are one coherent trio in `include.ts`.

## Acceptance criteria

- One exported `prepend` in `@blazetrails/activesupport`, the ancestry splice.
- `packages/activesupport/src/prepend.ts` and its `PrependMethod` /
  `PrependModule` types are gone, with every caller migrated.
- `DeprecatedConstantProxy#prependFeatures` keeps working through the surviving
  `prepend`; `proxy-wrappers.test.ts`'s "prepending proxy module" stays green.
- `prepend.test.ts` cases are ported onto the surviving shape (test names verbatim).
