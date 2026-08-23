---
title: "JoinDependency#instantiate partitions by aliasSet.has, not Rails' alias-pattern match?"
status: in-progress
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6906
claim: "2026-08-23T11:12:29Z"
assignee: "wave-5g-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

`JoinDependency#instantiate`
(`activerecord/lib/active_record/associations/join_dependency.rb:114-123`)
partitions the result columns by testing each name against Rails' join-alias
pattern:

```ruby
result_set.columns.each do |name|
  column_names << name unless /\At\d+_r\d+\z/.match?(name)
end
```

trails' `instantiate` (`packages/activerecord/src/associations/join-dependency.ts:738`)
tests `aliasSet.has(key)` instead — it holds the exact alias set the `aliases`
object emitted, so it re-derives the partition from membership rather than from
the alias spelling. That is why
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/join-dependency.json`
still carries the `instantiate` -> `match?` row after PR #6890 migrated that
shard's `empty?` and `map` rows to `@missingRailsCall` receipts — the row was
deliberately retained because the deviation is a design choice, not a language
shortcoming (RFC 0106 rule at
`scripts/api-compare/missing-rails-call-tags.ts:296-299`).

The two partitions agree today, but they are not the same predicate: a
user-supplied `select` naming a column that happens to match `t\d+_r\d+` is
dropped by Rails and kept by trails, and an alias the `aliases` object emits
that does NOT match the pattern is kept by Rails and dropped by trails. Rails'
own comment on the regex is that it is the heuristic; trails silently improved
it, which is exactly the "no abstraction Rails does not have" case.

## Converged shape

`instantiate` tests the column name against the Rails pattern:

```ts
for (const name of resultSet.columns) {
  if (!/^t\d+_r\d+$/.test(name)) columnNames.push(name);
}
```

with the `aliasSet` read deleted if nothing else needs it. Keep Rails' anchors
(`\A`/`\z` are `^`/`$` on a JS regex with no `m` flag).

Delete the `instantiate` -> `match?` row from
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/join-dependency.json`
by hand (only-shrink; do not reseed), and run
`pnpm parity:api:calls:tighten activerecord/associations/join-dependency.json`
if the mark goes stale.

## Acceptance criteria

- [ ] `instantiate` partitions the result columns by the
      `/\At\d+_r\d+\z/` pattern, matching `join_dependency.rb:120-122`.
- [ ] The `instantiate` -> `match?` baseline row is deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green; the eager-load and
      `includes` suites in particular.
