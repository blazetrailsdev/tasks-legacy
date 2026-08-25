---
title: "Type::Internal::Timezone is imported, not mixed in — is_utc?/default_timezone unmatched on date/datetime/time"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while relocating `ActiveRecord.default_timezone` (PR #5566).

`vendor/rails/activerecord/lib/active_record/type/internal/timezone.rb` is a
module mixed into the temporal types, so `is_utc?` and `default_timezone`
resolve as members of `Type::Date`, `Type::DateTime` and `Type::Time`.

trails ports it as a standalone `isUtc()` function plus a `Timezone` class in
`packages/activerecord/src/type/internal/timezone.ts`, and the type classes
(`type/date.ts`, `type/date-time.ts`, `type/time.ts`) import `isUtc` rather than
mixing the module in. `pnpm parity:api` therefore reports `default_timezone`
(and `is_utc?` for date/time) as missing per file:

- activerecord: `type/date.rb`, `type/date_time.rb`, `type/time.rb`
- activemodel: the same three, via its parallel
  `type/helpers/timezone.rb`

That is 7 unmatched entries across the two packages purely from mixin shape.
CLAUDE.md's "Module mixins" section prescribes `include()` / `Included<>` from
`@blazetrails/activesupport` for instance methods mixed in bulk, which is the
mechanism this should use.

## Acceptance criteria

- The temporal types mix the timezone module in (via `include()` / `Included<>`)
  instead of importing a standalone `isUtc`, so `isUtc` and `defaultTimezone`
  resolve as members of each type class.
- `pnpm parity:api` no longer lists `default_timezone` / `is_utc?` as missing
  for `type/date.rb`, `type/date_time.rb`, `type/time.rb` in activerecord, and
  the activemodel counterparts are converged or explicitly scoped out.
- The trails-only `Timezone` class / `TimezoneOptions` shape is removed or
  justified at the call site if it must stay.
- No new `parity:api:extra` entries; existing timezone tests pass with names unchanged.
