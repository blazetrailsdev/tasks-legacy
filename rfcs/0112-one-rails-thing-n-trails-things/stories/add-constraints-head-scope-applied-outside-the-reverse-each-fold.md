---
title: "add_constraints applies the chain head's scope outside the reverse_each fold (association_scope.rb:131-156)"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: 6881
claim: "2026-08-22T20:35:01Z"
assignee: "parity-api-build-must-not-drop-harvested-tags"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6871 (`merge-reflection-scope-chain-injects-a-klass-override-shim`),
which converged the chain fold onto Rails' `chain.reverse_each` but left the
CHAIN HEAD's own scope handled outside that loop.

Rails' `add_constraints`
(`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb:131-156`)
has exactly ONE loop body for every chain position, and the head's own scope is
just one `scope_chain_item` inside it:

```ruby
chain_head = chain.first
chain.reverse_each do |reflection|
  reflection.constraints.each do |scope_chain_item|
    item = eval_scope(reflection, scope_chain_item, owner)

    if scope_chain_item == chain_head.scope
      scope.merge! item.except(:where, :includes, :unscope, :order)
    elsif !item.references_values.empty?
      ...
    end
    ...
    scope.where_clause += item.where_clause
    scope.order_values = item.order_values | scope.order_values
  end
end
```

So the head's scope is evaluated by `eval_scope` like any other item (against
`reflection.build_scope(reflection.aliased_table)`, :167-170), then merged with
`except(:where, :includes, :unscope, :order)` PLUS the unconditional
`where_clause +=` / `order_values |` at the bottom of the body.

trails instead applies the head scope in a separate branch above the fold
(`packages/activerecord/src/associations/association-scope.ts`, `addConstraints`,
the `isThrough ? head.scope : head.scopeFor` split invoking the lambda directly
onto the live `scope` relation), and `_mergeReflectionScopeChain` then SKIPS the
item that is `=== chainHead.scope`. Two divergences fall out of that:

- the `isThrough` / `scopeFor` split has no Rails counterpart — Rails reads
  `reflection.constraints` uniformly and never asks whether the head is a
  through;
- the head scope is applied by direct lambda invocation onto the main relation
  rather than by `eval_scope` + `merge!(item.except(...))` + `where_clause +=` /
  `order_values |`, so a head scope carrying `limit` / `select` / `joins` merges
  with different precedence than Rails gives it.

## Converged shape

- Delete the separate head-scope branch in `addConstraints` and the
  `c === chainHead.scope` skip in `_mergeReflectionScopeChain`.
- Fold the head inside the same loop as every other entry, and render Rails'
  `if scope_chain_item == chain_head.scope` arm as
  `scope.merge!(item.except(:where, :includes, :unscope, :order))` — the trails
  `merge` with the same four keys excepted — before the shared
  `where_clause +=` / `order_values |` push that `_pushScopeIntoRelation`
  already implements.
- `_mergeReflectionScopeChain` keeps its `chainHead` parameter (Rails'
  `chain_head` closure local), now driving a merge arm rather than a skip.

## Acceptance criteria

- [ ] No `isThroughReflection()` / `scopeFor` split for the chain head in
      `addConstraints`.
- [ ] The head's scope reaches the relation through the same
      `evalScope` + push path as every other chain item, with Rails'
      `except(:where, :includes, :unscope, :order)` merge arm.
- [ ] Full `packages/activerecord/src/associations/` suite green on SQLite,
      PostgreSQL and MySQL/MariaDB.
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
