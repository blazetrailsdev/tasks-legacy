---
title: "checkConstraintForBang renders options with JSON.stringify, not Ruby Hash to_s"
status: draft
updated: 2026-08-02
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`SchemaStatements#checkConstraintForBang`
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
around the `checkConstraintFor` pair) builds its raise message with
`JSON.stringify(options)`:

```ts
`Table '${tableName}' has no check constraint for ${options.expression ?? JSON.stringify(options)}`;
```

Rails builds it by interpolating the options Hash directly
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1803-1806`):

```ruby
raise(ArgumentError, "Table '#{table_name}' has no check constraint for #{expression || options}")
```

Ruby interpolation calls `to_s` on the Hash, which under Ruby 3.4 renders
`{name: "quantity_check"}`. `JSON.stringify` renders `{"name":"quantity_check"}`
— different quoting and no space after the colon. Rails' own
`check_constraint_test.rb:283` asserts the message verbatim
(`assert_equal "Table 'trades' has no check constraint for #{{ name: "quantity_check" }}"`),
so the divergence is directly observable and blocks porting that assertion
faithfully. PR #5913 asserted the trails-shaped message instead, which is why
this is worth tracking rather than silently living with.

The same `JSON.stringify`-for-a-Ruby-Hash pattern likely appears in sibling
raise sites (`foreignKeyForBang`, `checkConstraintName`, …) — grep before
scoping.

## Acceptance criteria

- Add a shared helper that renders an options object the way Ruby 3.4 renders a
  symbol-keyed Hash (`{name: "x", validate: true}`), and route
  `checkConstraintForBang` through it.
- Sweep the sibling `JSON.stringify(options)` raise sites surfaced by grep and
  route them through the same helper; if that pushes past the LOC ceiling,
  ship the check-constraint one and file the rest.
- Update `packages/activerecord/src/migration/check-constraint.test.ts` to
  assert Rails' verbatim message rather than the trails-shaped one.
- No new assertion-ratchet debt.
