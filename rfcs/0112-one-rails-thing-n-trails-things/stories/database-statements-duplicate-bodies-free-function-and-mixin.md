---
title: "abstract/database-statements.ts defines each method twice (free function + mixin object); only the mixin copy is live and they drift"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 250
pr: 6772
claim: "2026-08-20T01:54:44Z"
assignee: "handle-warnings-body-belongs-on-abstract-mysql-adapter"
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/abstract/database-statements.ts`
defines every DatabaseStatements method TWICE:

- a file-level `export function <name>(this: DatabaseStatementsHost, ...)`
  (the "standalone / utility surface", ~line 583 for `execQuery`), and
- a body of the same name inside the `export const DatabaseStatements = { ... }`
  module object (~line 1723 for `execQuery`), which is what
  `include(AbstractAdapter, DatabaseStatements)` actually installs on the
  prototype.

Only the module-object body is live. The free function is dead for adapters,
and the two copies drift: before PR #6548 the free `execQuery` delegated to
`internalExecQuery` while the live mixin body called `this.execute` +
`Result.fromRowHashes` — a silent divergence from
`activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:147-149`
that survived only because all three concrete adapters overrode `execQuery`.
PR #6548 fixed both copies of `execQuery`; the duplication itself remains for
the rest of the module.

Rails has ONE definition per method inside `module DatabaseStatements`, so the
second copy is trails-invented surface.

## Converged shape

One definition per Rails method in this file. Either the module object holds
the bodies and the file-level `export function`s are deleted (updating the
handful of direct importers to call through the host), or the module object is
assembled from the file-level functions by reference
(`export const DatabaseStatements = { execQuery, execInsert, ... }`) so the two
can never drift.

## Acceptance criteria

1. No DatabaseStatements method has two independent bodies in
   `abstract/database-statements.ts`.
2. Each surviving body mirrors its Ruby counterpart in
   `abstract/database_statements.rb`.
3. `parity:api:calls` / `parity:api:calls:args` non-negative;
   `parity:api:extra --package activerecord` does not grow.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
