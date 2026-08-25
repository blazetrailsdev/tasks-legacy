---
title: "Point core_ext/date/conversions.rb at core-ext/date/conversions.ts and port its remaining members"
status: draft
updated: 2026-08-17
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

`scripts/parity/conventions.ts:249` maps
`activesupport:core_ext/date/conversions.rb` -> `time-ext.ts`, dating from when
the `Date` arm had no file of its own. It now does:
`packages/activesupport/src/core-ext/date/conversions.ts` holds `DATE_FORMATS`,
`toFs`, `toFormattedS` (PR #6645) and `toTime`, at Rails' own file layout.

The stale override has two visible costs:

- `parity:api:extra` scores `core-ext/date/conversions.ts` as
  `[no Rails counterpart]` — an empty allowed set — so every genuine Rails name
  in it (`DATE_FORMATS`, `long_ordinal`, `iso8601`, `toFs`, `toFormattedS`,
  `toTime`) is counted as extra surface.
- `core_ext/date/conversions.rb`'s members are credited against `time-ext.ts`,
  where the `Time` arm's same-named members (`to_fs`, `xmlschema`) stand in for
  them, so the `Date` arm's parity is not actually measured.

## Converged shape

Point the override at `core-ext/date/conversions.ts` and port the members of
`core_ext/date/conversions.rb` that the move would leave uncredited, onto that
file at their Rails spelling:

- `Date#readable_inspect` (`core_ext/date/conversions.rb:63-65`) — `strftime("%a, %d %b %Y")`
- `Date#default_inspect` / `inspect` (`:66-67`)
- `Date#xmlschema` (`:96-98`) — `in_time_zone.xmlschema`

Rails anchors: `activesupport/lib/active_support/core_ext/date/conversions.rb:8-98`.
Check the `parity:api` methods/files delta before and after — the point is that
it goes UP, because the `Date` arm starts being compared against its own file.

## Acceptance criteria

- [ ] `activesupport:core_ext/date/conversions.rb` maps to
      `core-ext/date/conversions.ts` in `scripts/parity/conventions.ts` (and
      `docs/ruby-ts-conventions.md` regenerates from it).
- [ ] `readable_inspect`, `default_inspect` and `xmlschema` are ported on the
      `Temporal.PlainDate` receiver in that file.
- [ ] `parity:api:extra --package activesupport` no longer reports
      `core-ext/date/conversions.ts` as `[no Rails counterpart]`.
- [ ] `pnpm parity:api` / `parity:test` deltas non-negative;
      `parity:api:calls` / `:args` clean.
