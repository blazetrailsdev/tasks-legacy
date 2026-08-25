---
title: "Register the MessagePack Duration and Range extension types"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::MessagePack::Extensions.install` registers 18 extension types
(`vendor/rails/activesupport/lib/active_support/message_pack/extensions.rb:19-101`).
trails now registers 0 Symbol, 1 Integer, 5 DateTime, 6 Date, 7 Time,
8 TimeWithZone, 9 TimeZone, 12 Set, 17 HashWithIndifferentAccess, 127 Object.

Still unregistered, with the ones that DO have a faithful trails analogue first:

- **10 `ActiveSupport::Duration`** — `packages/activesupport/src/duration.ts` is
  ported; `write_duration`/`read_duration` (extensions.rb:194-203) pack
  `duration.value` plus `_parts.values_at(*PARTS)`.
- **11 `Range`** — `write_range`/`read_range` (extensions.rb:205-213) pack
  begin/end/exclude_end.
- **16 `Regexp`** — packer `:to_s`, unpacker `:new`; JS `RegExp` differs in
  syntax and flags, so fidelity needs a decision.
- 2 BigDecimal, 3 Rational, 4 Complex, 13 URI::Generic, 14 IPAddr, 15 Pathname —
  no JS analogue today; descoped deliberately, listed in the comment above
  `enshrines type IDs` in `message-pack/serializer.test.ts`.

Duration and Range are the two with a clean path and no new dependency.

## Acceptance criteria

- Register types 10 and 11 with Rails' ids and payload layout, packing through
  `Extensions` methods named after Rails' (`writeDuration`/`readDuration`,
  `writeRange`/`readRange`).
- Rails' `roundtrips ActiveSupport::Duration` and `roundtrips Range` test names,
  verbatim, from `shared_serializer_tests.rb`; the new ids added to
  `enshrines type IDs` and removed from the descoped comment.
- `pnpm parity:api --package activesupport` non-negative.
