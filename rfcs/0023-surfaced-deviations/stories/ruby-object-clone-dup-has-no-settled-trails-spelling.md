---
title: "Ruby Object#clone / #dup has no settled trails spelling; each call site open-codes it"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby's `Object#clone` and `Object#dup` copy the ivars and then dispatch
`initialize_clone` / `initialize_dup`. trails has no settled spelling for that,
so each call site open-codes the allocate-and-copy dance.

Two instances exist today, and PR #6800 added the second:

- `packages/activemodel/src/model.ts`, `Model#dup` — `Object.create` +
  `Object.assign` + `initializeDup(this)`, mirroring Ruby `dup`.
- `packages/activemodel/src/attributes.ts`, the exported `freeze` link — the
  port of `Attributes#freeze` (`activemodel/lib/active_model/attributes.rb:150-153`):

  ```ruby
  def freeze # :nodoc:
    @attributes = @attributes.clone.freeze unless frozen?
    super
  end
  ```

  `@attributes.clone` is Ruby `Object#clone` on an `AttributeSet`, whose
  `initialize_clone` dups the inner hash
  (`activemodel/lib/active_model/attribute_set.rb:82-85`). trails spells it as
  `Object.create(Object.getPrototypeOf(attributes))` + `Object.assign` +
  `cloned.initializeClone(attributes)`.

The `initialize_clone` / `initialize_dup` halves are real Rails members and are
ported at their Rails names. Only the builtin `clone` / `dup` call itself has
no spelling, so every new port that needs one re-derives four lines of
prototype plumbing at its call site, and a reviewer has to re-verify each one
copies ivars the way Ruby does.

## Converged shape

Decide on one spelling and use it at both sites. The candidates, in order:

1. An activesupport helper (Ruby's `clone`/`dup` are `Object` methods, so a
   shared function is the honest mirror of "every object answers this"), named
   `clone` / `dup`, dispatching `initializeClone` / `initializeDup` when the
   receiver defines them. Costs a `@noRailsEquivalent PERMANENT` — JS has no
   `Object#clone`.
2. A `clone()` / `dup()` method on each class that needs one, alongside the
   already-ported `initializeClone` / `initializeDup`.

Whichever wins, it is ratified once in CLAUDE.md next to the other settled
idioms (`setX()` for async setters, `include()`/`Included<>` for Ruby
`include`) so the next port does not re-derive it.

## Acceptance criteria

- One spelling, used by both `Model#dup` and the `Attributes#freeze` link, with
  the Ruby semantics documented once rather than per call site.
- `attributes.test.ts`'s `can't modify attributes if frozen` and
  `attributes can be frozen again` stay green (they are the freeze coverage),
  along with the `dup` cases.
- `pnpm parity:api:extra` reports no new unexplained novel surface.
