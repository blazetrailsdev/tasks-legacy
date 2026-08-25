---
title: "retire-base-schema-dumper-create-factory"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' base `SchemaDumper` publishes no factory: `private_class_method :new`
(`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:11`), and
`create(connection, options)` exists only on the adapter subclass
(`connection_adapters/abstract/schema_dumper.rb:8-10`). Every Rails construction
goes through `connection.create_schema_dumper` (`schema_dumper.rb:44-49`).

PR #6140 made the base `abstract` and moved the real `create` onto the subclass,
but kept a `protected static create` on the base
(`packages/activerecord/src/schema-dumper.ts`) with a cast through a concrete
constructor type, because the base's `dump` / `dumpTableSchema` overloads
construct from a bare `SchemaSource` that carries no adapter and therefore no
`createSchemaDumper`. Those `SchemaSource` overloads are themselves trails
additions — Rails' `dump` takes a pool.

## Converged shape

The base has no `create`. `SchemaDumper.dump` reaches the dumper the way Rails
does, through `connection.create_schema_dumper(generate_options(config))`, and
the bare-`SchemaSource` overloads either route through the adapter layer
explicitly or are retired with RFC 0056's bare-base path.

## Acceptance criteria

- [ ] `protected static create` is gone from `packages/activerecord/src/schema-dumper.ts`.
- [ ] `dump` / `dumpTableSchema` construct via the adapter layer only.
- [ ] `pnpm parity:api:extra --package activerecord` shows no novel name added, and the
      dumper suites stay green.
