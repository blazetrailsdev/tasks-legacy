---
title: "Arel.sql ports only the no-binds arm, so the ? / :key placeholder form is unreachable"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "arel"
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

Surfaced while moving `Arel.sql` into `packages/arel/src/arel.ts` (PR #6357,
RFC 0084 `arel-sql-inlined-in-select-manager-lock`).

Rails' `Arel.sql` is a two-arm dispatch
(`vendor/rails/activerecord/lib/arel.rb:51-57`):

```ruby
def self.sql(sql_string, *positional_binds, retryable: false, **named_binds)
  if positional_binds.empty? && named_binds.empty?
    Arel::Nodes::SqlLiteral.new(sql_string, retryable: retryable)
  else
    Arel::Nodes::BoundSqlLiteral.new sql_string, positional_binds, named_binds
  end
end
```

trails' `sql` ports only the first arm — `sql(sqlString, { retryable })` —
so the `?` / `:key` placeholder form of `Arel.sql` does not exist. The
`BoundSqlLiteral` node itself IS ported
(`packages/arel/src/nodes/bound-sql-literal.ts`); what is missing is the
factory dispatch that reaches it. Callers that need one construct it directly
instead, bypassing the Rails entry point — `packages/activerecord/src/relation.ts:3710`
(`new Nodes.BoundSqlLiteral(...)`) and `relation.ts:6945`
(`buildNamedBoundSqlLiteral`). No non-test file calls `Arel.sql` with binds
because it cannot.

This is a real API gap, not cosmetics: `Post.where(Arel.sql("id = ?", 1))` is
documented Rails usage (arel.rb:41-48) and is unrepresentable today.

## Converged shape

`sql` takes Rails' full signature and both arms, returning
`SqlLiteral | BoundSqlLiteral`. Ruby's `*positional_binds` + `**named_binds`

- a `retryable:` kwarg needs the settled trails kwargs idiom rather than a
  literal transcription — pick it deliberately (a trailing options object cannot
  sit behind a rest parameter), and keep the Rails parameter NAMES
  (`sqlString`, `positionalBinds`, `namedBinds`, `retryable`).

Once the dispatch exists, `relation.ts:3710` and `relation.ts:6945` should
route through `Arel.sql` where their Rails counterparts do, rather than
constructing the node directly — check each against its Rails body before
changing it; some Rails sites legitimately call `BoundSqlLiteral.new`.

## Acceptance criteria

- `sql` implements both arms of `arel.rb:51-57`, with the Rails guard
  (`positional_binds.empty? && named_binds.empty?`) in the same direction.
- `Arel.sql("id = ?", 1)` and the `:key` form both produce a
  `BoundSqlLiteral` that renders through the existing visitor path.
- Direct `new Nodes.BoundSqlLiteral(...)` construction in `relation.ts` is
  re-checked against its Rails body and routed through `Arel.sql` where Rails
  does.
- No new public surface (`pnpm parity:api:extra --package arel`).
