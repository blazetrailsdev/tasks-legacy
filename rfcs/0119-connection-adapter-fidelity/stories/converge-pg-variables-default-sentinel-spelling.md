---
title: 'Accept Rails'' ":default" sentinel spelling for PG session variables'
status: draft
updated: 2026-08-22
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQLAdapter#configureConnection`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`, the
`:variables` loop in `_maybeConfigureConnection`) tests the sentinel value as
`val === "default"`. Rails tests two spellings:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:983`
  — `if v == ":default" || v == :default`

A Ruby Symbol value is a colon-prefixed string in trails (CLAUDE.md, "A Ruby
Symbol is a JS string"), so the ported guard should be
`val === ":default" || val === "default"`-shaped only if `"default"` is what
Rails' plain-string arm accepts — it is not. Rails accepts `":default"` (the
YAML/`database.yml` spelling) and the Symbol; it does NOT accept the bare
string `"default"`. trails accepts exactly the one spelling Rails rejects.

Surfaced while converging the `@config.fetch(:variables, {})` reads
(story `converge-pg-configure-connection-variables-fetch`, PR #6874), which
touched the adjacent lines.

## Acceptance criteria

- [ ] The sentinel guard matches postgresql_adapter.rb:983 — the `":default"`
      spelling is accepted (and the Symbol arm, per the repo's Symbol
      convention).
- [ ] The `PostgreSQLAdapterOptions` type for `variables` reflects the accepted
      spelling.
- [ ] Existing tests that pass `"default"` are updated to the Rails spelling,
      or the bare string is kept only if a Rails cite supports it.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
