---
title: "Arel visitors stringify a non-Node expr instead of visiting it"
status: in-progress
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7054
claim: "2026-08-25T16:56:36Z"
assignee: "arel-star-is-a-shared-const-not-a-per-call-method"
blocked-by: null
closed-reason: null
---

## Context

While converging `visitArelNodesBin` in PR #6450 (RFC 0099
`call-args-ar-mysql-case-sensitive-comparison`), the root cause turned out to be
a non-Rails fallback in the visitor:

```ts
if (o.expr instanceof Node) {
  this.visit(o.expr, collector);
} else if (o.expr !== null) {
  collector.append(String(o.expr));
}
```

Rails is an unconditional `visit o.expr, collector`
(`activerecord/lib/arel/visitors/mysql.rb:7-11`, and the same shape in
`to_sql.rb` for every unary visit). The `instanceof Node` guard silently
stringifies anything the visitor registry _does_ handle but that is not an Arel
`Node` — notably `ActiveModel::Attribute` binds, which render as
`[object Object]` instead of a bind placeholder. That is a wrong-SQL bug, not a
cosmetic divergence, and it hides argument-shape defects at call sites by making
an invented `quotedNode()` wrap look load-bearing.

`visitArelNodesBin` in `packages/arel/src/visitors/mysql.ts` was converged in PR #6450. The same fallback remains at:

- `packages/arel/src/visitors/mysql.ts:33-40` — `visitArelNodesUnqualifiedColumn`
  (Rails `arel/visitors/mysql.rb`: `visit o.expr, collector`)
- `packages/arel/src/visitors/postgresql.ts:55`
- `packages/arel/src/visitors/to-sql.ts:603, 761, 771, 855, 884, 1906`

## Acceptance criteria

1. Each listed site is converged to an unconditional `this.visit(o.expr, collector)`,
   verified against the corresponding `visit o.expr` line in the vendored
   `arel/visitors/{to_sql,mysql,postgresql}.rb` — cite `file:line` per site.
2. Any site where Rails genuinely does something else (a bare-name special case,
   e.g. base `ToSql#visit_Arel_Nodes_UnqualifiedColumn`) is left as Rails has it,
   with the cite; only the invented `String(o.expr)` arms go.
3. A regression test per converged site that renders an `ActiveModel::Attribute`
   bind through the node and asserts a bind placeholder rather than
   `[object Object]`. Must fail on baseline.
4. `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; all three
   adapter lanes green.
