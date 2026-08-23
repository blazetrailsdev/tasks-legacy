---
title: "Route AR dup's attribute duplication through the initialize_dup links"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
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

trails#6812 converged AR's `initialize_dup` onto a real `prepend()` super chain,
so `Core#initialize_dup` (`vendor/rails/activerecord/lib/active_record/core.rb:550-562`)
now supers down into ActiveModel's links on `Model.prototype`:
`Attributes#initialize_dup` (`vendor/rails/activemodel/lib/active_model/attributes.rb:111-114`),
`Validations#initialize_dup` (`vendor/rails/activemodel/lib/active_model/validations.rb:310-313`)
and `Dirty#initialize_dup` (`vendor/rails/activemodel/lib/active_model/dirty.rb:248-251`).

Those links are **reachable but inert**. Measured on #6812: a test asserting both
their effects (fresh `Errors` on the copy, own mutation tracker carrying the
source's pending changes) passes unchanged against the pre-#6812 baseline.

The cause is `dup` in `packages/activerecord/src/persistence.ts` (~:1330-1400). Ruby's
`Object#dup` allocates a shallow copy and then calls `initialize_dup`, so
`@attributes`, `@errors` and the trackers are all still the SOURCE's objects when
the chain runs — which is exactly why each link exists. trails instead builds the
copy with `new ctor({})` and then, before entering the chain, hand-rolls what the
links do: `init_attributes` (deep_dup + `reset(primary_key)` + the
FromUser-over-default rebuild), `_dirty.snapshot(dupedAttrs)` and
`reinstateNewRecordChanges`. The copy therefore already has a fresh `Errors` and a
correctly-bound tracker, and each AM link re-establishes state it already had.

## Converged shape

`dup` allocates the copy and hands attribute duplication to the chain:
`Core#initialize_dup` does `@attributes = init_attributes(other)` (core.rb:551),
`Attributes#initialize_dup` does `@attributes = @attributes.deep_dup`,
`Validations#initialize_dup` replaces the errors and `Dirty#initialize_dup`
rebuilds the tracker — each on state that is still the source's when the link
runs. `dup`'s hand-rolled `init_attributes` / snapshot / reinstate block goes
away, and the links become load-bearing (removing one reds a test).

Note `Core#init_attributes` is itself a real Rails method
(`core.rb`, exported from `packages/activerecord/src/core.ts` as `initAttributes`)
— the port should call it rather than keeping the inlined copy in `dup`.

## Acceptance criteria

- [ ] `dup` no longer performs `init_attributes` / dirty-snapshot /
      `reinstateNewRecordChanges` itself; the chain does it.
- [ ] Deleting any one of the three ActiveModel `initialize_dup` links reds a
      test (verify by hand before opening the PR).
- [ ] `dup_test.ts`, `timestamp`, `locking`, `aggregations` and `dirty` suites
      stay green on all four adapter lanes.
