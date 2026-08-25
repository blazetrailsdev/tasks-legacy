---
title: "Type::Binary#cast coerces non-String values Rails passes through"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
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

Surfaced while porting `type/binary_test.rb`'s assertions in PR #6519 (RFC 0105).
The assertion port is blocked on this deviation: converging the assertions alone
reds the test.

Rails `ActiveModel::Type::Binary#cast`
(`vendor/rails/activemodel/lib/active_model/type/binary.rb:20-27`):

```ruby
def cast(value)
  if value.is_a?(Data)
    value.to_s
  else
    value = super
    value = value.b if ::String === value && value.encoding != Encoding::BINARY
    value
  end
end
```

Only a `String` is coerced (to BINARY encoding); every other value falls through
`super` untouched. `test_type_cast_binary` (`test/cases/type/binary_test.rb:8-19`)
pins that directly: `assert_equal 1, type.cast(1)`.

trails `packages/activemodel/src/type/binary.ts:17-22` coerces unconditionally:

```ts
cast(value: unknown): Uint8Array | null {
  if (value === null || value === undefined) return null;
  if (value instanceof Data) return value.bytes;
  if (value instanceof Uint8Array) return value;
  return textEncoder.encode(String(value));   // <- Rails returns `value` here
}
```

so `cast(1)` answers the 1-byte array for `"1"` where Rails answers `1`.

## Converged shape

`cast` returns non-String, non-`Data`, non-byte values unchanged, mirroring
Rails' `super` arm; only the String arm is encoded. The return type widens from
`Uint8Array | null` accordingly. Check the AR quoting path
(`abstract/quoting.rb:83` dispatches on the `Data` wrapper, not on `cast`'s
output) and `isChangedInPlace`, which currently leans on `cast` always handing
back a `Uint8Array`.

Rails' `assert_equal Encoding::BINARY, type.cast("1").encoding` and
`assert_not_equal "ƒée", type.cast("ƒée")` have no JS counterpart — a
`Uint8Array` carries no encoding tag and JS has no operator overloading. Note
that at the call site rather than asserting it vacuously.

## Acceptance criteria

- `type.cast(1)` answers `1`; the String arm still answers bytes.
- `type/binary_test.rb` reports 0 assertion-count / -kind / -value mismatches in
  `pnpm parity:test -- --assertions --package activemodel`, and the mark is
  lowered by that amount.
- AR binary column round-trips (quoting, dirty tracking) still pass.
