---
title: "createTable bundles Rails' named kwargs and **options into one object, forcing a non-Rails local"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `schema-statements.ts` for RFC 0096 in PR #6356. It is
the one `naming` row in that file that could not be renamed, because the blocker
is a SIGNATURE divergence, not a spelling.

`schema_statements.rb:293` declares

```ruby
def create_table(table_name, id: :primary_key, primary_key: nil, force: nil, **options, &block)
  validate_create_table_options!(options)
```

so `options` is the rest-kwargs with `id` / `primary_key` / `force` already
split out, and `:294` passes exactly that hash on.

`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:349`
collapses all of it into ONE `optionsOrFn` object, so the body has to
re-destructure the rest and cannot also call it `options` — it is named
`validatedOptions`, and the call reads
`validateCreateTableOptionsBang(validatedOptions)` where Rails reads
`validate_create_table_options!(options)`. `create_join_table` (`:389`) has the
same shape and names its bundle `opts`.

Related but distinct from `call-args-ar-kwargs-vs-positional`, which covers
kwarg-vs-positional at CALL sites; this one is the CALLEE signature that forces
those locals.

## Converged shape

`createTable(tableName, { id, primaryKey, force, ...options }, block)` — or
whatever the settled trails kwargs idiom is for a Ruby `**rest` alongside named
kwargs — so the rest-hash is a distinct value the body can name `options`, as
Rails does.

## Acceptance criteria

1. `createTable` / `buildCreateTableDefinition` / `createJoinTable` split Rails'
   named kwargs from the `**options` rest, matching `schema_statements.rb:293`,
   `:331`, `:389`.
2. The bodies name the rest-hash `options`, and
   `validateCreateTableOptionsBang` / `findJoinTableName` receive it under that
   name.
3. `pnpm parity:api:calls:args` green; the `naming` row count for
   `schema-statements.ts` drops.
4. `pnpm parity:api` / `pnpm parity:api:extra` non-negative.
