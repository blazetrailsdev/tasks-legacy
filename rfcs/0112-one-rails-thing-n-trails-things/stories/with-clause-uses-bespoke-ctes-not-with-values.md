---
title: "Relation#with builds a bespoke _ctes array instead of with_values + build_with"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `build_with_value_from_hash`'s argument order (#6463).

Rails' `with` is a two-step: `WhereClause`-style `with_values` accumulate on the
relation, and `build_with(arel)`
(`activerecord/lib/active_record/relation/query_methods.rb:1908-1921`) turns
them into `Arel::Nodes::TableAlias` nodes via `build_with_value_from_hash`
(`:1923-1927`), handing them to `arel.with(...)` / `arel.with(:recursive, ...)`.

trails carries BOTH shapes. The ported `buildWithValueFromHash` /
`buildWithExpressionFromValue` live in
`packages/activerecord/src/relation/query-methods.ts:2359,2380` and are reached
only through the private forwarders `relation.ts:7023,7028` — nothing else
calls them. The live path is a trails invention: a `_ctes` array
(`query-methods.ts:373` `upsertCte`, `:443`, `:460`) that `relation.ts:4909`
and `query-methods.ts:2928` map into `Nodes.Cte` at build time.

So the ported Rails method is dead code and the shipped behaviour is a
parallel mechanism Rails does not have.

## Converged shape

- `withValues` accumulates as Rails does; `buildWith` is the only producer of
  the WITH clause, and it calls `buildWithValueFromHash`.
- `_ctes` / `upsertCte` and the two `new Nodes.Cte(...)` sites go away.
- `withRecursive` sets `@with_is_recursive` (`query_methods.rb:1920`).

## Acceptance criteria

- [ ] `relation/with.test.ts` stays green on all three adapters.
- [ ] No `_ctes` member remains on `Relation`.
- [ ] `buildWithValueFromHash` is reached from `buildWith`, not from a private
      forwarder with no other caller.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
