---
title: "ExtendedDeterministicUniquenessValidator.install_support should prepend rather than swap the prototype method"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by the RFC 0106 `@missingRailsCall` permanence audit (PR #6855). This
tag is the ONE of 95 `PERMANENT` claims the audit judged marginal rather than
clearly language-forced, and it was left at `PERMANENT` rather than downgraded
without a convergence plan. This story is that plan.

`activerecord/lib/active_record/encryption/extended_deterministic_uniqueness_validator.rb:7`
(`install_support`):

    ActiveRecord::Validations::UniquenessValidator.prepend(EncryptedUniquenessValidator)

`packages/activerecord/src/encryption/extended-deterministic-uniqueness-validator.ts:20`
does not call trails' `prepend()`. It swaps `validateEach` on the prototype by
hand, and its tag's stated reason is that `resetSupport` has to hand the ORIGINAL
method back for test teardown, which the `prepend()` shim's wrapper cannot
expose.

That reason rests on a limitation of the trails `prepend()` shim, not on a
TypeScript language shortcoming — so it is convergeable shim work, not a
permanent deviation. Rails itself needs no `reset_support`: `prepend` inserts a
module into the ancestor chain and `super` walks past it, so the original is
never lost and never needs handing back.

## Converged shape

Give `prepend()` a way to recover what it displaced — the ancestor-chain read
Ruby gets for free — so `installSupport` can be spelled as `prepend()` at
`extended_deterministic_uniqueness_validator.rb:7`'s call site and `resetSupport`
can undo it without the manual prototype swap. Then delete the
`@missingRailsCall prepend` tag.

Check first whether `resetSupport` is a trails invention at all: Rails has no
counterpart, so if the teardown it serves can be expressed by the interposed
module going away, the requirement driving the deviation disappears with it.

## Acceptance criteria

- [ ] `installSupport` calls `prepend()` at Rails' call site.
- [ ] Test teardown works without a hand-held reference to the original
      `validateEach`.
- [ ] The `@missingRailsCall prepend` tag is deleted — converged, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
