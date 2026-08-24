---
title: "pg-type-map-timezone-option-is-inert"
status: draft
updated: 2026-08-23
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while stripping free-form comments from
`packages/activerecord/src/connection-adapters/**` (story
`strip-freeform-comments-ar-connection-adapters`). The deleted comment at
`packages/activerecord/src/connection-adapters/postgresql/type-map-init.ts:252`
read:

> TODO: activemodel Type classes don't yet honor `timezone` — these options are
> ignored until TimeType / Timestamp are extended.

`initializeInstanceTypeMap` passes `{ timezone: defaultTimezone }` into
`registerClassWithPrecision(m, "time", TimeType, ...)` and
`registerClassWithPrecision(m, "timestamp", Timestamp, ...)`
(`type-map-init.ts:253-254`), but the activemodel `TimeType` / the PG
`Timestamp` type never read the option, so the argument is inert.

Rails: `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/type_map_initializer.rb`
and `activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb`'s
`initialize_type_map`, where `OID::DateTime` / `Type::Time` are constructed with
`precision:` only — the timezone handling lives in
`ActiveRecord.default_timezone` consumers. Determine whether the `timezone:`
option is a trails invention that should be deleted, or a real gap in the
ported types that should be honored.

## Acceptance criteria

- [ ] Establish from Rails whether `timezone:` belongs on these registrations.
- [ ] Either delete the inert option (converging on Rails' argument list) or
      make `TimeType` / `Timestamp` honor it, with a test.
- [ ] No comment restating the outcome — the code or a Rails citation carries it.
