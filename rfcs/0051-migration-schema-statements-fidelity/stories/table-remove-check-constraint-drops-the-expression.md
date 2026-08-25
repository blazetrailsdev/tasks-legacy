---
title: "table-remove-check-constraint-drops-the-expression"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Table#remove_check_constraint` in Rails is
`def remove_check_constraint(*args, **options); @base.remove_check_constraint(name, *args, **options); end`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:938-940`),
so `t.remove_check_constraint("price > 0", name: "price_check")` forwards
**three** arguments: the table name, the expression, and the options.

trails' `Table#removeCheckConstraint`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts`,
~:1826) instead picks one or the other:

```ts
if (typeof expressionOrOptions === "string") {
  return this._schema.removeCheckConstraint(
    this.name,
    options?.name ? options : expressionOrOptions,
  );
}
```

An expression **and** a `name:` collapses to just the options hash — the
expression is dropped. Under `CommandRecorder`, which stores arguments
verbatim, the recorded command has the wrong arity and the wrong content.

Surfaced while converging the empty-`**options` splat across the `Table`
forwarders (`table-forwarders-drop-empty-options-splat`, PR #7019). That PR
fixed only the empty-hash arity; this two-armed collapse is a separate
divergence and was left alone rather than propagated.

## Acceptance criteria

- `Table#removeCheckConstraint` forwards the expression and the options the
  way Ruby's `*args, **options` does — expression positional, options
  trailing and omitted when empty.
- A `command-recorder.trails.test.ts` case pins the recorded tuple for
  `t.removeCheckConstraint("price > 0", { name: "price_check" })`.
- Check `Table#checkConstraint` for the same shape while you are there.
- `parity:api:calls` / `parity:api:calls:args` non-negative; green on
  sqlite3, PostgreSQL and MySQL.
