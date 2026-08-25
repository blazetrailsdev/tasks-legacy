---
title: "serialize_cast_value drops the is_utc? getutc/getlocal arm"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Flagged in review of PR #6738, which mixed `Helpers::TimeValue` into
`DateTimeType` (story
`converge-date-time-apply-seconds-precision-onto-time-value-mixin`). Out of
that PR's scope; filed rather than folded in.

Rails' `serialize_cast_value` (`time_value.rb:10-21`) is TWO steps:

```ruby
def serialize_cast_value(value)
  value = apply_seconds_precision(value)

  if value.acts_like?(:time)
    if is_utc?
      value = value.getutc if !value.utc?
    else
      value = value.getlocal
    end
  end

  value
end
```

`packages/activemodel/src/type/date-time.ts`'s `serializeCastValue` is only the
first step — `return this.applySecondsPrecision(value)`. `type/time.ts`'s is
the same shape. The `is_utc?` normalization arm is absent from both, and
`Helpers::Timezone#is_utc?` IS ported (`type/helpers/timezone.ts`), so the
missing piece is the branch, not its input.

The likely reason it was never ported is the receiver: `DateTimeCastResult` is
a `Temporal.Instant`, which is an absolute instant carrying no zone
representation, so `getutc` / `getlocal` have nothing to switch — an Instant
always renders UTC. If that is the whole story then the arm is genuinely
unreachable and the deviation is in the choice of `Instant` as the `::Time`
port, which needs saying at the call site rather than leaving the method
looking like a truncated copy. If it is NOT the whole story — the adapters'
quoting layer reads the value's zone anywhere downstream — then
`default_timezone = :local` serializes the wrong wall-clock time and this is a
live bug.

## Acceptance criteria

- [ ] Establish whether the `is_utc?` arm is reachable given `Temporal.Instant`
      as the cast result, on both `Type::DateTime` and `Type::Time`.
- [ ] If reachable, port the branch verbatim (`time_value.rb:12-19`) with a
      test covering `default_timezone = :local`.
- [ ] If unreachable, record why at the `serializeCastValue` call site,
      naming `Instant`'s zonelessness as the reason the branch collapses.
- [ ] No test renames.
