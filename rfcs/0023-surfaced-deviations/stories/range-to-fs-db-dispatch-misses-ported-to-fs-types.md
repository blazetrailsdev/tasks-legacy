---
title: "RANGE_FORMATS[:db] endpoint dispatch falls back to String() for BigDecimal/PlainDateTime"
status: draft
updated: 2026-08-05
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

`RANGE_FORMATS[:db]` in Rails
(`vendor/rails/activesupport/lib/active_support/core_ext/range/conversions.rb:7-27`)
interpolates each endpoint by calling `to_fs(:db)` **on the endpoint itself**:

```ruby
when String then "BETWEEN '#{start}' AND '#{stop}'"
else
  "BETWEEN '#{start.to_fs(:db)}' AND '#{stop.to_fs(:db)}'"
```

That is an open dispatch — every class with a `to_fs` participates
(`Date`, `Time`, `DateTime`, `Numeric`/`BigDecimal` via
`core_ext/numeric/conversions.rb`, and any app class that defines one).

PR #6101 ported this in
`packages/activesupport/src/core-ext/range/conversions.ts` as a private
`toFsDb(value)` helper, because TS has no open classes to dispatch through.
It handles the endpoint types trails uses today and falls back to `String(v)`:

```ts
function toFsDb(value: unknown): string {
  if (value instanceof Temporal.PlainDate) return value.toString();
  if (value instanceof Date) return timeToFs(value, "db");
  if (value instanceof Temporal.Instant) return timeToFs(new Date(value.epochMilliseconds), "db");
  return String(value);
}
```

The gap: an endpoint whose Ruby counterpart HAS a ported `to_fs` but is not in
that `if` chain silently takes the `String(value)` fallback instead of its own
`:db` format. The known case is `BigDecimal`
(`packages/activesupport/src/core-ext/big-decimal/conversions.ts`, ported from
`core_ext/big_decimal/conversions.rb`) — Ruby's
`BigDecimal#to_fs(:db)` routes through `NumericWithFormat#to_fs`, trails'
returns whatever `String()` produces. `Temporal.PlainDateTime` /
`ZonedDateTime` endpoints are unhandled for the same reason.

Range endpoints are usually numbers, strings, or dates, so this is latent
rather than actively wrong — but it is a real divergence in a ported body, not
a language limit, since the dispatch table can be extended.

## Converged shape

Replace the hand-rolled `if` chain with a lookup that covers every trails type
whose Ruby counterpart defines `to_fs`, so adding a new `to_fs` port
automatically participates the way reopening a Ruby class does. Keep `toFsDb`
private (it is the TS stand-in for the dispatch, not a Rails method) and keep
the `case start when String` arm above it exactly as
`conversions.rb:10,15,22` has it — that arm is Rails', not part of this gap.

At minimum add: `BigDecimal`, `Temporal.PlainDateTime`,
`Temporal.ZonedDateTime`.

## Acceptance criteria

- A `BigDecimal` range endpoint formats through the ported `to_fs(:db)`, not
  `String()`.
- `Temporal.PlainDateTime` / `ZonedDateTime` endpoints format as `:db`.
- Existing `to_fs` assertions in
  `packages/activesupport/src/core-ext/range-ext.test.ts` still pass unchanged
  (they mirror `range_ext_test.rb:9-35`); do NOT rename any test.
- No new `parity:api:extra` surface — the dispatch stays a private helper.
