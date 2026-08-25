---
title: "StringType/DateTimeType serialize must not re-cast an already-cast value"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced by PR #4954 while porting `Relation#_substitute_values`.

trails' `StringType.serialize` (`packages/activemodel/src/type/string.ts:35-37`)
is `return this.cast(value);`. Rails' `ImmutableString#serialize`
(`vendor/rails/activemodel/lib/active_model/type/immutable_string.rb:48-54`)
does NOT cast — it is a case statement:

```ruby
def serialize(value)
  case value
  when ::Numeric, ::Symbol, ActiveSupport::Duration then value.to_s
  when true then @true
  when false then @false
  else super
  end
end

def serialize_cast_value(value) # :nodoc:
  value
end
```

`serialize_cast_value` is identity (`immutable_string.rb:57-59`). The same
self-casting shape appears in `DateTimeType.serialize`
(`packages/activemodel/src/type/date-time.ts:201`), which calls `this.cast(value)`
before `serializeCastValue`.

Consequence: any already-cast value serialized for the database is cast a second
time inside `serialize`. It is invisible today because the affected casts happen
to be idempotent, but it is a real fidelity gap and it actively masks
bind-level double-casting — a test that counts `type.cast` calls can never
observe 1 with a real type, which cost two failed test iterations on #4954
before stack-tracing the source.

## Acceptance criteria

- [ ] `StringType.serialize` mirrors `immutable_string.rb:48-54` — a branch on
      Numeric/Symbol/Duration/true/false with `super` fallback, no `this.cast`.
- [ ] `StringType.serializeCastValue` is identity, per `immutable_string.rb:57-59`.
- [ ] `DateTimeType.serialize` no longer re-casts an already-cast value.
- [ ] A cast-count test using a real type can observe exactly one cast for an
      already-cast value.
- [ ] parity:api / parity:test delta non-negative.
