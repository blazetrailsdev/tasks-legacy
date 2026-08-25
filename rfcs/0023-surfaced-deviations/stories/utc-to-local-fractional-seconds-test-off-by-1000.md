---
title: "utc_to_local fractional-seconds test asserts a millisecond where Rails asserts a microsecond"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`activesupport/test/time_zone_test.rb:25-38` (`test_utc_to_local_with_fractional_seconds`)
builds its input as `Time.utc(2000, 1, 1, 0, 0, 0, 1)` — usec = **1**, one
microsecond — and asserts `Time.utc(1999, 12, 31, 19, 0, 0, 1)` on the legacy
arm and `Time.new(1999, 12, 31, 19, 0, usec, -18000)` on the tzinfo-2 arm,
where `usec` is `Rational(1, 1_000_000)` (`:26-27`).

trails' port, `packages/activesupport/src/time-zone.test.ts:55-76`
(`it("utc to local with fractional seconds")`), passes **1000** instead of 1
and asserts `nsec === 1_000_000` / `standard.millisecond === 1`. That is a
whole millisecond, not the microsecond Rails pins, so the test exercises none
of the sub-millisecond precision its Rails counterpart exists to protect.

This is how the truncation fixed in trails#6966 got in: `utcToLocal`'s legacy
arm folded only `millisecond * 1_000 + microsecond` into `Time.utc`'s usec
positional and the mirrored test stayed green. The regression cover added in
that PR lives in the trails-only file
(`time-zone.trails.test.ts`, `"utc_to_local keeps digits below the microsecond
on the legacy arm"`); the mirrored test is still off-value.

Rails: `activesupport/test/time_zone_test.rb:25-38`,
`activesupport/lib/active_support/values/time_zone.rb:542-545`.

## Converged shape

`time-zone.test.ts`'s `utc to local with fractional seconds` passes `1` as the
usec positional on all four `Time.utc(...)` calls and asserts what Rails
asserts — `Time.utc(1999, 12, 31, 19, 0, 0, 1)` / `Time.utc(2000, 6, 30, 20, 0,
0, 1)` on the legacy arm, and the `Rational(1, 1_000_000)` second on the
tzinfo-2 arm (`standard.microsecond === 1`, not `standard.millisecond === 1`).
Do NOT rename the test.

## Acceptance criteria

- [ ] All four `Time.utc` calls in the mirrored test pass usec `1`.
- [ ] Legacy-arm assertions compare against `Time.utc(..., 0, 1)`, matching
      `time_zone_test.rb:30-31`.
- [ ] The tzinfo-2 arm asserts the microsecond component, matching `:35-36`.
- [ ] Test name unchanged; `pnpm parity:test` delta non-negative.
