---
title: "legacy-point-serialize-should-extract-number-for-point"
status: draft
updated: 2026-08-22
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/oid/legacy_point.rb`
extracts a private `number_for_point` that strips a trailing `.0` from a
coordinate, and `serialize` calls it once per coordinate. trails
`packages/activerecord/src/connection-adapters/postgresql/oid/legacy-point.ts`
applies the same rule **inline** in `serialize`, so the helper does not exist.

CLAUDE.md: "If Rails extracts a private helper, extract it, with the Rails
name." The current baseline reason argues the helper would be surface with a
single caller — that is a preference, not a language shortcoming, and the same
argument would delete every private Rails helper in the port.

Surfaced by the RFC 0106 call-set gate (the `number_for_point` row, now a
CONVERGEABLE `@missingRailsCall` tag at the call site).

## Acceptance criteria

- [ ] A private `numberForPoint` exists in `legacy-point.ts` mirroring the Ruby body, and `serialize` calls it at each of Rails' call sites.
- [ ] The `@missingRailsCall number_for_point` tag is deleted, not re-justified.
- [ ] Rails' legacy point test cases pass on the PostgreSQL lane.
