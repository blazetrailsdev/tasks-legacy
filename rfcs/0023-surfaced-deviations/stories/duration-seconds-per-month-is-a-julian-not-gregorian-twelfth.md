---
title: "Duration::SECONDS_PER_MONTH is a Julian twelfth (2629800), not Rails' Gregorian 2629746"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Partially fixed in #6465

PR #6465 landed both constants as the Rails integers in
`packages/activesupport/src/duration.ts` and added the `PARTS_IN_SECONDS` table:

```ts
const SECONDS_PER_MONTH = 2629746; // 1/12 of a gregorian year (duration.rb:117)
const SECONDS_PER_YEAR = 31556952; // length of a gregorian year (duration.rb:118)
```

Still outstanding from the acceptance criteria below:

- `SECONDS_PER_WEEK` is still `7 * SECONDS_PER_DAY` rather than the Rails
  literal `604800` (`duration.rb:116`) — readability only, same value.
- **No cover was added** asserting `Duration.months(1).inSeconds() === 2629746`
  and `Duration.years(1).inSeconds() === 31556952`. Without it the old values
  can regress silently, which is the whole point of this story.

Existing expectations did not need correcting — the suites were green on the new
values.

## Context

Found while porting `core_ext/date/calculations.rb` in PR #6286.

`packages/activesupport/src/duration.ts:28` defines

```ts
const SECONDS_PER_MONTH = 30.4375 * SECONDS_PER_DAY; // 1/12 of 365.25 * 86400
```

which is **2629800**. Rails is
`vendor/rails/activesupport/lib/active_support/duration.rb:117`:

```ruby
SECONDS_PER_MONTH  = 2629746  # 1/12 of a gregorian year
```

2629746 is 1/12 of `SECONDS_PER_YEAR = 31556952` — a _Gregorian_ year of
365.2425 days. trails' comment says "1/12 of 365.25", a _Julian_ year, so the
constant is 54 seconds per month too large. `SECONDS_PER_YEAR`
(`duration.ts:29`) is already correct at 365.2425 days.

Observable: `1.month.to_i` is 2629746 in Rails and 2629800 in trails, and every
`inSeconds()`-derived reader inherits the drift — `inMonths()`, `inYears()`,
`modulo`, and the `Duration` comparisons built on them. `13.months % 1.year`
(`duration_test.rb` `test_plus`) is the kind of assertion that turns on it.

The parity:api literal comparator cannot catch this: it skips non-literal
initialisers, and both trails constants are computed expressions
(`output/literal-mismatches.json` reports 1 mismatch and it is an unrelated
i18n one).

Not fixed in #6286, which was scoped to `core_ext/date/calculations.rb` and only
touched `Duration`'s part-key handling.

## Converged shape

Spell both constants as the Rails integers rather than as products, so they
match `duration.rb:113-118` by inspection:

```ts
const SECONDS_PER_MONTH = 2629746; // 1/12 of a gregorian year
const SECONDS_PER_YEAR = 31556952; // length of a gregorian year (365.2425 days)
```

`SECONDS_PER_WEEK` is likewise `604800` in Rails (`:116`) where trails computes
`7 * SECONDS_PER_DAY`; same value, so converge it for readability only.

## Acceptance criteria

- `SECONDS_PER_MONTH` is 2629746 and `SECONDS_PER_YEAR` is 31556952, spelled as
  the Rails integers with the Rails comments.
- A cover asserting `Duration.months(1).inSeconds() === 2629746` and
  `Duration.years(1).inSeconds() === 31556952`.
- Existing `duration.test.ts` / `core-ext/duration.test.ts` expectations that
  encoded the old value are corrected against Ruby, not loosened.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
