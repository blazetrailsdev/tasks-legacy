---
title: "Wire Associations and Inheritance into the initialize_dup chain"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

trails#6812 built AR's `initialize_dup` prepend chain with three links — Core
(`vendor/rails/activerecord/lib/active_record/core.rb:550-562`),
Locking::Optimistic (`optimistic.rb:72-75`) and Timestamp (`timestamp.rb:50-53`)
— plus Aggregations' lazy link (`aggregations.rb:6-9`). Two Rails links in the
same chain were left out, both deliberately and both out of that PR's scope:

- `Associations#initialize_dup` (`vendor/rails/activerecord/lib/active_record/associations.rb:69-72`):

  ```ruby
  def initialize_dup(*) # :nodoc:
    @association_cache = {}
    super
  end
  ```

  `include Associations` is base.rb:317, so this link sits above Timestamp.
  trails has no equivalent: the copy gets its caches from `new ctor({})` in
  `dup`, so nothing resets them as part of the chain.

- `Inheritance#initialize_dup` (`vendor/rails/activerecord/lib/active_record/inheritance.rb:343-346`):

  ```ruby
  def initialize_dup(other)
    super
    ensure_proper_type
  end
  ```

  trails deliberately does NOT re-assert the STI type on dup — the deviation is
  documented in `packages/activerecord/src/persistence.ts` (~:1387-1398), which
  states the type column is carried solely by the deep-dup'd attributes. That is
  a deviation, not a decision: `include Inheritance` is base.rb:303.

## Converged shape

Both links are prepended onto `Base.prototype` in Rails' include order alongside
the existing three (see the wiring block near the bottom of
`packages/activerecord/src/base.ts`), each opening with `super_`. The
persistence.ts comment asserting trails does not re-run `ensure_proper_type` on
dup is deleted rather than reworded.

Note the ordering constraint the existing chain already documents: the initialize
callbacks must observe the source's `lock_version` / timestamps before Locking
and Timestamp clear them, so a new link's position in the include order matters.

## Acceptance criteria

- [ ] `Associations#initialize_dup` and `Inheritance#initialize_dup` are links in
      the chain, wired in base.rb include order.
- [ ] A duped STI record's type column is written by `ensure_proper_type` rather
      than only inherited from the duped attributes — covered by a test.
- [ ] `dup`, `inheritance`, `sti`, `associations`, `timestamp` and `locking`
      suites stay green on all four adapter lanes.
