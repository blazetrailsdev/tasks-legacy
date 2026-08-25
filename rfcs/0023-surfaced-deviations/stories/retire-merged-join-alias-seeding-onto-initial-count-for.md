---
title: "Retire seedJoinClauseAliases: raw joins should reach AliasTracker.create as Arel nodes"
status: draft
updated: 2026-08-20
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

## Context

`packages/activerecord/src/relation/merged-join-alias-tracker.ts`
(`seedJoinClauseAliases`) has no Rails counterpart and says so in its own header
comment. Rails shares one `alias_tracker(leading_joins + join_nodes, aliases)`
across every join dependency in `build_joins`
(`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:1894`),
and raw join clauses ride `joins_values` as Arel nodes, counted by
`AliasTracker.initial_count_for`
(`associations/alias_tracker.rb:29-49`) when the tracker is created. trails
instead resolves raw joins into a bespoke `_joinClauses` array and then claims
their tables into the tracker in a second pass — including a `where.associated`
arm that re-derives the `{plural_name}_{owner_table}` candidate by hand rather
than from a reflection.

PR #6751 converged that arm's call onto `AliasTracker#aliasedTableFor`, but the
seeding pass itself is still invented surface (2 novel names, no Rails file
maps onto it).

## Converged shape

- Raw joins reach the tracker as `Arel::Nodes::Join` nodes through
  `AliasTracker.create`'s `joins` argument, so `initial_count_for` counts them —
  the Rails path — and `seedJoinClauseAliases` is deleted.
- The `where.associated` candidate comes from the reflection's
  `alias_candidate` (`reflection.rb`), not from a hand-built
  `pluralize(underscore(assoc))`.

## Acceptance criteria

- [ ] `relation/merged-join-alias-tracker.ts` is deleted, or reduced to a call
      that hands Arel join nodes to `AliasTracker.create`.
- [ ] `pnpm parity:api:extra --package activerecord` loses the 2 novel names
      that file contributes.
- [ ] Merged/manual-join alias suites green on SQLite, PostgreSQL and
      MySQL/MariaDB (`left-outer-join-association`,
      `inner-join-association`, `relation.trails`).
