---
title: "add_constraints' one loop body is split across three invented helpers"
status: claimed
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: "2026-08-22T21:34:59Z"
assignee: "api-build-should-retire-tags-compare-reports-stale"
blocked-by: null
closed-reason: null
---

## Context

Rails' `add_constraints` is ONE method with one loop body
(`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb:124-159`).
trails splits that one body across four:

- `addConstraints` — the `last_chain_scope` / `each_cons(2)` / `reverse_each` head;
- `_mergeReflectionScopeChain` — the `reflection.constraints.each` inner loop;
- `_pushScopeIntoRelation` — the `if`/`elsif` arms plus `all_includes`,
  `unscope!`, `where_clause +=`, `order_values |`;
- `_mergeReferencedJoins` — the `elsif !item.references_values.empty?` arm.

None of the three underscore-prefixed helpers has a Rails counterpart; Rails
extracts only `eval_scope` and `apply_scope` here, and trails already has both.
The split forces a synthetic parameter list that Rails never needs — after
PR #6881, `_pushScopeIntoRelation(scope, evaluated, reflection?, isChainHeadScope?)`
threads `reflection` and a boolean through two frames purely to reconstruct what
Rails reads as closure locals inside the loop.

Surfaced while converging the chain head into the fold (#6881,
`add-constraints-head-scope-applied-outside-the-reverse-each-fold`). That story
converged the CONTROL FLOW onto Rails' single loop; the DECOMPOSITION is the
remaining half, and the two are separable — the fold had to land first.

## Converged shape

Inline the three helpers into `addConstraints`, so the method reads as Rails'
does: `chain.reverse_each` → `reflection.constraints.each` → `item = evalScope(…)`
→ the `if scope_chain_item == chain_head.scope` / `elsif !item.references_values
.empty?` arms inline → `reflection.all_includes { … }` → `unscope!` /
`where_clause +=` / `order_values |`. `chainHead`, `reflection` and `item`
become ordinary locals of the one body, and the `isChainHeadScope` parameter
disappears with the frame that needed it.

`evalScope` and `applyScope` STAY — Rails extracts those (:161-172).

Watch: `_mergeReferencedJoins` carries the same-klass / cross-klass
`Relation::Merger` split (merger.rb:118-150) in a long comment; that prose moves
with the code, it is not surface to drop.

## Acceptance criteria

- [ ] `_mergeReflectionScopeChain`, `_pushScopeIntoRelation` and
      `_mergeReferencedJoins` are gone; `addConstraints` is one method mirroring
      association_scope.rb:124-159 line for line.
- [ ] `pnpm parity:api:extra --package activerecord` loses the three names.
- [ ] Full `packages/activerecord/src/associations/` suite green on SQLite,
      PostgreSQL and MySQL/MariaDB.
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
