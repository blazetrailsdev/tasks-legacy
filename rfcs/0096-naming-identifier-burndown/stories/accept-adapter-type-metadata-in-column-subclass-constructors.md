---
title: "PG/MySQL Column subclasses rebuild the TypeMetadata their adapter already built"
status: ready
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' adapter `Column` subclasses take the `TypeMetadata` their adapter's
`fetch_type_metadata` built and hand it straight to `super` — there is no
Column-side reconstruction at all
(`activerecord/lib/active_record/connection_adapters/postgresql/column.rb:9-14`,
`.../mysql/column.rb`, whose `Column` declares no `initialize` at all and takes
the base's, `.../abstract/column.rb:14-24`).

trails' subclass constructors instead take a flat plain object and rebuild the
metadata from it:

- `packages/activerecord/src/connection-adapters/postgresql/column.ts:20-48` —
  `sqlTypeMetadata: { sqlType?, type?, oid?, fmod?, limit?, precision?, scale? }`,
  then `const meta = new TypeMetadata({...}, { oid, fmod })`.
- `packages/activerecord/src/connection-adapters/mysql/column.ts:18-45` — same
  shape with `extra` on the flat object.

The rebuild is measurable waste on the reflection path: PG's
`newColumnFromField` (`postgresql-adapter.ts`, `fetchTypeMetadata`) and MySQL's
`newColumnFromField` (`mysql/schema-statements.ts:391`) each construct a real
`TypeMetadata`, pass it in as a structural object, and the Column throws it away
and builds an equal one. It also means the dedup pool never sees the instance
the adapter made, and a caller cannot pass a metadata subclass the Column does
not already know how to rebuild.

Surfaced while landing `converge-adapter-type-metadata-coder-round-trip`
(PR #7026), which made the subclasses real `SqlTypeMetadata` subclasses — the
prerequisite for accepting one directly.

## Converged shape

`PostgreSQL::Column` / `MySQL::Column` accept a `SqlTypeMetadata` (their own
subclass, in practice) and pass it to `super` unchanged, as Rails does. The
adapter's `fetchTypeMetadata` result becomes the Column's `sqlTypeMetadata`
identity. Keep the plain-object arm only if a test caller genuinely needs it,
and if so state that at the call site.

## Acceptance criteria

- [ ] Neither subclass constructor reconstructs a `TypeMetadata` from a flat
      object; the metadata the adapter built is the one the Column holds.
- [ ] `oid` / `fmod` / `extra` are still read through the delegation added in
      PR #7026 (`postgresql/column.rb:7`, `mysql/column.rb:7`).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
