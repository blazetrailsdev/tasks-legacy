---
title: "build_subquery's except/arel duck-type guards are trails inventions"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while converging `build_subquery`'s callee (PR #6867, RFC 0106).

`build_subquery` (`activerecord/lib/active_record/relation/query_methods.rb:1605-1611`)
is three statements — `except(:optimizer_hints).arel.as(subquery_alias)`, a
`SelectManager.new(...).project(...)`, and the `optimizer_hints` tap. It has no
receiver-capability check: a relation that cannot answer `arel` raises
`NoMethodError` from the send itself.

The port
(`packages/activerecord/src/relation/query-methods.ts`, `buildSubquery`) opens
with two invented guards Rails has no counterpart for:

```ts
const relation =
  typeof (this as any).except === "function" ? (this as any).except("optimizerHints") : this;
if (typeof relation.arel !== "function") {
  throw new ActiveRecordError("Cannot build subquery: relation does not support arel()");
}
```

Both are duck-type probes standing in for a send: the `except` ternary silently
skips the `except(:optimizer_hints)` Rails always performs when the receiver has
no `except`, and the `arel` guard converts Rails' `NoMethodError` into an
`ActiveRecordError` with an invented message string. The `except` arm is the
more consequential one — it changes the AST the subquery carries rather than
just the error class.

## Converged shape

```ts
const subquery = this.except("optimizerHints").arel().as(subqueryAlias);
```

Both calls unconditional, no capability probe, no invented error class or
message — the missing method raises on its own, as it does in Ruby. Anything the
guards were actually protecting (a host object that is not a full Relation)
should be fixed at that caller, not absorbed here.

## Acceptance criteria

- [ ] `buildSubquery` calls `except("optimizerHints")` and `arel()`
      unconditionally, with no `typeof ... === "function"` probe.
- [ ] The `ActiveRecordError("Cannot build subquery: ...")` raise is gone; no
      invented error class or message remains in the body.
- [ ] `pnpm parity:api:calls` / `:args` green, activerecord relation suites green.
