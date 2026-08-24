---
title: "Drop the invented :variables config validation from PostgreSQLAdapter"
status: draft
updated: 2026-08-22
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

`PostgreSQLAdapter`'s constructor
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`, the
`variables` block in the object-config branch) validates the `:variables`
config hash and raises for shapes Rails accepts without complaint:

- `throw new TypeError("variables must be a plain object")` for a non-plain
  object or a non-`Object.prototype` prototype.
- `throw new Error("Invalid PostgreSQL session variable name: ...")` for a key
  failing `/^[a-zA-Z_][a-zA-Z0-9_.]*$/`.
- `throw new TypeError("variables[...] must be string | number | boolean | null")`.

Rails has no such validation anywhere on this path:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:977`
  — `variables = @config.fetch(:variables, {}).stringify_keys`
- `postgresql_adapter.rb:982-988` — the loop `SET SESSION #{k} TO #{quote(v)}`
  interpolates the key directly and quotes only the value. Whatever is in the
  hash goes to the server; a bad key is the server's error, not the adapter's.

The validation was the guard that made the (now removed) frozen
`_sessionVariables` field safe to interpolate. Story
`converge-pg-configure-connection-variables-fetch` (PR #6874) removed the
freeze and moved to Rails' per-call `@config.fetch`, so the validation is now
the only remaining piece of invented surface in this cluster — three error
classes and message strings with no Rails counterpart, raised at a different
site (construction) than any Rails raise.

Two call sites in `packages/activerecord/src/adapters/postgresql/connection.test.ts`
(around the `Invalid PostgreSQL session variable name` and
`variables must be a plain object` assertions) are trails-only tests that exist
solely to cover the invented behaviour.

## Acceptance criteria

- [ ] The constructor no longer raises for `:variables` shapes Rails accepts —
      the hash is passed through to the `SET SESSION` loop as Rails does
      (postgresql_adapter.rb:982-988).
- [ ] The three invented error classes / message strings are deleted, not
      reworded or relocated.
- [ ] The two trails-only tests asserting those messages are removed or
      retargeted at the server-side error Rails would surface.
- [ ] If any validation must survive, it is justified at the call site with a
      Rails cite and a genuine TypeScript language shortcoming — note that key
      interpolation is identical in Rails, so "SQL injection" is not a
      trails-specific concern here.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
