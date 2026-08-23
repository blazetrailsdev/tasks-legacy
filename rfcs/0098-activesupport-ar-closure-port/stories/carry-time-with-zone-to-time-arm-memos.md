---
title: "TimeWithZone#to_time memoizes each preserve_timezone arm"
status: done
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps:
  - converge-time-with-zone-utc-onto-a-ruby-time
deps-rfc: []
est-loc: 120
priority: 2
pr: 6937
claim: "2026-08-23T18:51:39Z"
assignee: "carry-time-with-zone-to-time-arm-memos"
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::TimeWithZone#to_time`
(`activesupport/lib/active_support/time_with_zone.rb:493-501`) memoizes each of
its three arms in its own ivar:

```ruby
def to_time
  if preserve_timezone == :zone
    @to_time_with_timezone ||= getlocal(time_zone)
  elsif preserve_timezone
    @to_time_with_instance_offset ||= getlocal(utc_offset)
  else
    @to_time_with_system_offset ||= getlocal
  end
end
```

PR #6895 ported the three-arm switch
(`packages/activesupport/src/time-with-zone.ts:551-558`) but not the memos — the
value is recomputed on every call.

This is observable, not just a cost: Rails' own tests assert identity across
two calls — `assert_equal time.object_id, @twz.to_time.object_id`
(`activesupport/test/core_ext/time_with_zone_test.rb:553`, `:566`, `:580`). The
trails mirrors of those three tests
(`packages/activesupport/src/core-ext/time-with-zone.test.ts`) carry every other
assertion in the Rails bodies but not that one, because it cannot pass today.

The memos are also what makes `freeze` (`:1177`) safe — Rails preloads the ivars
before freezing precisely because `to_time` writes one.

## Converged shape

Three separate memo fields, one per arm, matching the Ruby ivar names, written
on first read of that arm. `freeze()` preloads whichever arm the current
`preserve_timezone` selects, as it already preloads `period` / `utc` / `time` /
`toDatetime`. The three mirrored tests regain their `object_id` assertion as an
identity check (`toBe`, not `toEqual`).

## Acceptance criteria

- [ ] Each arm memoizes into its own field; a second `toTime()` under the same
      setting answers the identical object.
- [ ] `test_to_time_with_preserve_timezone_using_zone` / `_using_offset` /
      `_using_true` mirrors assert that identity; no test name touched.
- [ ] `freeze` still preloads before `Object.freeze`.
- [ ] `parity:api:calls` / `:args` clean.
