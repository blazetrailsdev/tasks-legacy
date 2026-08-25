---
title: "bulkChangeTable's *_for_alter dispatch is a dead double-ternary and a hand-rolled partition"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 7024
claim: "2026-08-25T01:44:12Z"
assignee: "retire-attribute-set-narrow-to"
blocked-by: null
closed-reason: null
---

# `bulkChangeTable`'s `*_for_alter` dispatch is a dead double-ternary and an invented `partition`

## Context

Surfaced while converging `bulkChangeTable`'s loop header to Rails' pair
destructure in PR #7020 (RFC 0051), which touched only the `for` line.

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1555-1578`

```ruby
operations.each do |command, args|
  table, arguments = args.shift, args
  method = :"#{command}_for_alter"

  if respond_to?(method, true)
    sqls, procs = Array(send(method, table, *arguments)).partition { |v| v.is_a?(String) }
    sql_fragments.concat(sqls)
    non_combinable_operations.concat(procs)
  else
    ...
```

trails (`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
`bulkChangeTable`) spells the `respond_to?` as a ternary whose two arms are
byte-identical, so the whole expression is `typeof this[m] === "function" ? this : null`
written twice:

```ts
const forAlterTarget =
  typeof (this as any)[`${command}ForAlter`] === "function"
    ? (this as any)
    : typeof (this as any)[`${command}ForAlter`] === "function"
      ? (this as any)
      : null;
const forAlterMethod = forAlterTarget ? forAlterTarget[`${command}ForAlter`] : null;
if (typeof forAlterMethod === "function") {
```

Three `typeof` tests and a nullable target variable stand in for one
`respond_to?(method, true)`. The `partition` is also hand-rolled into a
`for` loop with a `typeof r === "string"` / `typeof r === "function"` pair
rather than the two-way split Rails writes.

## Converged shape

One `method` local (`schema_statements.rb:1561`), one membership test standing
in for `respond_to?(method, true)`, and `Array(...)` split the way Rails'
`partition` splits it — `sqls` concatenated onto `sqlFragments`, `procs` onto
`nonCombinable`. No `forAlterTarget`, no repeated ternary arm.

## Acceptance criteria

- [ ] The duplicated ternary arm is gone; the dispatch tests membership once.
- [ ] Locals are Rails' (`method`, `sqls`, `procs`, `table`, `arguments`),
      matching `schema_statements.rb:1559-1566`.
- [ ] Emitted SQL is unchanged — the MySQL bulk-alter path
      (`abstract-mysql-adapter.trails.test.ts`, `migration/change-table.test.ts`)
      is the exerciser.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
