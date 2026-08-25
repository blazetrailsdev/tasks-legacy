---
title: "find_some_ordered hand-rolls in_order_of and inverts Rails' branch order"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `findSomeOrdered`'s `except` call row in PR #6584
(RFC 0106 wave 2). The `except("limit","offset")` → `where` → `select` →
`records()` prefix now matches Rails line for line; the tail does not.

Rails (`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:567-580`):

```ruby
result = relation.records

if result.size == ids.size
  result.in_order_of(:id, ids.map { |id| model.type_for_attribute(primary_key).cast(id) })
else
  raise_record_not_found_exception!(ids, result.size, ids.size)
end
```

trails' `packages/activerecord/src/relation/finder-methods.ts` `findSomeOrdered`
instead inverts the branch (raises first, on `records.length !== ids.length`)
and then hand-rolls the ordering: it builds a `Map` of cast id → index and
`sort`s the result array with a comparator that falls back to `0` for an id it
cannot find. Two consequences:

- **`Enumerable#in_order_of` is never called.** trails has `inOrderOf` (it is
  sited in `relation.ts:1146` for the relation-level version), so the
  activesupport `Enumerable#in_order_of` is the piece to check for/port.
- **The `?? 0` fallbacks silently reorder** where Rails would simply not match.
  Unreachable today because the size check runs first, but it encodes a
  different contract than Rails'.

Note the branch inversion is itself a control-flow divergence: Rails' happy path
is the `if`, trails' is the fallthrough.

## Acceptance criteria

- [ ] `findSomeOrdered` calls `in_order_of`'s TS counterpart on the result array
      rather than sorting through a bespoke index map.
- [ ] Branch order matches Rails: the `result.size == ids.size` arm first, the
      raise in the `else`.
- [ ] The `?? 0` comparator fallbacks are gone with the hand-rolled sort.
- [ ] `pnpm parity:api:calls` does not regress; if the change retires the
      `find_some_ordered` row, delete it by hand and tighten the mark.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
