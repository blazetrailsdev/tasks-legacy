---
title: "Port Core#init_internals' four missing assignments"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 59
pr: 6933
claim: "2026-08-23T18:20:09Z"
assignee: "sweep-trails-only-test-files-associations"
blocked-by: null
closed-reason: null
---

## Context

trails#6812 made `Core#init_internals` the root of AR's `init_internals` prepend
chain, but did not touch its body, which is still missing four of Rails'
assignments. Rails
(`vendor/rails/activerecord/lib/active_record/core.rb:834-849`):

```ruby
def init_internals
  @readonly                 = false
  @previously_new_record    = false
  @destroyed                = false
  @marked_for_destruction   = false
  @destroyed_by_association = nil
  @_start_transaction_state = nil

  klass = self.class

  @primary_key         = klass.primary_key
  @strict_loading      = klass.strict_loading_by_default
  @strict_loading_mode = klass.strict_loading_mode

  klass.define_attribute_methods
end
```

`initInternals` in `packages/activerecord/src/core.ts` covers `@readonly`,
`@previously_new_record`, `@destroyed`, `@destroyed_by_association`,
`@strict_loading` and `@strict_loading_mode`. Absent:

- `@marked_for_destruction = false` — trails carries this on
  AutosaveAssociation instead.
- `@_start_transaction_state = nil` — trails assigns it only in
  `Transactions#init_internals` (`transactions.rb:432-437`), which also sets it;
  Rails sets it in BOTH, and Core's is the one that runs for a record whose
  class never reached the Transactions link.
- `@primary_key = klass.primary_key` — trails has no per-instance primary-key
  slot; every reader goes to the class. This is the largest of the four and may
  need its own investigation.
- `klass.define_attribute_methods` — trails generates attribute methods lazily
  elsewhere. This is the call that makes Rails' `attribute_types` schema load
  synchronous at construction, which is the same lazy-reflection gap that
  motivated the (now-removed) `ensure_proper_type` membership guard.

## Converged shape

`initInternals` in `core.ts` mirrors core.rb:834-849 line for line. Each of the
four omissions either lands or carries a `@missingRailsCall` reason at the call
site — not a silent absence. Split the `@primary_key` slot out into its own
story if it turns out to be load-bearing.

## Acceptance criteria

- [ ] `@marked_for_destruction` and `@_start_transaction_state` are assigned by
      `Core#init_internals`, in Rails' order.
- [ ] `define_attribute_methods` is either called or carries a reasoned
      `@missingRailsCall` at the call site.
- [ ] `@primary_key` is resolved (ported, or split to a follow-up story with the
      finding recorded).
- [ ] `parity:api:calls` shows no new baseline rows.
