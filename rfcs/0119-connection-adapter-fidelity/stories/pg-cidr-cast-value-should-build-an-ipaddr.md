---
title: "pg-cidr-cast-value-should-build-an-ipaddr"
status: ready
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
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

Rails `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/cidr.rb`
`cast_value` builds an `IPAddr.new(...)` and returns that object; trails
`packages/activerecord/src/connection-adapters/postgresql/oid/cidr.ts`
`castValue` returns the normalized CIDR **string** instead, because Ruby's
`IPAddr` has no port in trails.

Surfaced by the RFC 0106 call-set gate (the `new` row on `oid/cidr.json`, now a
CONVERGEABLE `@missingRailsCall` tag at the call site). Consumers that expect an
address object — `==` between a `cidr` and an `inet` value, prefix arithmetic,
`serialize` round-tripping — see a string.

## Acceptance criteria

- [ ] `castValue` returns an address value object with the behaviour Rails' `IPAddr` gives the `cidr`/`inet` types (equality, prefix, `to_s`), not a bare string.
- [ ] The `@missingRailsCall new` tag on `oid/cidr.ts` is deleted, not re-justified.
- [ ] Rails' `oid/cidr` and `oid/inet` test cases pass on the PostgreSQL lane.
