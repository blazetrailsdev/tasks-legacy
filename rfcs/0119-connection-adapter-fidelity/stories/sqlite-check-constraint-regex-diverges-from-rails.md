---
title: "sqlite3 check_constraints regex adds a quoted-name arm and caps nesting depth"
status: draft
updated: 2026-08-03
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

- Determine whether any trails path emits a quoted CHECK constraint name; if
  not, drop the quoted alternative so the regex matches Rails exactly.
- Either match Rails' unbounded nesting or pin the two-level limit with a test
  and a call-site justification.
- Tests named verbatim after the Rails check-constraint tests in
  `vendor/rails/activerecord/test/cases/migration/check_constraint_test.rb`.
