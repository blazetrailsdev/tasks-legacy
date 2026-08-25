---
title: "Port WhereClause#any? and spell include?'s having-clause guard as Rails does"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `Relation#include?` for RFC 0099 in PR #6363.

`WhereClause` in Rails delegates BOTH predicates over
(`vendor/rails/activerecord/lib/active_record/relation/where_clause.rb:8`):

```ruby
delegate :any?, :empty?, to: :predicates
```

trails' `WhereClause`
(`packages/activerecord/src/relation/where-clause.ts:34`) ports only
`isEmpty()`. `any()` was never ported, so every Rails body that reads
`clause.any?` is spelled as a negated `isEmpty()` here. The one that shows up
in the ratchet is `FinderMethods#include?`
(`vendor/rails/activerecord/lib/active_record/relation/finder_methods.rb:392`):

```ruby
if loaded? || offset_value || limit_value || having_clause.any?
```

against `packages/activerecord/src/relation.ts` (`include`), which reads
`!this._havingClause.isEmpty()`. That is the standing baseline row

```text
relation.ts | include? | having_clause
```

in `scripts/api-compare/call-mismatches-exclude/activerecord/relation.json`,
carrying the seeded RFC 0047 placeholder reason. PR #6363 converged the rest of
that body (the `id` local and the `exists?` call) but left this row, since
adding a method to `WhereClause` is a separate change from the `include?`
body shape.

Behaviour is identical today — this is a missing ported method, not a
behavioural gap, which is why it reads as "noise" in the baseline. It is still
a real gap: `any?` is a public Rails method on a ported class, so it also
counts against `parity:api`.

## Converged shape

Port `any()` onto `WhereClause` next to `isEmpty()`, delegating to the same
`predicates` array Rails delegates to, and spell the `include?` guard
`this._havingClause.any()` as `finder_methods.rb:392` does. Then delete the
`relation.ts | include? | having_clause` row by hand (only-shrink; never
`--write`).

## Acceptance criteria

1. `WhereClause#any()` exists and mirrors `where_clause.rb:8`'s delegation to
   `predicates`.
2. `Relation#include?` reads `having_clause.any?` as Rails does, not a negated
   `isEmpty()`.
3. Every other `!<clause>.isEmpty()` that ports a Rails `any?` is converged in
   the same pass (grep `isEmpty()` across `relation/` and check each against
   its Ruby line) — do not leave a second spelling of the same call behind.
4. The `relation.ts | include? | having_clause` baseline row is deleted;
   `pnpm parity:api:calls` green with a smaller row count, `pnpm parity:api`
   delta non-negative.
