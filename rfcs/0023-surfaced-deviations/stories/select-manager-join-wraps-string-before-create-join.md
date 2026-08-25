---
title: "SelectManager#join wraps a String relation in SqlLiteral before create_join; Rails passes it raw"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

`Arel::SelectManager#join` hands `create_join` the ORIGINAL `relation` — a raw
String included — and lets the join node hold it. trails converts a String to a
`SqlLiteral` first, so the node stores a different object than Rails' does.

Rails (`activerecord/lib/arel/select_manager.rb:102-113`):

```ruby
def join(relation, klass = Nodes::InnerJoin)
  return self unless relation

  case relation
  when String, Nodes::SqlLiteral
    raise EmptyJoinError if relation.empty?
    klass = Nodes::StringJoin
  end

  @ctx.source.right << create_join(relation, nil, klass)
  self
end
```

trails today (`packages/arel/src/select-manager.ts:213-232`) adds, after the
emptiness guard:

```ts
if (typeof relation === "string") relation = new SqlLiteral(relation);
```

because `createJoin` is typed on `Node`. Ruby has no such constraint: a
`StringJoin` whose `left` is a bare String is what Rails builds, and
`visit_Arel_Nodes_StringJoin` renders it via `visit`, which dispatches on
`String`. Whether the wrapped `SqlLiteral` renders identically in every case —
particularly around `retryable` and the `SqlLiteral`-vs-String arms in the
visitors — is exactly what has NOT been established.

Surfaced by the RFC 0096 arel naming burndown (PR #6350) as the residual
`create_join` `naming` row on `select-manager.ts`; the parameter itself was
renamed to Rails' `relation` there, leaving only the conversion.

## Converged shape

Widen `createJoin`'s left parameter to Rails' union (`Node | string`) and pass
`relation` through untouched, so the join node holds what Rails' holds — or
establish that the `SqlLiteral` wrap is observationally identical at every
visitor arm and record that at the call site.

Check `visit_Arel_Nodes_StringJoin` and the `String, Nodes::SqlLiteral` case arms
in `to_sql.rb` on both sides before choosing.

## Acceptance criteria

1. `join` passes `relation` to `createJoin` without an intervening conversion,
   or the conversion is justified at the call site against a specific visitor
   behaviour.
2. The `create_join` `naming` row for `select-manager.ts` in
   `pnpm parity:api:calls:args:report` is gone; report before/after.
3. String-join SQL output is unchanged across all three adapter lanes.
