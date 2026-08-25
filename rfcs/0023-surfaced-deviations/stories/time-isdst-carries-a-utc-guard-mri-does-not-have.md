---
title: 'Time#isdst''s === "UTC" early return is a heuristic leftover MRI has no branch for'
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "date"
deps: []
deps-rfc: []
est-loc: 25
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Time#isdst` (`packages/date/src/time.ts:983-986`) carries a branch MRI does not:

```ts
if (this.#timeZoneId == null || this.#timeZoneId === "UTC") return false;
```

MRI's `time_isdst` (`ruby/time.c`) has no UTC special case — it answers
`tm.tm_isdst`, and a UTC time's zoneinfo record simply carries `isdst = 0`. The
`== null` half is real (a time built from an offset has no zone to ask, so MRI
answers `false`), but the `"UTC"` half was a crutch of the January/July offset
heuristic that PR #6949 replaced: `tzdataIsdst("UTC", …)` already answers
`false`, because `UTC` has no row in the vendored table and a missing row means
"never observed DST".

The same shape may sit in `tzdataAbbreviation`'s neighbours — check for other
`=== "UTC"` guards in `time.ts` that the table now answers for.

## Converged shape

Drop the `|| this.#timeZoneId === "UTC"` arm, leaving the `== null` guard that
mirrors MRI's "no zone" reading. `packages/date/src/tzdata-isdst.trails.test.ts`
already pins `tzdataIsdst("Etc/UTC", …) === false`; add the `"UTC"` spelling
beside it, and keep `time.trails.test.ts`'s `Time.utc(...)` cases green.

## Acceptance criteria

- [ ] No `=== "UTC"` guard remains in `Time#isdst`.
- [ ] `Time.utc(...).isdst` is still `false`, via the table rather than the branch.
- [ ] The date suite stays green.
