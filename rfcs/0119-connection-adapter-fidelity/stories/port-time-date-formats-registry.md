---
title: "Port the Time::DATE_FORMATS / Date::DATE_FORMATS registry and route TimeWithZone#to_fs through it"
status: draft
updated: 2026-08-15
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Port the `Time::DATE_FORMATS` / `Date::DATE_FORMATS` registry

## Context

`ActiveSupport::TimeWithZone#to_fs`
(`vendor/rails/activesupport/lib/active_support/time_with_zone.rb:212-220`)
looks the format up in the `Time::DATE_FORMATS` hash
(`vendor/rails/activesupport/lib/active_support/core_ext/time/conversions.rb:8-20`)
and either calls the entry (when it responds to `call`) or passes it to
`strftime`:

```ruby
elsif formatter = ::Time::DATE_FORMATS[format]
  formatter.respond_to?(:call) ? formatter.call(self).to_s : strftime(formatter)
```

trails has no `DATE_FORMATS` registry at all. `TimeWithZone#toFs`
(`packages/activesupport/src/time-with-zone.ts:488`) instead switches on the
format name and inlines each entry's format string, so
`to_fs -> strftime(ref:formatter)` is a permanent call-argument divergence
(baselined in
`scripts/api-compare/call-mismatches-exclude/activesupport/time-with-zone.json`
with that reason). The same gap is noted at
`packages/activerecord/src/core.test.ts:74-77` and
`packages/activerecord/src/connection-adapters/abstract/sql-datetime.ts:30`.

Because the registry is missing, applications cannot register their own
format (`Time::DATE_FORMATS[:my_format] = "%b %e"`), which is public Rails
API.

## Acceptance criteria

- [ ] `DATE_FORMATS` exists for Time and Date with Rails' default entries and
      is mutable at runtime.
- [ ] `TimeWithZone#toFs` looks the formatter up in the registry and forwards
      it to `strftime`, including the callable arm.
- [ ] The `time-with-zone.ts to_fs strftime` args row is deleted from
      `scripts/api-compare/call-mismatches-exclude/activesupport/time-with-zone.json`.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green.
