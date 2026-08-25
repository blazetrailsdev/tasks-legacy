---
title: "OptimizerHints visitor owns maybe_visit's leading space"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while converging `collect_optimizer_hints` during PR #6358.

Rails splits the leading space and the comment body across two methods
(`vendor/rails/activerecord/lib/arel/visitors/to_sql.rb`):

```ruby
def collect_optimizer_hints(o, collector)   # :887
  maybe_visit o.optimizer_hints, collector
end

def maybe_visit(thing, collector)            # :891
  return collector unless thing
  collector << " "
  visit thing, collector
end

def visit_Arel_Nodes_OptimizerHints(o, collector)  # :170
  hints = o.expr.map { |v| sanitize_as_sql_comment(v) }.join(" ")
  collector << "/*+ #{hints} */"
end
```

`maybe_visit` emits the space; `visit_Arel_Nodes_OptimizerHints` emits
`/*+ ... */` with none. The port folds the space into the visitor —
`packages/arel/src/visitors/to-sql.ts#visitArelNodesOptimizerHints` appends
`` ` /*+ ${hints} */` `` with a leading space — so `collectOptimizerHints` could
not route through `maybeVisit` until #6362 did, and the pair now double-counts
the space only because the visitor still owns it. The rendered SQL is identical
today; the divergence is in which method owns the separator, and it makes the
visitor wrong for any OTHER caller that reaches it through a path expecting
Rails' no-space contract.

## Converged shape

`visitArelNodesOptimizerHints` appends `/*+ ${hints} */` with NO leading space,
exactly as `to_sql.rb:170-172`; the space stays in `maybeVisit`
(`to_sql.rb:891-895`), which `collectOptimizerHints` already calls after #6362.

## Acceptance criteria

1. `visitArelNodesOptimizerHints` emits no leading space; `maybeVisit` supplies it.
2. Emitted SQL is unchanged — `pnpm vitest run packages/arel/src/visitors` green,
   including the optimizer-hint cases in `to-sql.test.ts` and `mysql.test.ts`.
3. No new `parity:api:calls` / `parity:api:calls:args` rows.
