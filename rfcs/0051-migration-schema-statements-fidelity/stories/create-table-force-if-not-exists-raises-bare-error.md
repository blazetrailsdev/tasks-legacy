---
title: "create_table raises bare Error instead of ArgumentError for force + if_not_exists"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: 10
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `create_table` raises a bare `Error` for `force:` + `if_not_exists:`, Rails raises `ArgumentError`

## Context

Spotted while converging `create_table`'s `validate_table_length!` call-set row
(PR #6560, RFC 0106). That row is done; this adjacent raise was left untouched as
out of scope.

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:297-299`

```ruby
if force && options.key?(:if_not_exists)
  raise ArgumentError, "Options `:force` and `:if_not_exists` cannot be used simultaneously."
end
```

trails
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts`,
`createTable`) raises the base `Error` with the same message:

```ts
if (force && options.ifNotExists) {
  throw new Error("Options `:force` and `:if_not_exists` cannot be used simultaneously.");
}
```

CLAUDE.md is explicit that a port keeps the **same error class, same message
string, same raise site** — the class is the half that diverges here, and a
caller rescuing `ArgumentError` (as Rails' own tests do) would not catch this.

There is a second, smaller divergence on the same line: Rails tests
`options.key?(:if_not_exists)` — key presence — while trails tests the value's
truthiness, so `if_not_exists: false` raises in Rails and passes in trails.

## Converged shape

```ts
if (force && "ifNotExists" in options) {
  throw new ArgumentError("Options `:force` and `:if_not_exists` cannot be used simultaneously.");
}
```

`ArgumentError` is already imported in this file.

## Acceptance criteria

- [ ] The raise uses `ArgumentError` with the message unchanged, at the same
      point in the body (schema_statements.rb:297-299).
- [ ] The guard tests key presence, not value truthiness, so
      `ifNotExists: false` with `force:` raises as it does in Rails.
- [ ] A regression test that fails on the current baseline.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
