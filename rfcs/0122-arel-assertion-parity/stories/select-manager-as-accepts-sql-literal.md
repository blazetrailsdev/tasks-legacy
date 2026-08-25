---
title: "SelectManager#as rejects a SqlLiteral where Rails' SqlLiteral.new accepts one"
status: draft
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
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

`SelectManager#as` (`packages/arel/src/select-manager.ts:119-124`) types its
parameter `other: string` and passes it straight to
`new SqlLiteral(other, { retryable: true })`. Rails'
`def as(other) = create_table_alias grouping(ast), Nodes::SqlLiteral.new(other)`
(`vendor/rails/activerecord/lib/arel/select_manager.rb`) accepts anything
`SqlLiteral.new` accepts — and `select_manager_test.rb:47` calls it as
`manager.as(Arel.sql("foo"))`, i.e. with a `SqlLiteral`.

In trails that call wraps a `SqlLiteral` object inside another `SqlLiteral`,
whose `value` is then an object; `String(as.right)` throws
`Cannot convert object to primitive value`. Surfaced while converging
`select_manager_test.rb`'s assertions (RFC 0122); the test had to keep the
string arm to stay green.

## Acceptance criteria

- `SelectManager#as` accepts a `SqlLiteral` as well as a string, as
  `SqlLiteral.new` does in Ruby (a `SqlLiteral` passed in is not double-wrapped).
- `packages/arel/src/select-manager.test.ts`'s `makes an AS node by grouping the
AST` and `can make a subselect` pass `sql("foo")` the way the Ruby does.
- `pnpm vitest run packages/arel/src/select-manager.test.ts` green.
