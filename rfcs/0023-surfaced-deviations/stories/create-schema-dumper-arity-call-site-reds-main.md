---
title: "createSchemaDumper call site passes the adapter where options belong, redding main"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`main` does not typecheck at d059b0dc4.

`packages/activerecord/src/adapters/postgresql/postgresql-adapter.trails.test.ts:123`
calls

```ts
await adapter.createSchemaDumper(adapter).dumpTable(lines, "ex_idx_both");
```

PR #7014 converged `create_schema_dumper`'s arity to Rails'
`create_schema_dumper(options)`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1541-1543`)
— the receiver is `self`, the single argument is the options hash. The trails
signature is now
`createSchemaDumper(options: Record<string, unknown> = {}): SchemaDumper`
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1791`).

PR #7017 then landed the call site above, which passes the _adapter_ where
options belong — the pre-convergence trails shape. `tsc --build` fails:

```text
postgresql-adapter.trails.test.ts(123,40): error TS2345: Argument of type
'PostgreSQLAdapter' is not assignable to parameter of type
'Record<string, unknown>'. Index signature for type 'string' is missing.
```

This blocks every commit in every worktree, because the husky pre-commit hook
runs `tsc --build`.

Every other call site in the repo already passes no argument —
`adapters/postgresql/extension-migration.test.ts:70`,
`adapters/postgresql/schema.test.ts:701,744,759,778`, and the internal
`schema-dumper.ts:603` passes `{}`.

Surfaced while rebasing PR #7016, which had to carry the one-line fix locally to
be able to commit at all; review asked for it to live in its own PR instead.

## Acceptance criteria

- [ ] `postgresql-adapter.trails.test.ts:123` calls `adapter.createSchemaDumper()`,
      matching Rails' receiver/options split and every sibling call site.
- [ ] `pnpm build` is clean on `main`.
- [ ] The PG lane stays green for that test.
