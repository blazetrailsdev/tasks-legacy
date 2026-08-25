---
title: "MessagePack DateTime/Time offset slots are written as zero and dropped on read"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

PR #5634 mapped Rails' MessagePack temporal extension types onto Temporal:
`DateTime` -> `Temporal.PlainDateTime` (type 5) and `Time` -> `Temporal.Instant`
(type 7). Neither analogue carries a UTC offset, but Rails' payload format has an
offset slot in both
(`vendor/rails/activesupport/lib/active_support/message_pack/extensions.rb:138-167`:
`write_datetime` writes `datetime.offset` as a rational, `write_time` writes
`time.utc_offset`).

The port keeps the wire format byte-compatible by writing zero and, on read,
consuming the slot and discarding it. Consequences:

- A `DateTime` round-tripped through a Ruby producer loses its offset: the
  wall-clock components survive, the zone label does not.
- `read_time` on a payload written by MRI silently drops a non-UTC `utc_offset`.
  The instant is still correct (it is absolute), so this is a labelling loss, not
  a value loss.

Whether this is worth closing depends on whether trails wants a
`Temporal.ZonedDateTime`-backed `DateTime` analogue distinct from
`TimeWithZone` (type 8), which already round-trips its zone.

## Acceptance criteria

- Decide and record whether `DateTime` gets an offset-carrying analogue or the
  zero-offset write stays the documented deviation.
- If it gets one, `writeDatetime`/`readDatetime` round-trip a non-zero offset and
  `writeTime`/`readTime` preserve the `utc_offset` label.
- `pnpm parity:api --package activesupport` non-negative.
