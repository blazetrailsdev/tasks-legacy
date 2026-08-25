---
title: "QueryAttribute#type_cast is a no-op in Rails; trails casts"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by review of PR #4887 (arel-predications-unboundable-duck-types-like-rails).

Rails' `QueryAttribute#type_cast` is deliberately a **no-op**
(`activerecord/lib/active_record/relation/query_attribute.rb:22-24`):

```ruby
def type_cast(value)
  value
end
```

so `QueryAttribute#value` is the raw `value_before_type_cast`. Trails'
(`packages/activerecord/src/relation/query-attribute.ts:50-52`) casts instead:

```ts
typeCast(value: unknown): unknown {
  return this.type.cast(value);
}
```

This means `value` and `valueBeforeTypeCast` are the same object in trails where
Rails distinguishes them, and any Rails code reading `attribute.value` on a
QueryAttribute gets a cast value where Rails gets the raw one.

## Why it matters

It is currently masked by a second divergence that happens to cancel it, which is
a fragile state to leave the port in:

- `Attribute#isSerializable` (`activemodel/src/attribute.ts:96-97`) passes
  `this.value` to `type.isSerializable`. Rails passes the raw value and
  `Integer#serializable?` (integer.rb:74-80) does its own `cast_value = cast(value)`.
  Because trails pre-casts, the two layers agree today only by accident.
- `QueryAttribute#isUnboundable` (#4887) reads `this.value` for Ruby's
  `value <=> 0`, relying on it already being cast.

Fixing `typeCast` alone will therefore change behavior at both sites and must be
done with them, not before them.

## Acceptance criteria

- [ ] `QueryAttribute#typeCast` is a no-op, mirroring query_attribute.rb:22-24.
- [ ] `value` vs `valueBeforeTypeCast` semantics match Rails for a QueryAttribute.
- [ ] `isUnboundable`'s `value <=> 0` still reads the _cast_ value Rails'
      `serializable?` yields (integer.rb:75), not the now-raw `value`.
- [ ] `Attribute#isSerializable` still answers correctly once the value it passes
      is raw (see the companion story on `IntegerType#castValue`).
- [ ] parity:api / parity:test delta non-negative.

## Absorbed: `converge-query-attribute-type-cast-to-rails-no-op`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "converge-query-attribute-type-cast-to-rails-no-op"

### Context

`vendor/rails/activerecord/lib/active_record/relation/query_attribute.rb:22-24`
is:

```ruby
def type_cast(value)
  value
end
```

A `QueryAttribute` never routes its raw value through the type — `value` IS
`value_before_type_cast`, and `serializable?` / `unboundable?` / `infinite?`
all read the raw value.

trails' `packages/activerecord/src/relation/query-attribute.ts#typeCast` casts
instead (`return this.type.cast(value)`), a divergence already noted in the
`isUnboundable()` JSDoc in that file. PR #6528 hit it: once
`ActiveModel::Type::DateTime#cast_value` carries Rails' `else` arm (returning a
non-String receiver untouched instead of coercing it through `String(value)`),
a `StatementCache::Substitute` bind reaches a `normalizes` normalizer and
raises — Rails never gets there, because `type_cast` is a no-op. #6528 shipped
the narrow guard Rails' own constructor justifies (`query_attribute.rb:13-14`,
"we don't need to serialize StatementCache::Substitute"): `typeCast` returns a
`Substitute` unchanged. The general divergence stands.

Converging `typeCast` to Rails' no-op was measured on that branch: it makes
`normalized-attribute.test.ts` pass without the guard, and reds 3 tests in
`packages/activerecord/src/relation/query-attribute.test.ts` that assert the
casting behaviour (e.g. `new QueryAttribute("age", "25", intType).value` is
expected to be `25`, where Rails answers `"25"`). Those expectations need
checking against `vendor/rails/activerecord/test/cases/relation/` first — if
they are trails inventions they go; if they mirror a Rails test, the
`value` readers that depend on the cast are what has to move.

### Acceptance criteria

- [ ] `QueryAttribute#typeCast` is `return value`, matching
      `query_attribute.rb:22-24`, and the `Substitute` guard #6528 added is
      gone with it (it exists only because the cast does).
- [ ] Each red expectation in `relation/query-attribute.test.ts` is resolved
      against the Rails test it mirrors, not by re-adding a cast.
- [ ] `pnpm parity:api:calls` / `parity:api:calls:args` stay green; no new
      baseline rows.
