---
title: "ToSql#quoteTableName/quoteColumnName route the name through an invented rubyToS helper"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

`Arel::Visitors::ToSql#quote_table_name` and `#quote_column_name` pass `name`
straight to the connection after a SqlLiteral pass-through guard. trails routes
it through a local `rubyToS()` helper first.

Rails (`activerecord/lib/arel/visitors/to_sql.rb`):

```ruby
def quote_table_name(name)
  return name if Arel::Nodes::SqlLiteral === name
  @connection.quote_table_name(name)
end
```

trails today (`packages/arel/src/visitors/to-sql.ts:1673,1679`):

```ts
return this.connection.quoteTableName(rubyToS(name));
...
return this.connection.quoteColumnName(rubyToS(name));
```

`rubyToS` is a module-local function at `to-sql.ts:116` with no Rails
counterpart. Surfaced by the RFC 0096 arel naming burndown (PR #6350) as two
`naming` rows where Rails passes `name` and trails passes `rubyToS`; deliberately
NOT renamed there, because the local is an invented conversion and renaming it
would have papered over an extra call.

Establish what `rubyToS` is actually absorbing before deleting it — the callers
may be handing in a non-string that Ruby would have stringified implicitly at a
DIFFERENT point (Ruby's `quote_table_name` implementations call `to_s`
themselves in some adapters). If the conversion is load-bearing, it belongs
inside the adapter's `quoteTableName` where Ruby does it, not at the Arel call
site.

## Converged shape

Delete the `rubyToS` wrapper from both call sites and pass `name`, relocating any
genuinely required stringification into whichever adapter method Ruby performs it
in. If `rubyToS` turns out to have no remaining callers, remove it — it is
unreferenced invented surface.

## Acceptance criteria

1. `quoteTableName` / `quoteColumnName` in `visitors/to-sql.ts` pass `name` as
   Rails does.
2. `rubyToS` is deleted, or its remaining callers and their Rails justification
   are documented at the call site.
3. The two `naming` rows for `visitors/to-sql.ts` `quote_table_name` /
   `quote_column_name` in `pnpm parity:api:calls:args:report` are gone; report
   before/after.
4. `pnpm vitest run packages/arel` green, and the activerecord adapter suites
   that render table/column names stay green on all three lanes.

## Absorbed: `arel-visitor-to-s-belongs-in-adapter-quoting`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "arel-visitor-to-s-belongs-in-adapter-quoting"

### Context

Surfaced by `converge-to-sql-visitor-call-arguments` (RFC 0099), which converged
nine of the ten `naming` rows on `packages/arel/src/visitors/to-sql.ts` and left
these two:

```text
quote_table_name  -> quote_table_name   ruby: [ref:name]  ts: [call:rubyToS]
quote_column_name -> quote_column_name  ruby: [ref:name]  ts: [call:rubyToS]
```

Rails (`vendor/rails/activerecord/lib/arel/visitors/to_sql.rb:872-878`) passes
the name straight through:

```ruby
def quote_table_name(name)
  return name if Arel::Nodes::SqlLiteral === name
  @connection.quote_table_name(name)
end
```

The `to_s` Rails applies lives one layer down, in EVERY adapter's quoting
module — `quote_table_name(name)` is `"#{name.to_s}"`-shaped at
`sqlite3/quoting.rb:45`, `mysql/quoting.rb:47`, `postgresql/quoting.rb:47`.

trails hoists it into the visitor (`rubyToS(name)`, to-sql.ts:116) because the
`Connection` quoting surface is typed `quoteTableName(name: string)`, so the
visitor is the last place a non-String name (a Symbol, or the Array an
Attribute carries on the composite-primary-key default-order path) can be
coerced. That is a real behavioural dependency, not dead code — `rubyToS` also
reproduces Ruby's inspect-style `Array#to_s`, which JS's comma join does not —
so it was NOT absorbed into the converging PR.

### Acceptance criteria

- [ ] `quote_table_name` / `quote_column_name` in `to-sql.ts` pass `name`
      through, as Rails does.
- [ ] The `to_s` moves to where Rails has it: each adapter's `quoteTableName` /
      `quoteColumnName` (sqlite3, mysql, postgresql, and the abstract quoting
      interface), whose parameter widens from `string` to what Rails accepts.
- [ ] The Array arm keeps rendering inspect-style, with a test covering the
      composite-primary-key `table[primaryKey].desc` path that motivated it.
- [ ] The two `naming` rows leave `pnpm parity:api:calls:args:report`; arel and
      adapter quoting tests green on all three adapters.

## Triage note (2026-08-25): the helper has been renamed, the divergence has not

This story calls the helper `rubyToS`. It is now spelled **`toS`** — grep for
`rubyToS` in `packages/arel/src` returns nothing. The divergence itself is
unchanged and was re-verified live: `visitors/to-sql.ts:1484-1490`
(`quoteTableName`) and its `quoteColumnName` twin still route `name` through
`toS(...)` before handing it to the connection, where `to_sql.rb:872-878` passes
`name` straight through after the SqlLiteral guard.

The call-site comment now states the reason explicitly — trails reaches here
with an Array name for a composite primary key, which Ruby renders `["a", "b"]`
rather than `a,b` — which is exactly the load-bearing behaviour the absorbed
`arel-visitor-to-s-belongs-in-adapter-quoting` section says must move DOWN into
each adapter's quoting module, not be deleted. Read `name`'s parameter type
(`string | Nodes.SqlLiteral`) before starting: widening the adapter side is the
prerequisite.
