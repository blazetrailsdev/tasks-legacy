---
title: "Run remove_target!'s awaitable arms inside has_one replace, not from detachDisplacedTarget"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Landed in PR #6436 (`inline-has-one-replace-transaction-block`), which pulled
Rails' `replace` body back into one method
(`vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb:59-85`).
One half of `remove_target!` still does not run where Rails runs it.

`packages/activerecord/src/associations/has-one-association.ts`, `replace`'s
`save = false` arm now runs `remove_target!`'s default branch in memory —
`nullify_owner_attributes` + `remove_inverse_instance`
(has_one_association.rb:104-106). It still does NOT run:

- the `:delete` arm, `target.delete` (has_one_association.rb:97),
- the `:destroy` arm, `destroyed_by_association=` + `target.destroy` (:99-101),
- the nullify arm's `target.save` and its `RecordNotSaved` raise (:107-113).

All three need an `await` a synchronous arm cannot issue, so the awaitable
callers run them out of line via `detachDisplacedTarget`
(`has-one-association.ts`, called from `builder/has-one.ts`'s `build#{name}` /
`create#{name}`, `_createRecord`, and nested-attributes.ts'
`detachDisplacedThenSetNewRecord`). A caller that reaches `replace(record,
false)` WITHOUT going through one of those — `syncWrite`, i.e. mass assignment
onto an unpersisted owner — therefore leaves a persisted displaced row attached
in the DB where Rails would have detached it.

## Converged shape

`replace` runs `remove_target!` whole, at has_one_association.rb:69, for every
caller — which means the `save = false` arm becomes awaitable too, or the
remaining synchronous callers (`syncWrite`, i.e. RFC 0087 §1's mass-assignment
arm) are retired first so no synchronous path into `replace` survives. Note
RFC 0087's `retire-sync-association-mass-assignment-arms` is the natural
predecessor: with `syncWrite` gone, every `replace` caller can await, and
`detachDisplacedTarget` disappears with the split.

## Acceptance criteria

- [ ] `remove_target!`'s `:delete`, `:destroy` and `target.save` arms run from
      inside `replace`, at has_one_association.rb:69, on every path.
- [ ] `detachDisplacedTarget` is gone, or its remaining callers are justified
      at the call site.
- [ ] A mass-assignment displacement over a persisted displaced row detaches it,
      matching Rails.
- [ ] `pnpm parity:api:calls` green.
