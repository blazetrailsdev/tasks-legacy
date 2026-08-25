---
title: "PolymorphicArrayValue#klass must return nil for anything that is not a model or Relation"
status: draft
updated: 2026-08-12
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

Surfaced while reviewing PR #6427 (`call-args-ar-host-param-core`), tracing
`PolymorphicArrayValue#type_to_ids_mapping`.

Rails
`activerecord/lib/active_record/relation/predicate_builder/polymorphic_array_value.rb:35-39`:

```ruby
def klass(value)
  if value.is_a?(Base)
    value.class
  elsif value.is_a?(Relation)
    value.model
  end
end
```

Only an `ActiveRecord::Base` instance or a `Relation` yields a class; anything
else returns `nil`, so `klass(value)&.polymorphic_name`
(`polymorphic_array_value.rb:28`) is `nil` and the row groups under the
no-type bucket, and `primary_key(value)`
(`polymorphic_array_value.rb:32`) passes `nil` to `join_primary_key`.

trails `packages/activerecord/src/relation/predicate-builder/polymorphic-array-value.ts`
`klass()` instead ends with `return (value as any).constructor ?? null`, so ANY
non-null object — a plain `{}`, a Date, an Arel node — is treated as a model
class. PR #6427 converged the call at the use site to
`klass?.polymorphicName?.()`, which is Rails' shape, but the optional call is
load-bearing only because `klass()` can hand back a constructor that has no
`polymorphicName`. With `klass()` converged the `?.` guard goes away too.

## Converged shape

```ts
private klass(value: unknown): unknown {
  if (value instanceof Base) return value.constructor;
  if (isRelation(value)) return (value as any)._model;
  return null;
}
```

using whatever `Base`/Relation brand test this file can reach without closing an
import cycle (the file currently duck-types Relation as `"_model" in value &&
"toArel" in value`). Then `type_to_ids_mapping` reads
`klass?.polymorphicName()` with no second optional call.

## Acceptance criteria

1. `klass()` returns a class only for a model instance or a Relation, matching
   `polymorphic_array_value.rb:35-39`; every other value yields `null`.
2. The `?.polymorphicName?.()` double-optional at the call site collapses to
   Rails' single `&.`.
3. Existing polymorphic where-clause coverage stays green (the
   `where({ commentable: [...] })` suites).
