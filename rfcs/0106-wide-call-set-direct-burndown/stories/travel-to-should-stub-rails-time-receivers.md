---
title: "travel-to-should-stub-rails-time-receivers"
status: done
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6872
claim: "2026-08-22T18:49:58Z"
assignee: "travel-to-should-stub-rails-time-receivers"
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/testing/time-helpers.ts` `travelTo` diverges from
`vendor/rails/activesupport/lib/active_support/testing/time_helpers.rb:178-190`:
Rails stubs `Time.now`, `Time.new`, `Date.today` and `DateTime.now`
individually, and each stub body calls a constructor on the stubbed receiver
(`at(now)`, `jd(...)`). trails routes all four through a single internal
`clock` object taking a `Temporal.Instant` directly, so no `Time.at` call is
made.

Three deviation receipts sit on this one method:

- `@missingRailsCall at — CONVERGEABLE` at the call site (minted by
  `wave-5b-tail-sweep`).
- Two `kind: "args"` rows still in
  `scripts/api-compare/call-mismatches-exclude/activesupport/testing/time-helpers.json`
  for `stub_object` and `stubbing`, both citing the same `clock` indirection
  (`const:Time`, `str:now` vs the trails clock value).

They are one deviation, and converging the receiver shape retires all three.

## Acceptance criteria

- [ ] `travelTo` stubs the same receivers Rails stubs, so `stub_object` /
      `stubbing` receive the Rails argument shapes and `at`/`jd` are called on
      the stubbed constructors.
- [ ] The `@missingRailsCall at` tag and both `args` baseline rows are deleted,
      not reworded; the emptied shard is removed.
- [ ] `pnpm parity:api:calls`, `parity:api:calls:args`, `parity:api:reasons`,
      `parity:api:detached` green.
