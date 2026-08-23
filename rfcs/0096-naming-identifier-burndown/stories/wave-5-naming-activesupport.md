---
title: "Wave 5: burn down the AR-closure naming rows in activesupport"
status: in-progress
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activesupport"]
deps: []
deps-rfc: []
est-loc: 100
priority: 10
pr: 6917
claim: "2026-08-23T14:04:32Z"
assignee: "wave-5-naming-activesupport"
blocked-by: null
closed-reason: null
---

## Context

RFC 0096's closing story `naming-gate-flip` is blocked on the AR require-closure
reaching **zero convergeable `naming` rows** (`burndown` +
`module-mixin-receiver`). Waves 1-4 drained the population it was scoped
against; a fresh reading of
`scripts/api-compare/output/call-arg-mismatches.json` (artifact of 2026-08-21,
rendered by `pnpm parity:api:calls:args:report`) shows **107 convergeable rows
still standing inside the closure**, across 67 files. This wave-5 band splits
those 107 into six non-overlapping file sets so the flip has a defined finish
line again.

Rules are unchanged from the RFC's `## Design`:

- **Rename to the Rails identifier, not to a better one.** If Rails writes `o`,
  the TS local is `o`, camelCased per `docs/ruby-ts-conventions.md`.
- **Body-local only.** No behavior change, no public surface change.
- **A row that is really an a1 (argument order) or a3 (invented helper /
  conversion) finding is NOT renamed away.** File it against the RFC that owns
  the file and leave the row. Several rows below look like that shape — e.g.
  `activesupport/notifications.ts#instrument` passes a different argument list,
  not a differently-named one — so read each pair against the vendored Ruby
  before renaming.
- `module-mixin-receiver` rows converge by rewiring to the `this`-typed mixin
  idiom (CLAUDE.md, Module mixins), not by renaming the parameter.

## Rows in this slot

26 rows across 18 files. **File set:** `activesupport` (all files listed below; no activerecord files).

- `activesupport/core-ext/date-and-time/calculations.ts` — 4
  - `beginning_of_quarter`: ruby `ref:beginningOfMonth,kwargs{month=ref:firstQuarterMonth}` vs ts `ref:receiver,kwargs{month=ref:firstQuarterMonth}`
  - `end_of_quarter`: ruby `ref:beginningOfMonth,kwargs{month=ref:lastQuarterMonth}` vs ts `ref:receiver,kwargs{month=ref:lastQuarterMonth}`
- `activesupport/time-zone-config.ts` — 3
  - `zone=`: ruby `ref:timeZone` vs ts `ref:zone`
  - `use_zone`: ruby `ref:timeZone` vs ts `ref:zone`
- `activesupport/duration.ts` — 2
  - `sum`: ruby `ref:parts` vs ts `ref:_partKeys`
  - `parse`: ruby `ref:calculateTotalSeconds,ref:parts` vs ts `ref:value,ref:parts`
- `activesupport/inflector.ts` — 2
  - `classify`: ruby `ref:sub` vs ts `ref:stripped`
  - `apply_inflections`: ruby `ref:result` vs ts `ref:word`
- `activesupport/values/time-zone.ts` — 2
  - `rfc3339`: ruby `ref:utc,ref:this` vs ts `ref:instantFrom,ref:this`
  - `period_for_utc`: ruby `ref:time` vs ts `ref:date`
- `activesupport/cache/coder.ts` — 1
  - `load`: ruby `ref:byteslice` vs ts `ref:rawVersion`
- `activesupport/cache/file-store.ts` — 1
  - `modify_value`: ruby `ref:key,ref:entry` vs ts `ref:key,ref:constructor`
- `activesupport/array-utils.ts` — 1
  - `to_xml`: ruby `ref:name` vs ts `ref:rubyClassName`
- `activesupport/core-ext/date/calculations.ts` — 1 (mmr 1)
  - `plus_with_duration`: ruby `ref:this` vs ts `ref:date`
- `activesupport/core-ext/hash/conversions.ts` — 1
  - `process_hash`: ruby `ref:detect` vs ts `ref:find`
- `activesupport/enumerable-utils.ts` — 1
  - `presence_in`: ruby `ref:anotherObject` vs ts `ref:value`
- `activesupport/time-ext.ts` — 1
  - `advance`: ruby `ref:timeAdvancedByDate,ref:secondsToAdvance` vs ts `ref:constructor,ref:secondsToAdvance`
- `activesupport/encrypted-file.ts` — 1
  - `read`: ruby `ref:strip` vs ts `ref:raw`
- `activesupport/hash-with-indifferent-access.ts` — 1
  - `update_with_single_argument`: ruby `ref:value` vs ts `ref:v`
- `activesupport/transliterate.ts` — 1
  - `transliterate`: ruby `ref:unicodeNormalize,kwargs{locale=ref:locale,replacement=ref:replacement}` vs ts `ref:normalize,kwargs{locale=ref:locale,replacement=ref:replacement}`
- `activesupport/message-pack/serializer.ts` — 1
  - `load`: ruby `ref:messagePackPool` vs ts `ref:dumped`
- `activesupport/notifications.ts` — 1
  - `instrument`: ruby `ref:instrumenter,ref:name,ref:payload` vs ts `ref:name,ref:payload,ref:block`
- `activesupport/subscriber.ts` — 1
  - `add_event_subscriber`: ruby `ref:pattern,ref:subscriber` vs ts `ref:pattern,ref:sub`

## Acceptance criteria

1. Every convergeable (`burndown` / `module-mixin-receiver`) `naming` row in the
   file set above is either converged to the Rails identifier, rewired to the
   `this`-typed mixin idiom, or re-filed as an a1/a3 finding against the RFC
   that owns the file — with the re-filed story id named in the PR body.
2. `pnpm parity:api:calls:args:report` shows this slot's convergeable count at
   **zero**; no row in the file set is added to any
   `call-mismatches-exclude/` shard (CLAUDE.md — converge, never ratify).
3. No public surface, method name, field name or behavior changes; the diff is
   locals and parameters only (plus mixin-receiver rewiring where it applies).
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls:args` stays green.

## Notes for the claimer

The per-file counts above are from the 2026-08-21 parity artifact and are
**advisory**. Re-run
`API_COMPARE_FORCE=1 pnpm parity:api --calls && pnpm parity:api:calls:args:report`
at claim time and work from the fresh reading — counts drift as sibling RFCs
touch the same files.
