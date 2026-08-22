---
title: "time-helpers-stub-date-and-datetime-clock"
status: blocked
updated: 2026-08-22
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-14T22:49:42Z"
assignee: "retire-time-zone-config-test-only-zone-seams"
blocked-by: "Narrowed by PR #6872. The two named prerequisites now EXIST: `Time.at` takes a Time (time.c time_s_at, argument's own zone) and `Time.new` is ported (time_s_init), and travel_to already stubs Time.now / Time.new / Date.today / DateTime.now with Rails' bodies (time_helpers.rb:177-190) — both `kind: args` rows and the `at` receipt are retired and the shard is deleted. What is left is ONLY the hot-path cost this story always named, now measured: currentTimeInstant() (time-travel.ts) is read on every TimeWithZone construction, and Time.now() costs 0.52ms/op against 0.008ms/op for a bare Temporal.Now.instant() — ~70x, dominated by Temporal.Now.timeZoneId() at 0.117ms plus the Time constructor. So travel_to stubs the `clock` seat alongside the four Rails receivers, justified at that call site. Unblocking means making @blazetrails/date's Time.now cheap enough for that path (cache the resolved timeZoneId; skip the validating constructor on the #atInstant seat), not re-deciding the dependency direction."
closed-reason: null
---

## Context

Residue of `time-helpers-stub-real-clock-methods` (PR for that story converged
`travel_to` onto a single stub mechanism — `SimpleStubs` over `time-travel.ts`'s
`clock.now` — parsed the String arm through `Time.zone.parse`, and wired
`afterTeardown` into `packages/activerecord/src/cases/helper.ts`).

Two things are left in
`scripts/api-compare/call-mismatches-exclude/activesupport/testing/time-helpers.json`:

- `at` — Rails' `stubs.stub_object(Time, :now) { at(now) }` builds the stubbed
  value with `Time.at`; the trails clock method returns a `Temporal.Instant`
  directly (time_helpers.rb:177-178).
- `order:parse,toTime` — Rails' FIRST `to_time` is the Ruby `Date` arm's
  `date_or_time.midnight.to_time` (time_helpers.rb:162-163). trails' Date arm
  takes a JS `Date`, which is a Ruby `Time`, not a Ruby `Date`, so the only
  `toTime` left is the one after `Time.zone.parse` and the pair is inverted.

Rails also stubs `Time.new`, `Date.today` and `DateTime.now`
(time_helpers.rb:180-190). trails stubs only the one clock method production
code reads; `@blazetrails/date`'s `Time.now` / `Date.today` / `DateTime.now`
read `Temporal.Now` directly and are not travel-aware. Making them travel-aware
means routing the `date` package's clock through the trails clock seam, which is
its own design question (dependency direction, hot-path cost on
`currentTimeInstant`).

## Acceptance criteria

- [ ] Decide and implement the clock seam for `@blazetrails/date`'s `Time.now`,
      `Date.today` and `DateTime.now` so `travel_to` stubs them the way
      time_helpers.rb:177-190 does, or block the story with the specific
      dependency-direction blocker.
- [ ] Both remaining rows in
      `call-mismatches-exclude/activesupport/testing/time-helpers.json` are
      deleted.
