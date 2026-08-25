---
title: "Duck-typed Attribute/BinaryData checks work around a duplicate activemodel copy"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
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

Two places in the quoting surface detect an ActiveModel value by **duck
typing** rather than `instanceof`, both for the same stated reason: the dep
tree can resolve two copies of `@blazetrails/activemodel`, under which
`instanceof` silently misses and the wrapper reaches the driver as a plain
object (node-postgres then JSON-stringifies it).

- `connection-adapters/abstract/quoting.ts`, `typeCastedBinds` — detects
  `Attribute` as `value instanceof ModelAttribute || (typeof value === "object"
&& "valueForDatabase" in value)`. Mirrors `type_casted_binds`
  (`activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:224`),
  where Ruby's `Attribute` case needs no such widening.
- `connection-adapters/postgresql/quoting.ts`, `isBinaryData` — same shape for
  `Type::Binary::Data` at the `type_cast` binary arm
  (`activerecord/lib/active_record/connection_adapters/postgresql/quoting.rb:207`).

Rails has neither widening: a Ruby constant is process-unique, so
`when ActiveModel::Attribute` / `when Type::Binary::Data` is exact. The duck
typing is a workaround for a **packaging** problem, not a language shortcoming
— nothing about TS prevents `instanceof` here once the dep tree resolves one
copy of the package.

The `isBinaryData` predicate was introduced by #6294 when `_bindForPg` folded
into `type_cast`; it consolidated two inlined duck-typed arms into one, but did
not remove the underlying need.

## Converged shape

Fix the resolution so `@blazetrails/activemodel` is a single instance in the
dep tree (pnpm `dedupe` / a workspace `resolutions` pin — confirm what actually
produces the duplicate first; it may only occur under a consumer's install, not
in-repo). Then both checks become the plain `instanceof` Rails' `case/when`
is, and `isBinaryData` disappears as a helper Rails has no counterpart for.

If a duplicate copy turns out to be genuinely unavoidable for consumers, that
is a packaging decision to record explicitly — but the burden is to show the
duplicate is real and unfixable, not to keep the widening because it is
already there.

## Acceptance criteria

- [ ] The duplicate `@blazetrails/activemodel` resolution is confirmed or
      disproved, with the reproducing install recorded.
- [ ] `typeCastedBinds` detects `Attribute` with `instanceof` alone (rb:224).
- [ ] PG `type_cast`'s binary arm uses `instanceof BinaryData` alone (rb:207);
      the `isBinaryData` helper is deleted.
- [ ] PG binary round-trip (bytes 128-255) and the bind paths stay green on all
      three adapters.
- [ ] parity:api / parity:test delta non-negative.
