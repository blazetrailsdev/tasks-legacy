---
title: "Time#isdst, tzdataAbbreviation and TimeZone#isDst all guess DST from January/July offsets; tzdata carries the bit"
status: done
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 420
priority: null
pr: 6949
claim: "2026-08-23T21:02:49Z"
assignee: "drop-djar-relation-prototype-toarray-punch"
blocked-by: null
closed-reason: null
---

## Context

Three places in trails answer "is this instant in daylight saving?" by comparing
the instant's offset to the smaller of the zone's January and July offsets:

- `packages/date/src/time.ts` — `Time#isdst` / `Time#dst?` (MRI reads a real
  flag: `ruby/time.c` `time_isdst` answers `tm.tm_isdst`, from the zone's tzdata
  transition record).
- `packages/date/src/time.ts` — `tzdataAbbreviation`, which picks
  `abbreviations[1]` on the same reading.
- `packages/activesupport/src/values/time-zone.ts` — `Timezone#isDst`, the port
  of `TZInfo::Timezone#dst?`, which reads `TimezonePeriod#dst?` upstream —
  itself the tzdata `isdst` bit (tzinfo 2.x
  `lib/tzinfo/timezone_offset.rb`, `dst?`).

PR #6937 measured the heuristic against MRI: 3,714 samples (418
`Intl.supportedValuesOf("timeZone")` zones x 9 dates, `TZ=<zone> ruby -e 'p
Time.local(...).isdst'`) gives **121 mismatches**. The alternatives that need no
data table are all worse or no better:

| reading                                                                             | mismatches                        |
| ----------------------------------------------------------------------------------- | --------------------------------- |
| January/July sampling (today)                                                       | 121                               |
| offset vs surrounding `getTimeZoneTransition` segments, 3/6/7/9/12/18-month windows | 778 / 719 / 621 / 748 / 709 / 680 |
| min over 24 monthly samples of the year                                             | 113                               |
| CLDR `timeZoneName` (`long` / `shortGeneric`)                                       | does not carry the flag at all    |

The transition reading calls every PERMANENT standard-offset shift DST
(`America/Cancun` 2015, `Europe/Istanbul` 2016, `Asia/Amman` 2022,
`Africa/Juba` 2020 — all `false` in MRI). CLDR fails in both directions:
`America/New_York` 1943 formats as `GMT-4` with no daylight wording (MRI
`true`), `Europe/Dublin` in summer formats as `Irish Standard Time` (MRI
`true`). No ECMAScript or `Temporal` API exposes tzdata's `isdst` bit.

`time-isdst-samples-two-months-instead-of-reading-the-dst-flag` is blocked on
exactly this; this story is the data-table half it was re-scoped to.

## Converged shape

Vendor a generated tzdata `isdst` table into `@blazetrails/date` — per zone, the
transition instants and the `isdst` bit of the offset each one starts — and read
it from all three sites so `Time#isdst`, `tzdataAbbreviation` and
`Timezone#isDst` answer the flag MRI and TZInfo answer rather than guessing it
from offsets. Generation belongs in `scripts/` with a regeneration guard, the
shape `canonical-zone-identifiers-table-has-no-regeneration-guard` established
for the sibling table.

Size the table before committing to the shape: 418 zones' full transition
history is large, and a cut (post-1970, or bits only at the transitions the
runtime can already locate via `getTimeZoneTransition`) may carry the flag at a
fraction of the bytes.

## Acceptance criteria

- [ ] A vendored tzdata table carries the `isdst` bit, with a regeneration guard.
- [ ] `Time#isdst`, `tzdataAbbreviation` and `TimeZone`'s `isDst` all read it —
      no January/July sampling left in any of the three.
- [ ] The 3,714-sample MRI comparison in this body drops to 0 mismatches, with
      the year-round-DST (`America/New_York` 1943, 1974) and permanent-shift
      (`America/Cancun`, `Europe/Istanbul`, `Asia/Amman`) cases as named tests.
- [ ] `time-isdst-samples-two-months-instead-of-reading-the-dst-flag` unblocks.
