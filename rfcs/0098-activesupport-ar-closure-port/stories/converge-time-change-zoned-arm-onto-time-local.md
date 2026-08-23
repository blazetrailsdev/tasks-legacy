---
title: "Time#change's zoned arm rebuilds through Time.local with isdst"
status: claimed
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: 5
pr: null
claim: "2026-08-23T17:32:06Z"
assignee: "converge-time-with-zone-utc-onto-a-ruby-time"
blocked-by: null
closed-reason: null
---

## Context

PR #6895 added a `::Time` overload to `change`
(`packages/activesupport/src/time-ext.ts`), the port of
`Time#change` (`activesupport/lib/active_support/core_ext/time/calculations.rb:123-177`).
It reseats the changed components through `Time`'s constructor, whose zone
argument is a `utc_offset` and not a zone id, so a receiver that carried a
NAMED zone comes back carrying only its offset — `zone` goes from `"EST"` to
`null`.

Rails' own zoned arm is

```ruby
elsif zone
  ::Time.local(new_sec, new_min, new_hour, new_day, new_month, new_year, nil, nil, isdst, nil)
```

(`calculations.rb:172-175`) — `::Time.local` in Ruby's reversed component order,
with the receiver's `isdst` passed through so a repeated wall clock after a DST
fall-back resolves to the receiver's occurrence rather than the first one.

`@blazetrails/date`'s `Time.local` takes the forward component order and no
`isdst` (`packages/date/src/time.ts`), so trails cannot express that argument
today; the current overload takes the offset path instead, which is correct on
the instant but wrong on `zone` and on the DST-repeat disambiguation.

## Converged shape

`Time.local` grows Ruby's `isdst` argument (MRI `time_s_mktime` / `time.c`
`time_utc_or_local`), and `change`'s zoned arm calls it, so a zoned receiver
answers a zoned `::Time` whose `zone` and DST occurrence both match Rails.
Whether trails' `Time.local` also takes the reversed positional order Rails
passes is part of the story: `parity:api:calls:args` compares argument shape,
so the two have to be settled together.

Rails' third arm — `zone.respond_to?(:utc_to_local)` (`:148-171`), the TZInfo
zone-OBJECT receiver and its second-occurrence correction — stays unreachable
while `Time` carries a zone id string rather than a zone object; price it here
and either port it or record why it cannot be reached.

## Acceptance criteria

- [ ] `change` over a `::Time` with a named zone answers a `::Time` whose
      `zone` is that zone's abbreviation, not `null`.
- [ ] The DST fall-back repeat resolves to the receiver's occurrence.
- [ ] `Time.local`'s argument shape and `change`'s call site agree with Rails
      under `parity:api:calls:args`.
- [ ] The `zone.respond_to?(:utc_to_local)` arm is ported or its
      unreachability recorded with a citation.
