---
title: "detect_enum_conflict! bypasses method_defined_within? on both arms"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: ["converge-method-defined-within-onto-owner-comparison"]
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6844 (`wave-4c-ar-core-residue-attributes-remainder-part-6`),
which retired the `enum.json` call-set shard. Two of its seven rows —
`detect_enum_conflict! -> method_defined_within?` — could NOT be converged and
ship as a single `@missingRailsCall` tag on `detectEnumConflictBang`
(`packages/activerecord/src/enum.ts`). This story converges them.

Rails (`activerecord/lib/active_record/enum.rb:374-386`):

```ruby
def detect_enum_conflict!(enum_name, method_name, klass_method = false)
  if klass_method && dangerous_class_method?(method_name)
    raise_conflict_error(enum_name, method_name, type: "class")
  elsif klass_method && method_defined_within?(method_name, Relation)
    raise_conflict_error(enum_name, method_name, type: "class", source: Relation.name)
  elsif klass_method && method_name.to_sym == :id
    raise_conflict_error(enum_name, method_name)
  elsif !klass_method && dangerous_attribute_method?(method_name)
    raise_conflict_error(enum_name, method_name)
  elsif !klass_method && method_defined_within?(method_name, _enum_methods_module, Module)
    raise_conflict_error(enum_name, method_name, source: "another enum")
  end
end
```

Two arms, two distinct blockers.

**enum.rb:377 (`Relation`).** trails calls `isRelationInstanceMethod`
(`packages/activerecord/src/scoping/named.ts:71-78`), a trails-only helper that
walks `Relation.prototype` up to but not including `Object.prototype`. Routing
it through `isMethodDefinedWithin` was tried in #6844 and reverted: measured
against the built `dist/`, `isMethodDefinedWithin("toString", Relation)` returns
`true` where Ruby's owner comparison returns `false`, so an enum value named
`toString` would be rejected where Rails accepts it. The cause is
`instanceMethodOwner` (`packages/activerecord/src/attribute-methods.ts:457-465`),
which walks the CONSTRUCTOR chain — `Relation` -> `Function.prototype` -> stops,
because `Object.prototype` is `typeof "object"`, not `"function"`. It therefore
never inspects `Object.prototype`'s own methods, and the inner owner comparison
degrades to the `return true` arm. Passing an explicit third argument does not
help; verified both two- and three-argument forms.

This is the same defect [[converge-method-defined-within-onto-owner-comparison]]
tracks, hence the dep. That story's quoted `isMethodDefinedWithin` body is stale
(the port has since gained `instanceMethodOwner`), but its **Converged shape** —
resolve the owning prototype via `Object.getOwnPropertyDescriptor` walked up the
PROTOTYPE chain, not the constructor chain — is exactly what unblocks this one.

**enum.rb:383 (`_enum_methods_module`).** trails tracks generated value-method
names in a per-class `_enumMethodsModuleNames` Set (`enum.ts`, in `_enum`)
instead. Rails' third argument is `Module`, the superklass to compare owners
against; trails' enum methods module is an interposed prototype carrier
(`getOrCreateModuleCarrier`), not a class, so it has no `.prototype` for
`isMethodDefinedWithin` to read.

## Converged shape

1. Land [[converge-method-defined-within-onto-owner-comparison]] first, so
   `isMethodDefinedWithin` compares owning prototypes.
2. enum.rb:377 becomes `isMethodDefinedWithin.call(this, methodName, Relation)`,
   and `isRelationInstanceMethod` is deleted (its only other caller is
   `scoping/named.ts`'s `scope`, which ports the same Rails predicate and should
   move with it).
3. enum.rb:383 becomes an `isMethodDefinedWithin` call over the carrier — either
   by teaching it to accept a bare prototype object alongside a constructor, or
   by giving the enum methods module a real class so it has a `.prototype`.
   `_enumMethodsModuleNames` is deleted once the carrier itself answers the
   question, which is what Rails reads.
4. The `@missingRailsCall method_defined_within?` tag on `detectEnumConflictBang`
   is deleted.

## Acceptance criteria

- [ ] Both `method_defined_within?` arms call the port of it, in Rails' `elsif`
      order, with Rails' argument lists.
- [ ] `isRelationInstanceMethod` and `_enumMethodsModuleNames` are gone.
- [ ] An enum value named `toString` is still accepted (the regression that
      blocked the naive convergence in #6844) — covered by a test.
- [ ] The `@missingRailsCall` tag is removed; `pnpm parity:api:calls` green
      with no new baseline row.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
