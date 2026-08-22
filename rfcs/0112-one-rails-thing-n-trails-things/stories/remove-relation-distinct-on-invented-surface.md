---
title: "Remove Relation#distinctOn — Rails has distinct_on only on Arel::SelectManager"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 80
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while moving the projection family out of `relation.ts` in PR #6629
(RFC 0107 split 2/4). `distinct` moved to `relation/query-methods.ts`;
`Relation#distinctOn` was left behind in `relation.ts:1096-1101` because it has
no counterpart to move to.

Rails has no `ActiveRecord::Relation#distinct_on`. The only `distinct_on` in the
whole Rails tree is
`arel/lib/arel/select_manager.rb:163-166`, an Arel-level setter:

```ruby
def distinct_on(value)
  @ctx.set_quantifier = value ? Arel::Nodes::DistinctOn.new(value) : nil
  self
end
```

The trails method is an invented Relation-level surface backed by an invented
`_distinctOnColumns` field:

- `packages/activerecord/src/relation.ts:1096` — `distinctOn(...columns)`
- `packages/activerecord/src/relation.ts:495` — `private _distinctOnColumns`
- `packages/activerecord/src/relation.ts:4367` — copied in the clone/spawn path
- `packages/activerecord/src/relation/query-methods.ts:268` — declared on
  `QueryMethodsHost` so the Arel builders can read it

It carries no `@noRailsEquivalent` tag, so it is unaccounted extra surface.

## Converged shape

Remove `Relation#distinctOn` and `_distinctOnColumns`. A PostgreSQL
`DISTINCT ON` is reached in Rails through Arel — `SelectManager#distinct_on`
with an `Arel::Nodes::DistinctOn`, which trails' arel package already models —
so any caller that needs it composes the Arel node rather than going through a
Relation method Rails does not have.

Audit call sites first: if a trails-internal builder depends on the field,
route it through the Arel setter; if only tests reach for it, the tests move to
the Arel surface. If a caller turns out to need a Relation-level entry point
that Rails genuinely lacks, that is the point to `pnpm tasks block` with the
specific caller — not to keep the method with a fresh justification.

## Acceptance criteria

- [ ] `Relation#distinctOn` and `_distinctOnColumns` are gone from
      `relation.ts`, the spawn/clone copy at `:4367`, and the
      `QueryMethodsHost` declaration in `relation/query-methods.ts`.
- [ ] `DISTINCT ON` still emits for the PG paths that used it, via
      `SelectManager#distinct_on` / `Arel::Nodes::DistinctOn`
      (`arel/lib/arel/select_manager.rb:163-166`).
- [ ] `pnpm parity:api:extra --package activerecord` drops the novel names.
- [ ] `pnpm parity:api:calls` / `:args` clean.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
