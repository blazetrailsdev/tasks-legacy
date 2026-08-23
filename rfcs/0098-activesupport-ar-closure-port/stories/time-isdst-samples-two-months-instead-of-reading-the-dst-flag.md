---
title: "Time#isdst guesses the standard offset from January and July; MRI reads tm_isdst"
status: ready
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6930 added `Time#isdst` / `Time#dst?` to `@blazetrails/date`
(`packages/date/src/time.ts`), because Rails' `Time#change` passes the
receiver's `isdst` to `::Time.local` on its `elsif zone` arm
(`activesupport/lib/active_support/core_ext/time/calculations.rb:172-175`).

MRI reads a real flag: `time.c` `time_isdst` answers `tm.tm_isdst`, which comes
from the zone's tzdata transition record. trails approximates it:

```ts
const january = Number(zoned.with({ month: 1, day: 1 }).offsetNanoseconds);
const july = Number(zoned.with({ month: 7, day: 1 }).offsetNanoseconds);
return Number(zoned.offsetNanoseconds) > Math.min(january, july);
```

"Standard offset is the smaller of the January and July offsets" is the idiom
`tzdataAbbreviation` in the same file already uses, so this is consistent with
the file — but it is a heuristic, and it is wrong wherever the zone's two
sample months do not straddle its transitions:

- a zone whose DST window sits entirely inside one hemisphere's half-year and
  misses both sample months;
- historical years where a zone observed year-round DST (US 1974, `America/*`),
  so both samples carry the DST offset and every instant reads `isdst: false`;
- a zone whose standard offset changed between January and July of the queried
  year, where the smaller sample is a different _standard_ offset rather than a
  DST one.

`#mktimeIsdst` (same file) sits on top of this reading, so a wrong `isdst` also
picks the wrong occurrence for `Time.local`'s ten-argument form and, through it,
for `change`'s zoned arm.

## Converged shape

Read the zone's actual DST flag rather than sampling two months. Options to
price, cheapest first:

1. `Intl.DateTimeFormat(..., { timeZoneName: "shortGeneric" })` vs `"short"` —
   distinguishes "Eastern Time" from "EDT", but is locale data, not tzdata.
2. Compare the instant's offset against the offset immediately **before the
   zone's most recent transition at or before it**, which `Temporal`'s
   `getTimeZoneTransition` exposes directly:
   `zoned.getTimeZoneTransition({ direction: "previous" })`. The standard offset
   is then a local property of the surrounding transitions rather than a
   whole-year guess. This is the one to try first — it is exact for the
   year-round-DST and shifted-standard-offset cases above.
3. A tzdata `isdst` bit, if one of the vendored zone tables carries it.

Whichever lands, `tzdataAbbreviation`'s January/July sampling in the same file
is the sibling instance of the identical heuristic — converge it in the same
pass or file the follow-up, but do not leave one converged and one not.

Regression guard: add cases for a year-round-DST year and for a southern-
hemisphere zone (`Australia/Lord_Howe` is already exercised by
`core-ext/time-ext.test.ts`), asserting against `ruby -e 'p Time.local(...).isdst'`
under the matching `TZ`.

## Acceptance criteria

- [ ] `Time#isdst` reads the receiver's actual DST state, not a January/July
      offset comparison.
- [ ] A year-round-DST year and a southern-hemisphere zone both answer what MRI
      answers.
- [ ] `Time.local`'s ten-argument form and `change`'s `elsif zone` arm still
      pick the receiver's occurrence.
- [ ] `tzdataAbbreviation`'s matching heuristic is converged with it or its
      follow-up is filed.
