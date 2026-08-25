---
title: 'Spell select_for_count''s :all Symbol as ":all" instead of the SQL literal "*"'
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `select_for_count` onto `with_connection`
(PR #6570, RFC 0106).

Rails' `select_for_count` returns the Symbol `:all` when there are no select
values (`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:645-653`),
and `perform_calculation` branches on it (`:436-449`):

```ruby
column_name ||= select_for_count
if column_name == :all
  ...
```

`:all` also reaches `aggregate_column` (`:414-423`), which maps it to
`Arel.star`, and `build_count_subquery?` / `build_count_subquery`
(`:655-678`).

trails spells that Symbol as `"*"` — and then has to accept `"all"` as a second
literal at every branch, because callers pass the Rails name through:

```ts
// relation/calculations.ts
if (columnName === "*" || columnName === "all") { ... }
// "*" is the JS analogue of Rails' `:all` symbol (calculations.rb:646-654).
```

CLAUDE.md's settled rule is that a Ruby Symbol whose value drives control flow
keeps its leading colon in the string — `":all"` — precisely so a reader can
tell the Symbol arm from a user-supplied column string. `"*"` is a SQL literal a
user can legitimately pass as a column name, so the two are not distinguishable
here, and the doubled `=== "*" || === "all"` guard is the symptom.

## Converged shape

Spell the Symbol `":all"` and branch on it, in `select_for_count`,
`perform_calculation`, `aggregate_column`, `build_count_subquery?` and
`build_count_subquery` — the five sites `calculations.rb` names it at. A
user-passed `"*"` / `"all"` string then stays a column string and takes the
non-Symbol arm, as it does in Ruby.

Audit the public entry points on the way through: `count("*")` and
`count(:all)` are both legal in Rails, so whatever normalization the public
`count` does today has to keep both working while the internal value becomes
`":all"`.

Related: [[count-select-for-count-full-convergence]] (the rest of
`select_for_count`'s body) and
[[converge-count-subquery-all-distinct-select-values]] (the `:all` + distinct
arm of `build_count_subquery`) both touch these same branches — check them
before starting so the three do not collide.

## Acceptance criteria

- [ ] `selectForCount` returns `":all"`, not `"*"`.
- [ ] No `=== "*" || === "all"` doubled guard remains in
      `relation/calculations.ts`; each site branches on the Symbol alone.
- [ ] `count("*")`, `count("all")` and the internal Symbol path all still
      produce the Rails SQL, with `calculations.test.ts` green.
- [ ] No new call-SET or call-ARG rows.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
