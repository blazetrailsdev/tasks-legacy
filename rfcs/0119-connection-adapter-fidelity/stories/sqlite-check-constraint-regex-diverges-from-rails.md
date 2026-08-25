---
title: "sqlite3 check_constraints regex adds a quoted-name arm and caps nesting depth"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`SQLite3Adapter#checkConstraints`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`, ~line 2126,
converged onto `queryValue` in #5934) scans the CREATE TABLE SQL with a regex
that accepts a double-quoted constraint name:

```ts
/CONSTRAINT\s+(?:"((?:[^"]|"")*)"|(\w+))\s+CHECK\s*\((...)\)/gi;
```

Rails' `sqlite3/schema_statements.rb:101` uses a bare `\w+` for the name and no
quoted alternative:

```ruby
/CONSTRAINT\s+(?<name>\w+)\s+CHECK\s+\((?<expression>(:?[^()]|\(\g<expression>\))+)\)/i
```

Rails' `visit_CheckConstraintDefinition` emits the name unquoted, so the quoted
arm never fires on Rails-generated DDL — it only matters for DDL written by
hand or by a trails path that quotes the name. The expression sub-pattern also
differs: Rails uses a recursive subexpression (`\g<expression>`) with unbounded
nesting depth; the port hand-expands it to two levels.

Both are pre-existing (not introduced by #5934) and carry an inline note today,
but neither is verified against a Rails test.

## Acceptance criteria

- The `check_constraints` regex matches Rails' (`sqlite3_adapter.rb`): the
  quoted-name alternative is dropped once confirmed no trails path emits a
  quoted CHECK constraint name, and the nesting is unbounded rather than capped
  at the current two levels (`fn1`/`fn2`, sqlite3-adapter.ts:234-247).
- Tests named verbatim after the Rails check-constraint tests in
  `vendor/rails/activerecord/test/cases/migration/check_constraint_test.rb`.
- A test covers a CHECK expression nested deeper than two function calls and
  fails on baseline.
- If unbounded nesting is genuinely unexpressible in a JS RegExp, that is the
  language wall: `pnpm tasks block` naming it. Pinning the two-level limit with
  a call-site justification is not convergence.
