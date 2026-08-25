---
title: "Generate the bare pattern's reader through the pattern path, retiring defineAliasAccessor"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`define_attribute_method_pattern`'s bare pattern (empty prefix and suffix) is
what Rails uses to generate the plain attribute reader, via
`define_method_attribute` (`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:333-336`,
`vendor/rails/activerecord/lib/active_record/attribute_methods/read.rb:20-34`).

trails generates the plain reader elsewhere — `attribute()` defines it as a real
accessor property — so `defineAttributeMethodPattern`
(`packages/activemodel/src/attribute-methods.ts`) returns early for the bare
pattern rather than generating a colliding method. PR #6543 then had to carve an
exception into that exception: an _alias_ name has no `attribute()`-defined
accessor, so for `alias_attribute` the bare pattern is what defines it. That
lives in `defineAliasAccessor`, a `@noRailsEquivalent` helper that emits one
get/set property where Rails emits a reader from the bare pattern and a writer
from the `attribute=` pattern.

Two deviations are stacked here: the bare pattern generating nothing for the
canonical name, and the writer riding on the reader's property instead of coming
from its own pattern. Both trace to the same root — readers are accessor
properties rather than generated methods — and neither is a TypeScript
shortcoming: `Object.defineProperty` can install a plain method just as easily.

## Converged shape

The bare pattern generates the reader for every name it is asked about, canonical
or alias, through the same `define_proxy_call` path every other pattern uses, and
the writer comes from the `attribute=` pattern. `attribute()` stops defining the
accessor itself. `defineAliasAccessor` disappears with its `@noRailsEquivalent`
tag.

This is deliberately larger than it looks — every read path that assumes
`record.name` resolves to an accessor property (dirty tracking, attribute
inspection, the `strict` unknown-attribute setter dispatch) has to be checked —
so it wants its own PR, and possibly its own RFC.

## Acceptance criteria

- [ ] `defineAttributeMethodPattern` has no bare-pattern carve-out; the arm
      generates for canonical and alias names alike.
- [ ] `defineAliasAccessor` and its `@noRailsEquivalent` tag are gone.
- [ ] The alias reader still raises `MissingAttributeError` for an
      uninitialized attribute, and the alias writer still routes to
      `writeAttribute` on the canonical name.
- [ ] AR + AM suites green on all three adapters; `pnpm parity:api:extra` for
      both packages does not grow.
