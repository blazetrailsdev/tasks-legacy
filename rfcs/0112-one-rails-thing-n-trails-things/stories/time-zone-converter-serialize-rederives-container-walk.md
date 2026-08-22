---
title: "TimeZoneConverter#serialize re-derives the container walk instead of delegating"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `TimeZoneConverter` is a `DelegateClass(Type::Value)` and defines no
`serialize` / `serialize_cast_value` of its own
(`vendor/rails/activerecord/lib/active_record/attribute_methods/time_zone_conversion.rb:8-52`):
both forward straight to the subtype, which handles a `TimeWithZone` because it
`acts_like?(:time)`.

trails carries a `_resolveForSerialize` helper plus the local `isRangeLike` /
`mapRange` / `RangeLike` trio
(`packages/activerecord/src/attribute-methods/time-zone-conversion.ts:168-178,
~300-340` as of PR #6485) that walks Arrays and Ranges to strip `TimeWithZone`
down to a `Temporal.Instant` before delegating, because `DateTimeType.castValue`
cannot parse a `TimeWithZone`.

PR #6485 removed the same re-derived container dispatch from `cast` and
`deserialize` by routing them through the subtype's `map` hook
(`activemodel/lib/active_model/type/value.rb:117-119`,
`connection_adapters/postgresql/oid/range.rb:50-54`,
`connection_adapters/postgresql/oid/array.rb:67-69`). The serialize path is the
last holdout and keeps `isRangeLike`/`mapRange` alive in the file.

## Converged shape

Teach the time types to accept a `TimeWithZone` the way Ruby's `Time`-alike is
accepted (`ActiveModel::Type::DateTime#cast_value` -> `value.acts_like?(:time)`),
then delete `serialize` / `serializeCastValue` / `_resolveForSerialize` /
`isRangeLike` / `mapRange` / `RangeLike` from `time-zone-conversion.ts` and let
DelegateClass forwarding do the work, exactly as Rails does.

## Acceptance criteria

1. `TimeZoneConverter` no longer overrides `serialize` / `serializeCastValue`
   with a bespoke pre-walk (or, if a thin DelegateClass forward is still needed
   in TS, it forwards the value untouched).
2. `_resolveForSerialize`, `isRangeLike`, `mapRange` and `RangeLike` are deleted
   from `attribute-methods/time-zone-conversion.ts`.
3. `pnpm parity:api:extra --package activerecord` shows no new novel surface;
   the PG `range`/`array`/`timestamp` and `attribute-methods/*` suites stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
