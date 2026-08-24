---
title: 'OID::Array#type invents an "array" literal where Rails delegates nil'
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Rails' `OID::Array` delegates `type` straight to its subtype
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/array.rb:13`,
`delegate :type, :user_input_in_time_zone, :limit, :precision, :scale, to: :subtype`),
and `ActiveModel::Type::Value#type` returns `nil` unless a subclass overrides it
(`vendor/rails/activemodel/lib/active_model/type/value.rb:33-34`). So an
`OID::Array` wrapping a bare `Type::Value` reports `nil` for `type`.

trails' port returns the string `"array"` in that case —
`packages/activerecord/src/connection-adapters/postgresql/oid/array.ts:84-89`:

```ts
override type(): string {
  const subtypeType = this.subtype.type;
  if (typeof subtypeType === "function") return subtypeType.call(this.subtype) ?? "array";
  if (typeof subtypeType === "string") return subtypeType;
  return "array";
}
```

The `?? "array"` arm was added in #5203 when `ArraySubtype.type` was widened to
`() => string | undefined` (so a plain `Type` satisfies the interface and the
`addModifier` registrations need no cast). It is unreached by the modifiers
registered there — `StringType` / `IntegerType` both override `type()` with a
real string — but it is a standing divergence from Rails' nil, as is the
pre-existing trailing `return "array"`.

Surfaced while reviewing #5203 (ar-pg-array-range-type-modifiers-never-registered).

## Acceptance criteria

- `Array#type()` returns `undefined` (Ruby's `nil`) when the subtype's `type`
  is nil, rather than the invented `"array"` literal — both the `??` arm and
  the trailing default.
- Return type widened to `string | undefined`; callers that assumed a string
  are audited and updated.
- A test covers an `OID::Array` over a bare `ValueType`, asserting the nil
  delegation. Existing PG array tests stay green.
- If any caller genuinely needs a non-nil fallback, the fallback moves to that
  call site with a comment, not into the delegating accessor.
