---
title: "Remove the toJSON fallback arm from core-ext/object/json.ts's asJson dispatcher"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6205 ported `core_ext/object/json.rb`'s `as_json` layer into
`packages/activesupport/src/core-ext/object/json.ts`. Its `asJson()` dispatcher
— which stands in for Ruby's method lookup, since TS cannot reopen built-ins —
carries one arm with no Rails counterpart at all, just before its
`Object.asJson` tail:

```ts
// A JS built-in carrying its own JSON form (`Date`, …). Ruby's counterparts
// define `as_json`; calling `toJSON` reaches the same primitive, where
// recursing into the instance as an attribute bag would emit `{}`.
if (typeof (value as { toJSON?: unknown }).toJSON === "function") {
  return (value as { toJSON(): unknown }).toJSON();
}
```

Ruby has no `to_json`-as-`as_json` fallback; `json.rb` gives every relevant
class an explicit `as_json`. The arm exists because trails values can still be
JS built-ins (`Date`, `URL`, …) that predate the Temporal migration, and
spreading one as an attribute bag would emit `{}`.

It is load-bearing today: `json/encoding.test.ts`'s "hash with time to json"
and "time to json includes local offset" both put a JS `Date` through the
encoder.

## Converged shape

Delete the arm. Rails' answer is an explicit `as_json` per class, so each JS
built-in that reaches the encoder gets a named arm from `json.rb` instead:
a JS `Date` is the `Time` analogue (`json.rb:200-208`) and should route to
`Time.asJson`, a `URL` is `URI::Generic#as_json` (`json.rb:230-234`, `to_s`).
Anything left with no Rails counterpart is a caller bug, not a case for a
generic fallback.

Check first whether a JS `Date` should reach the encoder at all — trails' Time
analogue is `Temporal.Instant`/`ZonedDateTime`
(see `project_js_date_rejected_temporal_is_time_analogue`), so the right
convergence may be at the call sites rather than in the dispatcher.

## Acceptance criteria

- [ ] The `toJSON` fallback arm is gone from `core-ext/object/json.ts`.
- [ ] Each type that relied on it has a named arm cited to its `json.rb` line
      range, or its call site stopped producing that type.
- [ ] `json/encoding.test.ts` and `json/encoding.trails.test.ts` green; no test
      renamed.
- [ ] `pnpm parity:api:extra --package activesupport` clean; `pnpm parity:api
--package activesupport` non-negative.
