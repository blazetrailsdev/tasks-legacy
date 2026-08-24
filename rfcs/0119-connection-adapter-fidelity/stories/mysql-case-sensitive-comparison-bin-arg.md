---
title: "case_sensitive_comparison passes an invented quotedNode() to Arel::Nodes::Bin"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
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

`AbstractMysqlAdapter#case_sensitive_comparison` wraps its argument in an
invented conversion before handing it to `Arel::Nodes::Bin`.

Rails (`activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:600-607`):

```ruby
def case_sensitive_comparison(attribute, value) # :nodoc:
  column = column_for_attribute(attribute)

  if column.collation && !column.case_sensitive?
    attribute.eq(Arel::Nodes::Bin.new(value))
  else
    super
  end
end
```

trails (`packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:1043`):

```ts
return attribute.eq(new Nodes.Bin(attribute.quotedNode(value)));
```

`attribute.quotedNode(value)` has no counterpart in the Ruby body — Rails passes
the bare `value` to `Bin.new` and lets the visitor quote it. This surfaces as a
`naming` row in `output/call-arg-mismatches.json`
(`new | ruby=ref:value | ts=ref:quotedNode`), but it is an argument-SHAPE defect,
not a rename: it was left in place while landing
`naming-burndown-activerecord-rest` (#6354) per that story's AC4.

## Acceptance criteria

1. `case_sensitive_comparison` passes `value` to `Nodes.Bin` exactly as
   `abstract_mysql_adapter.rb:604` does; the `quotedNode` conversion is removed,
   or — if the emitted SQL genuinely changes — the quoting is moved to wherever
   Rails performs it (the Arel visitor), not the call site.
2. The MySQL uniqueness-validation tests that exercise a case-insensitive
   collation still pass on the MySQL/MariaDB lanes.
3. The row disappears from `pnpm parity:api:calls:args:report`; no baseline row
   is added.
