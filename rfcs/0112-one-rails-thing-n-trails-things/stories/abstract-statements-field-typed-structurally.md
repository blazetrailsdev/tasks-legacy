---
title: "AbstractAdapter._statements is a duck-type where Rails names ConnectionAdapters::StatementPool"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
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

Rails' `@statements` is a `ConnectionAdapters::StatementPool` — one class, with
one adapter subclass each:

```ruby
# activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:156
@statements = build_statement_pool
```

trails already mirrors that hierarchy: `StatementPool<T>` in
`packages/activerecord/src/connection-adapters/statement-pool.ts` carries the
`Mirrors: ActiveRecord::ConnectionAdapters::StatementPool` docline, and all
three adapter pools descend from it —
`postgresql-adapter.ts` (`StatementPool extends GenericStatementPool<PreparedStatement>`),
`sqlite3-adapter.ts` (`StatementPool extends GenericStatementPool<SqliteStatement>`),
and `abstract-mysql-adapter.ts` → `mysql2-adapter.ts`
(`Mysql2StatementPool extends MysqlStatementPool`).

But the field PR #7050 added to the base adapter is typed **structurally**, not
as that class:

```ts
// packages/activerecord/src/connection-adapters/abstract-adapter.ts
_statements?: { reset(): void; clear(): void | Promise<void> } | null;
```

So the one place Rails names the type is the one place trails does not. The
same duck-type is repeated a third time in mysql2's `PerformQueryHost`
(`connection-adapters/mysql2/database-statements.ts`), which declares
`_statements?: { delete(key: string): unknown } | null` for the
`@statements.delete(sql)` at `mysql2/database_statements.rb:70`.

This is expedience, not a language shortcoming: `statement-pool.ts` is a leaf
module with no runtime imports, so `abstract-adapter.ts` can import the class
without closing a cycle.

## Converged shape

Type the base field as the shared class and let the adapters narrow it:

```ts
import { StatementPool } from "./statement-pool.js";
// ...
_statements?: StatementPool | null;
```

`PerformQueryHost`'s member follows. Check the variance actually holds first —
the reason a structural type was reached for is that mysql2's pool is
`Mysql2StatementPool | null` while sqlite3's is non-nullable and PG's is
`declare`d, and `clear()`'s return type differs across the three
(`void` vs `Promise<void>`); if the base class's own signature does not already
cover all three, widen it there rather than re-introducing a duck-type at the
field.

## Acceptance criteria

- [ ] `AbstractAdapter._statements` is typed as the shared `StatementPool`
      class, not a structural shape.
- [ ] mysql2's `PerformQueryHost._statements` likewise.
- [ ] No adapter re-declares the field with an incompatible shape.
- [ ] `pnpm typecheck` clean; all three adapter lanes green.
