---
title: "Binary dirty tracking is inverted: equal-bytes assign reports changed, in-place mutation does not"
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

Binary attribute dirty tracking is inverted from Rails in both directions —
verified on `main` (SQLite), independent of #4891:

```text
equal-bytes reassign -> changed: true   (Rails: false)  # false positive
in-place mutation    -> changed: false  (Rails: true)   # false negative
```

Repro (canonical `binaries` / `Binary`):

```ts
const rec = await Binary.create({ name: "d", data: new Uint8Array([0x80, 0xde]) });
const found = await Binary.find(rec.id);
found.data = new Uint8Array([0x80, 0xde]); // distinct array, equal bytes
found.changed; // => true; Rails: "\x80\xde".b == "\x80\xde".b, so NOT changed

const f2 = await Binary.find(rec.id);
f2.data[0] = 0x01; // mutate in place
f2.changed; // => false; Rails' changed_in_place? => true
```

**Root cause (false positive):** `ActiveModel::Type::Value#changed?`
(`vendor/rails/activemodel/lib/active_model/type/value.rb:84-86`) is
`old_value != new_value` — Ruby _value_ equality, so two byte-equal BINARY
Strings are equal. Trails ports it at `packages/activemodel/src/type/value.ts:78-80`
as `oldValue !== newValue` — JS _reference_ equality, which is always true for two
distinct `Uint8Array`s. `BinaryType` does not override `isChanged`, so every
binary attribute assigned an equal-bytes array is reported dirty and re-written on
save.

**Root cause (false negative):** needs investigation. `BinaryType#isChangedInPlace`
(`packages/activemodel/src/type/binary.ts`) does walk bytes correctly and is
byte-correct in isolation (unit-tested in #4891), so the gap is likely in whether
`Attribute#changedInPlace` / `changedInPlaceFromDatabase`
(`packages/activemodel/src/attribute.ts:88,106`) reach it for this type — compare
against Rails' `changed_in_place?` (binary.rb:35-38) and
`Attribute#changed_in_place?`.

Note `Type::Value#changed?`'s reference-equality port is **not** binary-specific —
any type whose values are JS objects rather than primitives inherits the same false
positive. Scope the fix deliberately: a blanket change to `value.ts:78` would touch
every type. Surfaced in review of #4891 (BinaryType#serialize returns the Data
wrapper), which deliberately left it alone as out of scope.

## Acceptance criteria

- [ ] Assigning a byte-equal `Uint8Array` to a binary attribute reports
      `changed === false`, matching Rails' value equality.
- [ ] Mutating a binary attribute's bytes in place reports `changed === true`,
      matching Rails' `changed_in_place?` (binary.rb:35-38).
- [ ] Decide and document whether the fix belongs in `BinaryType` (an `isChanged`
      override comparing bytes) or in `Value#isChanged` (`value.ts:78`) — if the
      latter, enumerate which other types change behavior and cover them.
- [ ] No spurious UPDATE: saving a record after an equal-bytes assignment issues no
      write for that column.
- [ ] parity:api / parity:test delta non-negative.
