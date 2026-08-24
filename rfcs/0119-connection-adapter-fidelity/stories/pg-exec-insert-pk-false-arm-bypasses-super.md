---
title: "PG exec_insert's pk == false arm runs bespoke scaffolding instead of Rails' super"
status: closed
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Already converged on main (152b2ebe9): postgresql-adapter.ts:2450-2456 is Rails' single arm — `if (this._useInsertReturning || pk === false) return super.execInsert(...)` — citing database_statements.rb:46-47. The bespoke scaffolding on the pk === false path is gone."
---

# PG `exec_insert`'s `pk == false` arm runs bespoke scaffolding instead of Rails' `super`

## Context

Rails has ONE arm for both conditions
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/database_statements.rb:45-47`):

```ruby
def exec_insert(sql, name = nil, binds = [], pk = nil, sequence_name = nil, returning: nil)
  if use_insert_returning? || pk == false
    super
  else
```

PR #6589 (RFC 0106) restored the single `if (this._useInsertReturning || pk === false)`
guard in `packages/activerecord/src/connection-adapters/postgresql-adapter.ts`
(`execInsert`), but inside it the `pk === false` sub-case still does NOT call
`super`. It appends any `returning` columns by hand, calls `preprocessQuery`, and
runs `_instrumentedQueryOnClient` inside its own `withRawConnection` — a body
Rails does not have.

The reason recorded at the call site: trails' mixed-in `DatabaseStatements`
default routes through `executeMutation`, which auto-appends `RETURNING id` for
bare INSERTs whenever `use_insert_returning` is on
(`postgresql-adapter.ts`, the `executeMutation` RETURNING-append branch) — which
would defeat the `pk == false` opt-out — and `execQuery` cannot be used either
because it skips `materializeTransactions` / `dirtyCurrentTransaction`, so an
INSERT inside a lazy transaction would escape rollback.

That is a divergence in `executeMutation`, not a TypeScript language
shortcoming: Rails' `exec_insert` never auto-appends anything, it builds the
RETURNING clause in `sql_for_insert`
(`abstract/database_statements.rb`, `sql_for_insert`) which is gated on
`pk` being present. The auto-append is the invention; the bespoke `pk == false`
arm exists only to dodge it.

## Converged shape

- Fix the root cause: `executeMutation` must not synthesize `RETURNING id` for a
  bare INSERT. The RETURNING clause belongs to `sqlForInsert`, driven by `pk` /
  `returning`, exactly as Rails does it — so a `pk == false` call naturally
  produces no RETURNING.
- With that gone, delete the `pk === false` sub-body and let the merged arm read
  as Rails' does: a single delegation to `super` for
  `use_insert_returning? || pk == false`.
- Keep the write-path guarantees the current arm was protecting
  (materialize + dirty the current transaction) — they must come from the
  shared `super` path, not from a per-arm hand-rolled copy.

## Acceptance criteria

- [ ] `execInsert`'s `use_insert_returning? || pk == false` arm is a single
      `super` delegation, matching `postgresql/database_statements.rb:46-47`.
- [ ] `executeMutation` no longer auto-appends `RETURNING id`; the clause comes
      from `sqlForInsert`.
- [ ] A `pk: false` insert still emits no RETURNING clause, and an INSERT inside
      a lazy transaction still materializes and dirties it — both covered by
      tests that fail on the naive deletion.
- [ ] `pnpm parity:api:calls` green; no baseline widened.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
