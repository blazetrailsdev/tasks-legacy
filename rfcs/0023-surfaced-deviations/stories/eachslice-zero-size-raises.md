---
title: "eachSlice spins on a zero slice size where Ruby's each_slice raises"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `eachSlice` spins on a zero slice size where Ruby's `each_slice` raises

## Context

Surfaced in review of the `:destroy_async` arm (RFC 0106 wave 4d, PR #6762).
`HasManyAssociation#handle_dependency` calls
`ids.each_slice(owner.class.destroy_association_async_batch_size || ids.size)`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_association.rb:52`),
which the trails body now routes through `eachSlice`
(`packages/activesupport/src/enumerable-utils.ts:208-214`).

Ruby's `Enumerable#each_slice(0)` raises `ArgumentError: invalid slice size`.
The trails port is a bare `for (let i = 0; i < collection.length; i += n)`, so
`n === 0` loops forever and `n < 0` never advances either. Note this is NOT a
`||` vs `??` question at the call site: `0` is truthy in Ruby, so
`0 || ids.size` evaluates to `0` exactly as `??` does — both hand the slice
size straight through, and the divergence is entirely inside `eachSlice`.

The sibling `inGroupsOf` (`array-utils.ts:60`) already raises on a bad group
size (see `array-utils.trails.test.ts:63`), so the guard shape is settled in
this package.

## Acceptance criteria

- [ ] `eachSlice` raises `ArgumentError` with Ruby's `invalid slice size`
      message for a non-positive `n`, matching `Enumerable#each_slice`.
- [ ] The same audit is applied to the neighbouring chunker at
      `enumerable-utils.ts:195-203`, which shares the loop shape.
- [ ] A test pins the raise; existing `eachSlice` callers are checked for a
      caller that relied on the spin (none expected).
