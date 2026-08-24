---
title: "sql_for_insert is async in trails because supports_insert_returning? and primary_key are; Rails' is sync"
status: draft
updated: 2026-08-21
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: ["sql-for-insert-pk-inference-binds-a-promise"]
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

Rails' `sql_for_insert` is a plain synchronous method
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:702-717`):

```ruby
def sql_for_insert(sql, pk, binds, returning) # :nodoc:
  if supports_insert_returning?
    if pk.nil?
      table_ref = extract_table_ref_from_insert_sql(sql)
      pk = primary_key(table_ref) if table_ref
    end
    returning_columns = returning || Array(pk)
    ...
```

Both predicates it calls are sync in Ruby: `supports_insert_returning?` is a
capability answer and `primary_key(table_ref)` is a schema-cache read.

In trails both are **async** — `supportsInsertReturning()` returns
`Promise<boolean>` on every adapter (`abstract-adapter.ts`,
`sqlite3-adapter.ts`, `postgresql-adapter.ts`, `abstract-mysql-adapter.ts`,
where it gates on a `databaseVersion()` probe) and `primaryKey(table)` returns a
Promise. PR #6835 therefore had to make `sqlForInsert` itself `async`
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts`),
which propagates: `exec_insert` awaits it, and any future sync caller cannot.

This was not cosmetic. Before #6835 the body read
`if (this.supportsInsertReturning?.())` — an un-awaited Promise, which is
**always truthy** — so the RETURNING clause would have been appended on every
backend the moment `exec_insert` started calling it unconditionally, including
MySQL 8, which has no `INSERT ... RETURNING`. Making the body async was the
minimum fix; it does not remove the underlying sync/async mismatch, it spreads
it.

Related and already filed: `sql-for-insert-pk-inference-binds-a-promise`
(0023-surfaced-deviations) covers the pk-inference half — #6835 fixed the
Promise-stringification defect there but did not add that story's required
baseline-failing regression test or its composite-primary-key arm, so it stays
open.

## Converged shape

Make the capability predicate synchronous so `sqlForInsert` can be sync, as
Rails is. `supports_insert_returning?` depends only on the connected server
version, which is known once the connection is established: resolve and memoize
the version at connect/configure time and let `supportsInsertReturning()` read
the memo, the way
`mysql-supports-predicates-read-database-version-field-not-reader` and
`converge-pg-supports-optimizer-hints-memo` already moved other `supports_*`
predicates onto a field.

`primaryKey(table)` is the second half: Rails reads it from the schema cache
synchronously. If the trails schema cache is warm by the time `sql_for_insert`
runs, that read can be sync too; if it cannot be, say so explicitly in the story
rather than leaving the async spelling unexplained.

## Acceptance criteria

- [ ] `supportsInsertReturning()` is synchronous on every adapter, reading a
      memoized version rather than awaiting a probe.
- [ ] `sqlForInsert` is synchronous, matching `database_statements.rb:702`.
- [ ] No call site regains a bare-truthiness test on a Promise; the MySQL 8
      no-RETURNING path stays correct (a test asserting MySQL 8 emits no
      RETURNING must fail on a baseline that drops the await).
- [ ] If `primaryKey` cannot be made sync, the story is split and the residual
      is filed with the specific blocker — not ratified in place.
