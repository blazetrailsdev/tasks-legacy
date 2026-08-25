---
title: "_load_from_sql collapses Rails' STI / homogeneous branch because it never sees the Result"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `_load_from_sql` collapses Rails' STI / homogeneous branch because it never sees the Result

## Context

Surfaced converging the `querying.json` call-set rows in PR #6735 (RFC 0106
wave 4c). Three rows — `_load_from_sql -> inheritance_column`,
`-> instantiate`, `-> instantiate_instance_of` — are all one divergence.

Rails (`vendor/rails/activerecord/lib/active_record/querying.rb:71-95`) branches
on the RESULT SET:

```ruby
message_bus.instrument("instantiation.active_record", payload) do
  if result_set.includes_column?(inheritance_column)
    result_set.indexed_rows.map { |record| instantiate(record, column_types, &block) }
  else
    # Instantiate a homogeneous set
    result_set.indexed_rows.map { |record| instantiate_instance_of(self, record, column_types, &block) }
  end
end
```

trails' `_loadFromSql` (`packages/activerecord/src/querying.ts:147-171`) takes
plain rows, not the `Result`, so it has no `includes_column?` to ask. It takes a
single `_instantiate` path for both arms, relying on `_instantiate`
short-circuiting the STI lookup when the inheritance column's value is absent.

`findBySql` (`querying.ts:47-60`) already HAS the `Result` and calls
`result.toArray()` plus `result.columnTypes` before handing them over, so the
information is thrown away one frame above the branch.

Note the second-order trap, which is why this is not a one-line fix: routing the
homogeneous arm through `instantiateInstanceOf`
(`packages/activerecord/src/persistence.ts:2174-2190`) also changes WHICH of
`_instantiate`'s two type maps the column types land in. `instantiate`
(`persistence.ts:150-173`) passes its `types` as `overrideTypes` so an explicit
per-attribute type beats the schema cast type (Rails' `LazyAttributeHash`
resolves `additional_types[name] || types[name]`), while `instantiateInstanceOf`
passes them as `columnTypes`, which known columns ignore. See the comment at
`persistence.ts:161-167`.

## Converged shape

`_loadFromSql` takes the `Result`, reads `includesColumn(inheritanceColumn)`,
and branches to `instantiate` / `instantiateInstanceOf` exactly as
`querying.rb:88-93` does — after reconciling the two type maps so both arms
carry the same semantics Rails gives them.

## Acceptance criteria

- [ ] `_loadFromSql` branches on the result set's inheritance-column presence.
- [ ] The STI arm calls `instantiate`, the homogeneous arm
      `instantiateInstanceOf(this, ...)`.
- [ ] The `overrideTypes` vs `columnTypes` split is resolved, not papered over;
      say in the PR which map each arm feeds and why.
- [ ] The three `querying.json` `kind: "set"` rows are deleted, then
      `pnpm parity:api:calls:tighten activerecord/querying.json`.
- [ ] STI suites green on all three lanes.
