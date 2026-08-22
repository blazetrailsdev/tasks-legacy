---
title: "Port Relation's select_values / group_values writers so calculations stop assigning the backing fields"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 180
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails assigns relation values through the generated writers —
`relation.select_values = [...]` and `relation.group_values = group_fields`
(`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:484`,
`:552-553`), part of `Relation::VALUE_METHODS`
(`activerecord/lib/active_record/relation.rb`).

trails' `Relation` exposes `selectValues` and `groupValues` as READERS only, so
the calculation arms assign the backing stores directly —
`relation._selectColumns = [...]`, `relation._groupColumns = [...]` in
`packages/activerecord/src/relation/calculations.ts` (both grouped arms,
`executeSimpleCalculation`, `buildCountSubquery`). Reaching past the public
reader into the private field is the deviation; it also forces
`groupNodes as unknown as string[]` casts because `_groupColumns` is typed
`string[]` while Rails stores the `arel_columns`-resolved nodes.

## Converged shape

Give `Relation` the Rails value writers (`set`-prefixed only if a TS `set`
accessor cannot express it) for at least `selectValues` and `groupValues`,
typed to hold Arel nodes as Rails does, and have the calculation arms assign
through them.

## Acceptance criteria

- [ ] `Relation` exposes writers for `selectValues` / `groupValues` matching the
      Rails value-method names.
- [ ] No calculation arm assigns `_selectColumns` / `_groupColumns` directly,
      and the `as unknown as string[]` casts are gone.
- [ ] `calculations.test.ts`, `calculations.trails.test.ts` and
      `relations.test.ts` stay green (including `group with subquery in from
does not use original table name`, which depends on the resolved nodes
      being stored).

## Update from the 2026-08-18 RFC 0023 triage pass

**The main half is converged.** `_selectColumns` and `_groupColumns` no longer
exist anywhere in `packages/` or `scripts/`. `Relation` now exposes public
`selectValues` / `groupValues` writers and the calculation arms assign through
them, matching `calculations.rb:484` and `:552-553`:

- `relation/calculations.ts:446-447` — `relation.groupValues = groupNodes`,
  `relation.selectValues = selectValues`
- and at `:771`, `:911`, `:1002`, `:1307`, `:1313`, `:1379`.

**What remains is the typing half only.** `groupValues` is still declared
`string[]` (`relation/query-methods.ts:303`, and the mirroring host interface at
`relation/calculations.ts:158`) while Rails stores the `arel_columns`-resolved
nodes, so `calculations.ts:446` still has to launder them through
`as unknown as string[]`.

Scope this story to widening the `selectValues` / `groupValues` element type to
hold Arel nodes and deleting the resulting casts.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
