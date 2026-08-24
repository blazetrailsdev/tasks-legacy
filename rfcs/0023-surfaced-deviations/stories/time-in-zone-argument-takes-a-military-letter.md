---
title: "time-in-zone-argument-takes-a-military-letter"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

MRI takes a MILITARY zone letter wherever `Time` takes a `utc_offset` —
`"A".."I"`, `"K".."Z"`, with `"Z"` meaning UTC and `"J"` excluded — and
rejects an IANA identifier outright:

```console
$ ruby -e 'p Time.now(in: "Z").utc?'
true
$ ruby -e 'p Time.now(in: "Etc/UTC")'
"+HH:MM", "-HH:MM", "UTC" or "A".."I","K".."Z" expected for utc_offset: Etc/UTC (ArgumentError)
```

trails does the opposite. `Time.#atInstant`
(`packages/date/src/time.ts`) hands a string `zone` straight to
`Temporal.Instant#toZonedDateTimeISO`, so `Time.now({ in: "Z" })` throws
`RangeError: Unrecognized time zone Z` where MRI answers a UTC time, and an
IANA identifier is accepted where MRI raises `ArgumentError`.

`utc_offset` is read by `zone_offset` / `utc_offset_arg` (`ruby/time.c`
`time_zone_offset`, and `timev.rb`'s message above); the letter table is
`zone_offset`'s `A`..`Z` arm.

Found while fixing the UTC-only `Time#utc?` bug in PR #6956 — that PR bases
the new `#tzmodeUtc` bit on the RESOLVED zone id (`zoned.timeZoneId === "UTC"`)
specifically so a `"Z"` that starts resolving lands as a UTC time with no
further change here.

## Converged shape

Read a string `zone` through the same `utcOffsetArgument` path the constructor
uses, so a military letter resolves to its offset (`"Z"` → `0`, and a UTC-mode
time) and an unrecognized spelling raises MRI's `ArgumentError` with MRI's
message rather than a Temporal `RangeError`.

Decide deliberately whether trails keeps accepting IANA identifiers as an
extension — the rest of the port leans on them (`#timeZoneId`, `zone`,
`isdst`, the tzdata walk), so this is likely a documented widening rather than
a straight port, but it should be stated at the call site rather than falling
out of the implementation.

## Acceptance criteria

- [ ] `Time.now({ in: "Z" })` answers a time with `isUtc()` true, not a throw.
- [ ] The military-letter table `"A".."I"`, `"K".."Z"` resolves to its offset.
- [ ] An unrecognized zone raises MRI's `ArgumentError` and message.
- [ ] Any accepted-IANA widening is justified at the call site.
