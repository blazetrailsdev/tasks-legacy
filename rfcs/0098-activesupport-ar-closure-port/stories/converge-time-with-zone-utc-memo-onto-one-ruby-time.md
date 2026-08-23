---
title: "TimeWithZone carries @utc twice; converge the seat onto one ::Time and the TimeZone readers with it"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 320
priority: null
pr: 6934
claim: "2026-08-23T18:32:16Z"
assignee: "seat-the-per-instance-primary-key-slot"
blocked-by: null
closed-reason: null
---

## Context

Rails' `@utc` is one thing: both the memo and the value `#utc` answers —
`@utc ||= incorporate_utc_offset(@time, -utc_offset)`
(`activesupport/lib/active_support/time_with_zone.rb:63-65`), a `::Time`.
`#period` reads that same ivar: `@period ||= time_zone.period_for_utc(@utc)`
(`:72-74`).

PR #6930 made `#utc` answer a `::Time` but left the internal seat alone, so
trails now carries the value **twice** in
`packages/activesupport/src/time-with-zone.ts`:

- `_utc: UtcConstructed | null` — the `Temporal.Instant`, which is what
  `_zoned`, `_utcPlain`, `period` and `_incorporateUtcOffset` all take;
- `_utcTime?: Time` — the `::Time` `utc()` answers, seated from it;
- `_utcConstructed` — a private getter carrying Rails' `||=` for the first.

Two memos and a private getter where Rails has one ivar. This was measured during
PR #6930 priced this and deferred it because the blast radius is the `TimeZone` reader surface,
not `TimeWithZone` itself.

## Converged shape

`_utc` holds the `::Time`, `utc()` answers it directly, and `_utcTime` /
`_utcConstructed` both disappear.

That requires the readers it feeds to take a `::Time` where they take a
`Temporal.Instant` today, in `packages/activesupport/src/values/time-zone.ts`:

- `periodForUtc(time)` — Rails' `TZInfo::Timezone#period_for_utc`, which takes
  a `::Time`;
- `periodsForLocal(time)` / `periodForLocal(time, dst)`;
- `TimeWithZone#_incorporateUtcOffset` and
  `#_transferTimeValuesToUtcConstructor` (`time_with_zone.rb:583-587`, whose
  Ruby body is literally `Time.utc(year, month, day, hour, min, sec + subsec)`).

`_zoned` / `_utcPlain` / `_epochMs` are trails-only privates with no Rails
counterpart; once the seat is a `::Time`, re-derive them from it (or delete the
ones that fall out) rather than keeping a parallel instant.

Size this before claiming — it may want splitting at the `values/time-zone.ts`
boundary, with the reader signatures moving first and `TimeWithZone` following
from `main`.

## Acceptance criteria

- [ ] `TimeWithZone` carries `@utc` once, as a `::Time`; `_utcTime` and
      `_utcConstructed` are gone.
- [ ] `periodForUtc` / `periodsForLocal` / `periodForLocal` take the `::Time`
      their TZInfo counterparts take.
- [ ] `_transferTimeValuesToUtcConstructor` reads as `Time.utc(year, month, day,
hour, min, sec + subsec)` (time_with_zone.rb:583-587).
- [ ] `parity:api:calls` / `:args` clean; `parity:api:extra` novel count for
      `time-with-zone.ts` does not rise.
