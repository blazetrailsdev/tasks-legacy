---
title: "Collapse Attribute#dup onto initialize_dup so builder.rb's .dup call sites re-dup mutable values"
status: ready
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two PRs landed the same Ruby method from opposite ends and `main` now carries
both halves, un-joined:

- #7027 added `Attribute#deepDup()` and `private initializeDup()`
  (`packages/activemodel/src/attribute.ts:258-282`). `deepDup()` shallow-copies
  via `Object.assign(Object.create(proto), this)` and then calls
  `initializeDup`, which re-dups a duplicable `_value` through
  `isDuplicable` from `@blazetrails/activesupport`. That is a faithful
  `initialize_dup` (`vendor/rails/activemodel/lib/active_model/attribute.rb:155-159`).
- #7028 added `Attribute#dup()` (`attribute.ts:321`) as a bare shallow copy
  with **no** `initializeDup` call, to replace `attribute-set/builder.ts`'s
  `dupAttribute` / `LazyAttributeHash.cloneAttr`.

In Ruby there is one method. `Object#dup` is what _triggers_ `initialize_dup`,
and `Object#deep_dup` is `duplicable? ? dup : self`
(`activesupport/lib/active_support/core_ext/object/deep_dup.rb:16`) — so
`deep_dup` gets the value re-dup _through_ `dup`, never beside it.

The consequence on `main`: the three `attr.dup()` call sites in
`attribute-set/builder.ts` — `LazyAttributeSet#defaultAttribute` (`:132`,
Rails `builder.rb:84`), `LazyAttributeHash#deepDup` (`:245`, Rails
`builder.rb:120`), `LazyAttributeHash#assignDefaultValue` (`:339`, Rails
`builder.rb:175`) — all skip `initialize_dup`. Ruby runs it at every one of
them, because every one of them is spelled `.dup` in `builder.rb`. So a schema
default with a mutable cast value (a `Date`, an array) is shared between the
copy and the prototype where Rails separates them.

`Attribute#dup()`'s JSDoc on `main` discloses this gap and names this story.

## Converged shape

`dup()` owns the copy and runs `initializeDup`; `deepDup()` becomes Ruby's
`duplicable? ? dup : self` over it, rather than re-implementing the copy:

```ts
dup(): Attribute {
  const dup = Object.assign(Object.create(Object.getPrototypeOf(this) as object), this) as this;
  dup.initializeDup(this);
  return dup;
}

deepDup(): Attribute {
  return this.dup();
}
```

## Acceptance criteria

- `Attribute#dup()` runs `initializeDup` (attribute.rb:155-159); the copy
  itself is written once, not in both methods.
- `Attribute#deepDup()` delegates to `dup()` rather than duplicating its body
  (deep_dup.rb:16). Its existing behaviour and tests are unchanged.
- A regression test that fails on baseline: build a `LazyAttributeHash` (or an
  `AttributeSet` via `Builder`) whose `defaultAttributes` entry holds a mutable
  cast value, materialize it twice, mutate one copy's value in place, assert
  the other copy and the prototype are unchanged.
- The caveat paragraph in `Attribute#dup()`'s JSDoc naming this story is
  removed.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
