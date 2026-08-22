---
title: "Inline to-sql.ts's extracted helpers and unify compile"
status: done
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: ["arel"]
deps: []
deps-rfc: []
est-loc: 180
priority: 7
pr: 6856
claim: "2026-08-22T12:20:33Z"
assignee: "arel-operator-spellings-in-conventions"
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/visitors/to-sql.ts` carries **4 novel names and 0 moved** —
the highest all-novel count in the package
(`pnpm parity:api:extra --package arel`, 2026-08-22).
Rails: `vendor/rails/activerecord/lib/arel/visitors/to_sql.rb`.

| name                      | line                                                 | note                                                                                                |
| ------------------------- | ---------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `resolveValueForDatabase` | `to-sql.ts:30` (module-level export, used at `:297`) | Rails inlines the `value_for_database` handling in `visit_ActiveModel_Attribute` (`to_sql.rb` ~756) |
| `cteRelationSelfWraps`    | `:85` (used at `:1262`)                              | Rails inlines the check in `visit_Arel_Nodes_Cte`                                                   |
| `compileWithCollector`    | `:1797`                                              | Rails' `ToSql` has `compile(node, collector = Arel::Collectors::SQLString.new)` and nothing else    |
| `compileWithBinds`        | `:1807`                                              | same — the bind-collector variant is a trails split of `compile`                                    |

All four are triage category 3: Rails inlines them, so inline them. The two
`compile*` methods are the interesting pair — collapsing them back onto a
single `compile` whose collector is an argument is the Rails shape, and the
callers (`to-sql.ts:201`, `:208`, `tree-manager.ts:70`, plus
`packages/activerecord/src` — grep before starting) pass the collector.

`visitors/postgresql.ts` also exposes a novel `PostgreSQLWithBinds`
(1 novel, 0 moved) which is a sibling of the same split; retire it here if it
falls out of the `compile` unification, otherwise leave it to its own story
and say so.

## Acceptance criteria

- `resolveValueForDatabase` and `cteRelationSelfWraps` inlined at their single
  use sites and deleted from the module's exports.
- `compileWithCollector` / `compileWithBinds` collapsed onto a Rails-shaped
  `compile`, or — if the bind path genuinely cannot share one signature — the
  PR body states which TypeScript shortcoming forces the split and it costs one
  tag from the RFC's budget.
- `pnpm parity:api:extra --package arel` for `visitors/to-sql.ts`: novel
  **4 → 0** (or 4 → 2 with one documented tag).
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` clean — inlining a
  helper into a ported body is precisely what those gates measure.
- `pnpm vitest run packages/arel/src/visitors` green, plus the AR
  bind-parameter suites.
