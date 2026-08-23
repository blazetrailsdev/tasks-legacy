---
title: "CollectionAssociation#replace does not call transaction; the ReplacePlan split moved it"
status: ready
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CollectionAssociation#replace` (`activerecord/lib/active_record/associations/collection_association.rb:249-256`) is:

```ruby
def replace(other_array)
  other_array.each { |val| raise_on_type(val) }
  original_target = skip_strict_loading { load_target }.dup

  if owner.new_record?
    replace_records(other_array, original_target)
  else
    replace_common_records_in_memory(other_array, original_target)
    if other_array != original_target
      transaction { replace_records(other_array, original_target) }
    else
      other_array
    end
  end
end
```

trails' `replace` (`packages/activerecord/src/associations/collection-association.ts`) does not
call `transaction` at all: the synchronous writer path returns a `ReplacePlan`
and the transaction is opened later, in `persistReplacePlan`
(`collection-association.ts:775-791`). That split is why
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/collection-association.json`
still carries the `replace` -> `transaction` row after PR #6890 migrated that
shard's two `empty?` rows to `@missingRailsCall` receipts — the row was
deliberately retained because the deviation is owned convergence work, not a
language shortcoming (RFC 0106 rule at
`scripts/api-compare/missing-rails-call-tags.ts:296-299`).

One Rails method is two trails methods, and the `transaction` boundary moved
with it: in Rails the transaction wraps `replace_records` only, on the
non-new-record arm, and only when `other_array != original_target`.

## Converged shape

`replace` opens the transaction itself, at Rails' branch and with Rails' guard,
so the `transaction { replace_records(...) }` call is visible in the body that
`collection_association.rb:249-256` maps onto. Whether `persistReplacePlan`
survives as a private helper is secondary — what has to converge is that
`replace` is the method that calls `transaction`.

Delete the `replace` -> `transaction` row from
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/collection-association.json`
by hand (only-shrink; do not reseed), and run
`pnpm parity:api:calls:tighten activerecord/associations/collection-association.json`
if the mark goes stale.

## Acceptance criteria

- [ ] `replace` calls `transaction`, on Rails' arm and under Rails' guard.
- [ ] The `replace` -> `transaction` baseline row is deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
