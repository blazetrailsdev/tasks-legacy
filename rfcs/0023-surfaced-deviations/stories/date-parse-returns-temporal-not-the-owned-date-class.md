---
title: "Date.parse / DateTime.parse answer Temporal values instead of the package's own Date / DateTime"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "date"
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by trails#6924 (RFC 0098 `time-with-zone-residue-structural-blockers`,
arm D), which settled where the `acts_like?` markers live.

Ruby's `Date.parse` answers a `::Date` and `DateTime.parse` a `::DateTime`
(ruby/date `ext/date/date_core.c` `date_s_parse` / `datetime_s_parse`; the
Ruby-visible surface is `lib/date.rb`). trails' answer a third party's values
instead:

- `packages/date/src/date.ts:6869` — `Date.parse(...): Temporal.PlainDate`
- `packages/date/src/date.ts:9280` — `DateTime.parse(...): Temporal.PlainDateTime | Temporal.ZonedDateTime`

The classes themselves exist and are the port's `::Date` / `::DateTime`
(`date.ts:6145`, `date.ts:8509`) — the builders just convert away from them on
the way out (`.toDate()` / `.toDatetime()`).

That is what forced the deviation #6924 had to ship. Rails hangs
`acts_like_date?` / `acts_like_time?` by reopening the class
(`activesupport/lib/active_support/core_ext/date/acts_like.rb:5-9`,
`activesupport/lib/active_support/core_ext/date_time/acts_like.rb:6-13`), and a
marker method needs an instance of the class to sit on. Where trails owns the
class AND returns it, the port is literal: `Time#actsLikeTime`
(`packages/date/src/time.ts`, mirroring
`activesupport/lib/active_support/core_ext/time/acts_like.rb:5-8`) is a real
marker that `Object.actsLike` finds through `respondTo`, exactly as Ruby does.
Where trails returns `Temporal` values it cannot, so
`packages/date/src/acts-like.ts` stands in with `actsLikeDate` / `actsLikeTime`
predicates carrying `@noRailsEquivalent PERMANENT` receipts, and the
`SCOPED_SKIP_GROUPS` entry in `scripts/parity/conventions.ts` records the lost
Rails file path for those members (RFC 0098's reachable ceiling dropped by 1).

Note this is scoped to what the two `parse` builders (and their `iso8601` /
`_strptime` siblings) hand back. It is NOT a proposal to install markers on the
`Temporal` polyfill prototypes — RFC 0098 rejected that as a global side effect
on a third-party package, and that decision stands.

## Converged shape

`Date.parse` returns a `Date` and `DateTime.parse` returns a `DateTime`, as
`lib/date.rb` does. Then both classes carry real `actsLikeDate` /
`actsLikeTime` marker methods the way `Time` already does, `Object.actsLike`
answers all three arms through `respondTo` with no predicate standing in, and
the stand-in module retires.

Expect the cost to be in the call sites, not the builders: every consumer that
today receives a `Temporal.PlainDate` from `Date.parse` and reads
`dayOfWeek`/`month` off it has to go through the `Date` surface instead. Size
the sweep before committing to it — this may want splitting per consumer
cluster, and `Temporal` almost certainly stays the internal representation
behind the class.

## Acceptance criteria

- [ ] `Date.parse` / `DateTime.parse` (and the `iso8601` siblings that share
      their build) answer this package's own `Date` / `DateTime`, per
      `lib/date.rb`.
- [ ] `Date` and `DateTime` carry `actsLikeDate` / `actsLikeTime` marker
      methods, mirroring the reopened Rails classes.
- [ ] `packages/date/src/acts-like.ts` shrinks to only the receivers that
      genuinely cannot carry a marker — ideally to nothing — and every
      `@noRailsEquivalent` receipt it no longer needs is deleted.
- [ ] The `SCOPED_SKIP_GROUPS` entry in `scripts/parity/conventions.ts` for the
      three `acts_like.rb` files is narrowed to whatever actually remains, and
      RFC 0098's changelog records the ceiling coming back up.
- [ ] `pnpm parity:api`, `pnpm parity:test` deltas non-negative;
      `pnpm parity:api:extra` clean.
