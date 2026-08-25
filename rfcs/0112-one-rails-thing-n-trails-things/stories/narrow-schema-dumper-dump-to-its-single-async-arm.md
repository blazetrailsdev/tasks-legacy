---
title: "SchemaDumper.dump advertises a sync arm Rails' dump does not have"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `narrow-schema-source-to-its-single-async-arm`
(PR #7060), which collapsed every `SchemaSource` member and every dumper hook
onto the single `Promise<...>` arm Rails' `@connection` presents.

One `T | Promise<T>` union survived, on `SchemaDumper.dump` itself
(`packages/activerecord/src/schema-dumper.ts`):

```ts
static dump(
  pool: ConnectionPoolLike | SchemaSource | DatabaseAdapter = baseClass().connectionPool(),
  stream: string[] = [],
  config: SchemaDumperConfig = baseClass(),
): string[] | Promise<string[]> {
```

Rails' `SchemaDumper.dump` (`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:43-48`)
has one return:

```ruby
def self.dump(pool = ActiveRecord::Base.connection_pool, stream = STDOUT, config = ActiveRecord::Base)
  connection = pool.schema_cache.connection
  ...
  new(pool, generate_options(config)).dump(stream)
  stream
end
```

It was left out of #7060 deliberately — that story's acceptance criteria named
the `SchemaSource` members and the four hooks, and `dump` has its own callers to
sweep. Every internal entry point is now `await`-driven (#7051 collapsed
`#tables` and `#dump` onto Rails' single bodies), so the sync arm here is the
same type-level invitation to reintroduce a sync branch that the interface half
just shed.

## Converged shape

Narrow `SchemaDumper.dump`'s return to `Promise<string[]>` and update the call
sites that still consume it as a possibly-sync value. Check the subclass
overrides (`connection-adapters/abstract/schema-dumper.ts` and the
postgresql/mysql ones) declare the same single arm.

Rails cite: `schema_dumper.rb:43-48`, `:113` (`@connection`), `:134-158`.

## Acceptance criteria

- [ ] `SchemaDumper.dump` declares `Promise<string[]>`; no `T | Promise<T>`
      union remains anywhere in `schema-dumper.ts`.
- [ ] No `instanceof Promise` fork is introduced at any call site.
- [ ] `schema-dumper.test.ts`, `schema-dumper.trails.test.ts`, the three
      adapter dumper test files and `adapters/sqlite3/virtual-table.test.ts`
      stay green.
