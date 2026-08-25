---
title: "TableDefinition smuggles conn through the options bag and carries a dead adapterName"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `TableDefinition#initialize` takes the connection as its **first
positional parameter** and has no `adapter_name` at all
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:371-393`):

```ruby
def initialize(
  conn,
  name,
  temporary: false,
  if_not_exists: false,
  options: nil,
  as: nil,
  comment: nil,
  **
)
  @conn = conn
  ...
end
```

Every `create_table_definition` in Rails matches that shape —
`TableDefinition.new(self, name, **options)`
(`abstract/schema_statements.rb:1705`), `MySQL::TableDefinition.new(self, name,
**options)` (`mysql/schema_statements.rb:172-174`).

trails instead smuggles the connection into the **options bag** as an `adapter`
key, alongside an invented `adapterName`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:1071-1093`):

```ts
constructor(
  name: string,
  tdOptions: {
    adapterName?: "sqlite" | "postgres" | "mysql2";
    adapter: TableDefinitionConn;
    temporary?: boolean;
    ...
  },
)
```

Two consequences:

1. **`_adapterName` is dead.** It is `private`, assigned once at :1086
   (`tdOptions.adapterName ?? "sqlite"`) and read nowhere in the repo — `grep -rn
"_adapterName" packages/activerecord/src --include=*.ts` returns only the
   declaration, the assignment, and an unrelated `Migration` getter. PR #7019
   removed the last producer that fed it (`buildCreateTableDefinition` used to
   inject `adapterName: this.adapterName` into `ctdOptions`); the abstract
   `createTableDefinition` still sets it, but nothing consumes it.
2. **The connection travels as an option**, so every subclass ctor and every
   `createTableDefinition` override has to thread `adapter: this` through a hash
   Rails passes positionally, and callers that build a plain options object have
   to remember to add it — a footgun PR #7019 hit in
   `schema-statements-privates.trails.test.ts`, where three stubs constructed
   `new TableDefinition(name, options)` and blew up on
   `this._adapter.validColumnDefinitionOptions()` once the caller stopped
   injecting `adapter`.

Note the field is genuinely adapter-dependent surface of the kind
[[project-column-primarykey-flag-is-adapter-dependent]] warns about; this story
is the ctor shape, not the `columns`/`primary_key` dispatch that
`columns-primary-key-dispatch-on-adaptername` covers.

## Converged shape

`TableDefinition`'s constructor takes `conn` first and `name` second, with the
remaining Rails kwargs in the options object:

```ts
constructor(conn: TableDefinitionConn, name: string, tdOptions: { ... } = {})
```

`adapterName` and the private `_adapterName` field are deleted outright — no
Rails counterpart, no reader. The three `createTableDefinition` overrides
(abstract, PostgreSQL, SQLite3, MySQL) pass `this` positionally, mirroring
`TableDefinition.new(self, name, **options)`.

## Acceptance criteria

- `TableDefinition` (and the PostgreSQL / MySQL / SQLite3 subclasses) take the
  connection as the first positional constructor argument, `name` second.
- The `adapterName` option and the `_adapterName` field are gone; nothing in
  `packages/activerecord/src` references either.
- Every `createTableDefinition` and every direct `new *TableDefinition(...)`
  call site — tests included — moves to the positional form.
- `parity:api` activerecord matched-method count does not decrease; novel count
  does not increase; `parity:api:calls` / `:calls:args` non-negative.
- Green on sqlite3, PostgreSQL and MySQL.
