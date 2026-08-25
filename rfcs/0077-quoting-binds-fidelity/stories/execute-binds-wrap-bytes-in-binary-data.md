---
title: "Wrap raw execute() byte binds so type_cast needs no byte-view arm"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 5
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #6291 (`wrap-binary-and-array-binds-at-source`). That story tried to
delete the bare byte arms from `type_cast` on the premise that
`BinaryType#serialize` wraps every value in `Type::Binary::Data`
(`activemodel/lib/active_model/type/binary.rb:30-33`). CI disproved it on all
three lanes: `TypeError: can't cast Buffer`.

Rails reaches `type_cast` with bytes **two** ways:

- `Type::Binary::Data` — `activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:96`
  (`when Symbol, ActiveSupport::Multibyte::Chars, Type::Binary::Data then value.to_s`).
- a BINARY/ASCII-8BIT Ruby `String` handed straight to `execute(sql, binds)` —
  caught by `when nil, Numeric, String then value` (`abstract/quoting.rb:102`).
  This is what `test_type_cast_binary_encoding_without_logger`
  (`activerecord/test/cases/adapters/sqlite3/quoting_test.rb:32`) and
  `test_type_cast_should_not_mutate_encoding`
  (`activerecord/test/cases/adapters/sqlite3/sqlite3_adapter_test.rb:482`) bind.

A JS string is UTF-16 and cannot carry arbitrary bytes, so #6291 landed
`if (ArrayBuffer.isView(value)) return value;` at the rb:102 position in
`packages/activerecord/src/connection-adapters/abstract/quoting.ts` (one arm,
inherited by sqlite3/mysql via `else super` — down from three copies).

The arm is a documented boundary, not a converged one. It exists because trails'
raw-`execute` callers bind bare byte views where Ruby binds a byte String.

## Converged shape

Wrap at the boundary instead of branching in the cast chain: have the
raw-`execute(sql, binds)` entry points (and the trails test callers in
`packages/activerecord/src/adapters/sqlite3/sqlite3-adapter.test.ts:298-311`,
`adapters/sqlite3/quoting.test.ts:63`) hand a `BinaryData` for byte payloads, so
rb:96 is the only byte arm and rb:102 stays string-only as in Rails.

If the boundary genuinely cannot wrap (a driver contract that must see a raw
Buffer), `pnpm tasks block` with the specific caller — do not re-justify the arm.

## Acceptance criteria

- [ ] Raw byte binds reach `type_cast` wrapped, or the blocking caller is named.
- [ ] `ArrayBuffer.isView` arm at the rb:102 position in `abstract/quoting.ts` is
      deleted; the chain is arm-for-arm with `abstract/quoting.rb:94-105`.
- [ ] `binary.trails.test.ts` "casts both byte forms Rails reaches type_cast with"
      is updated to the converged reality; sqlite3 binary tests pass on all three lanes.
