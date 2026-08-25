---
title: "Converge executeGroupedCalculation's groupNodes local and eager re-derivation onto calculations.rb:515-525"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`executeGroupedCalculation` (`packages/activerecord/src/relation/calculations.ts`)
diverges from `calculations.rb:525` in two related ways left behind by
`converge-execute-grouped-calculation-body-to-rails-source-order` (PR #6716):

1. **Local name.** Rails REASSIGNS the same local:
   `group_fields = relation.arel_columns(group_fields)` (calculations.rb:525),
   and everything downstream (`group_fields.map` at :530, `group_aliases.zip(group_fields)`
   at :534, `relation.group_values = group_fields` at :552) reads that one name.
   trails introduces a second local, `groupNodes`, and leaves `groupFields`
   holding the pre-resolution value. A Rails dev reading the two side by side
   sees a name that does not exist in the Ruby.
2. **`apply_join_dependency` re-derivation.** Rails' grouped arm never touches
   the join dependency: `calculate` folds it in once at calculations.rb:232
   before dispatching. trails re-derives it inside `executeGroupedCalculation`
   with an `isEagerLoading` guard and a callback that captures the yielded
   relation (`Relation` is thenable), which is a whole branch Rails' body does
   not have.

## Acceptance criteria

- [ ] The `arel_columns` result is held in `groupFields` (the Rails local),
      with no second `groupNodes` binding; the TS typing follows the value,
      not the other way round.
- [ ] The eager re-derivation either moves up into the `calculate` dispatch
      (calculations.rb:232) so the grouped body starts at :515 as Rails does,
      or the branch carries a call-site justification naming the specific
      TypeScript shortcoming that forces it.
- [ ] `pnpm parity:api:calls` / `:args` green; no new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
