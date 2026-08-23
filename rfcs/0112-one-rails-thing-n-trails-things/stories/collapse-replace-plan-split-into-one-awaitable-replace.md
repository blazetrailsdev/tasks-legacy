---
title: "CollectionAssociation#replace is split across a ReplacePlan and replaceRecordsInTransaction"
status: done
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: 6904
claim: "2026-08-23T10:42:30Z"
assignee: "call-set-migrator-cannot-tag-members-split-into-a-subdirectory"
blocked-by: null
closed-reason: null
---

## Context

`CollectionAssociation#replace`
(`vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:242-256`)
is ONE method that does its own `load_target` and, on the persisted arm, runs
`transaction { replace_records(other_array, original_target) }` inline:

```ruby
def replace(other_array)
  other_array.each { |val| raise_on_type_mismatch!(val) }
  original_target = skip_strict_loading { load_target }.dup
  ...
    if other_array != original_target
      transaction { replace_records(other_array, original_target) }
```

After PR #6902 trails' `replace`
(`packages/activerecord/src/associations/collection-association.ts`) DOES open
the transaction at Rails' branch and guard, so the `replace` -> `transaction`
call-set row is gone. What remains is the split the transaction rides out on:

- `replace` is synchronous, so the transaction's block is the named private
  helper `replaceRecordsInTransaction`, and the open transaction's promise is
  returned on a `ReplacePlan` (`collection-association.ts:24-35`) for the
  caller to await. `ReplacePlan` has no Rails counterpart.
- Rails' `original_target = skip_strict_loading { load_target }.dup` is DB I/O
  the sync body cannot run on the persisted arm, so the in-memory target
  stands in and `replaceRecordsInTransaction` re-reads the real baseline off
  `scope()` before diffing — a second, differently-shaped read of what Rails
  reads once, at the top of `replace`.
- Three call sites (`writer`, `assignIds`, `CollectionProxy#replace`) spell
  "call `replace`, then await `plan.pending`" where Rails just calls `replace`.

The whole split exists because a JS property setter cannot await (RFC 0068),
and `syncWrite` (`collection-association.ts:~178`) is its other half: it
refuses the persisted arm outright rather than deferring the writes.

## Converged shape

One `replace`, awaitable, doing its own `load_target` at the top the way
`collection_association.rb:244` does — no `ReplacePlan`, no
`replaceRecordsInTransaction`, no baseline re-read in a second method, and no
`plan.pending` step at the three call sites.

That is gated on the sync-entry paths that reach `replace` today: mass
assignment (`assignAttributes` returns nil and assigns inline, per
`activemodel/lib/active_model/attribute_assignment.rb:32-35`) and
`assignIds`. Retiring the plan means those paths either become awaitable or
keep refusing the persisted arm through `syncWrite` alone — decide that first,
since it is what actually blocks the collapse. Related: RFC 0087
`delete-collection-sync-writers` (done) removed the sync writer surface this
split was originally built for.

## Acceptance criteria

- [ ] `replace` is one method that loads its own `original_target` and calls
      `transaction { replace_records(...) }` inline, matching
      `collection_association.rb:242-256`.
- [ ] `ReplacePlan` and `replaceRecordsInTransaction` are deleted, and the
      three call sites call `replace` with no second step.
- [ ] No baseline row or `@noRailsEquivalent` tag is added; `pnpm parity:api:calls`,
      `pnpm parity:api:calls:args` and `pnpm parity:api:extra --package activerecord` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
