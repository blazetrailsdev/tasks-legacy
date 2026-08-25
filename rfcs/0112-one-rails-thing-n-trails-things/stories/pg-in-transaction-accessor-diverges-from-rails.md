---
title: "PG inTransaction accessor is materialized-only where Rails' in_transaction? counts lazy frames"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 60
pr: 7049
claim: "2026-08-25T16:58:47Z"
assignee: "converge-duplicate-url-options-and-url-for"
blocked-by: null
closed-reason: null
---

## Context

Rails' `PostgreSQLAdapter#in_transaction?` is
`open_transactions > 0`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:908-910`),
so an open **lazy** (un-materialized) frame counts — a read-only
`transaction { record.reload }` never emits a physical `BEGIN`, yet
`in_transaction?` is true.

trails has a same-named accessor with different semantics:
`get inTransaction()` in `connection-adapters/postgresql-adapter.ts` returns the
private `_inTransaction` flag, which is set only when a transaction is actually
materialized. The two disagree exactly on lazy frames.

Because of that, the cached-plan arm of `performQuery`
(`connection-adapters/postgresql/database-statements.ts`, landed in PR #6330)
cannot call the Rails-named accessor and spells Rails' predicate out inline as
`this.openTransactions > 0`, with the citation, to keep Rails' semantics. So the
file now carries Rails' meaning under no name while a differently-behaved
`inTransaction` sits on the adapter — a name collision that will mislead the
next porter.

Readers of the accessor today: `transactions.ts:580`, `adapter.test.ts:117`,
`transactions.trails.test.ts`, `test-fixtures/with-transactional-fixtures.test.ts`.

## Converged shape

- Port `in_transaction?` under its Rails name with Rails' body
  (`openTransactions > 0`) and have `performQuery`'s rescue call it instead of
  inlining the comparison.
- Decide what happens to the existing materialized-only flag: either rename it
  to something that says what it means (it is a physical-BEGIN marker, not
  Rails' predicate) and repoint its four readers, or fold those readers onto
  the Rails predicate where the lazy-frame difference does not matter. Check
  each reader — `transactions.ts:580`'s `hadOuterTransaction` in particular —
  before flipping its semantics.

## Acceptance criteria

- [ ] `inTransaction` answers Rails' `open_transactions > 0`, or the
      materialized-only flag no longer occupies the Rails name.
- [ ] `performQuery`'s cached-plan arm calls the ported predicate rather than
      inlining `openTransactions > 0`.
- [ ] The four existing readers keep their intended behavior (verify
      `with-transactional-fixtures.test.ts` and `transactions.trails.test.ts`).
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:extra --package activerecord` clean.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
