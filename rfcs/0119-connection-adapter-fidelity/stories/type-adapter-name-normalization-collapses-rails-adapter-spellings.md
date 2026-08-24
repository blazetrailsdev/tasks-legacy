---
title: "adapterNameFromConfig collapses Rails adapter spellings into three families"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while working #5304 (PG type-map-init modifier/register source order).
The moved call site reads `{ adapter: "postgres" }` where Rails writes
`adapter: :postgresql` (`postgresql_adapter.rb:1166-1167`).

Rails' `ActiveRecord::Type.adapter_name_from`
(`vendor/rails/activerecord/lib/active_record/type.rb:49-51`) is
`model.connection_db_config.adapter.to_sym` — the configured adapter name
verbatim, no normalization. Adapter-specific registrations are therefore keyed
by the exact adapter symbol.

trails routes every adapter name through `adapterNameFromConfig`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts:127-140`),
which collapses names into three families:

- `postgresql` / `postgres` / `pg` -> `"postgres"`
- `mysql` / `mysql2` / `mariadb` -> `"mysql"`
- everything else -> `"sqlite"` (silent default arm)

Two consequences worth deciding on deliberately:

1. A caller following Rails docs and registering a type with
   `adapter: "postgresql"` (or `"mysql2"`) never matches, because
   registrations are stored under the collapsed family name while
   `lookup` normalizes the current adapter to the same family. Any code
   that registers with the Rails-spelled name is silently dead.
2. The `default:` arm maps an _unknown_ adapter to `"sqlite"` rather than
   raising or passing the name through, so a typo'd or third-party adapter
   silently inherits SQLite type registrations.

Note this is distinct from the existing draft story
`type-adapter-name-from-swallows-unconfigured-instead-of-raising`, which
covers the `ConnectionNotEstablished` catch in `adapterNameFrom`
(`type.ts:151-163`). This story is about the normalization function itself.

## Acceptance criteria

- Decide and record whether the family collapse is a deliberate trails
  deviation or a bug; if deliberate, justify it at the `adapterNameFromConfig`
  call site (per the justify-deviations-at-call-site convention).
- Rails-spelled adapter names (`postgresql`, `mysql2`) either resolve
  identically to the family name or are documented as unsupported spellings.
- The unknown-adapter `default: -> "sqlite"` arm either raises, passes the
  name through, or carries a comment stating why silently defaulting is
  correct.
- Existing adapter-specific type registration tests still pass on all three
  adapters.
