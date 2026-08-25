---
title: "Delete the dead PG aliased_types @nie stub shadowed by the real class override"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 15
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:864-870`
carries a dead module-level `@nie` stub:

```ts
/** @internal */
function aliasedTypes(name: any, fallback: any): never {
  // @nie disposition=port-real rails=.../postgresql/schema_definitions.rb:289 cluster=pg-long-tail
  throw new NotImplementedError(
    "ActiveRecord::ConnectionAdapters::PostgreSQL::TableDefinition#aliased_types is not implemented",
  );
}
```

It is unreachable. The real port already exists as a class member on
`PostgreSQL::TableDefinition` at the same file's `:274`:

```ts
override aliasedTypes(_name: string, fallback: string): string {
  return fallback;
}
```

which matches Rails
`activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:288-290`
(`def aliased_types(name, fallback) = fallback`) exactly. Nothing in the package
references the free function — `grep -rn 'aliasedTypes'` finds only the class
override, the abstract base (`abstract/schema-definitions.ts:1098`, `:1461`),
the MySQL override (`mysql/schema-definitions.ts:256`) and tests.

Found while porting the sibling `integer_like_primary_key_type` in PR #6109,
which had the identical dead-stub-shadowing-a-real-override shape; that one was
deleted there because the story touched it. This one was left alone as
out of scope.

## Converged shape

Delete the free `aliasedTypes` function. Drop the `NotImplementedError` import
from the file if it becomes unused (check: other `@nie` stubs may still use it).

This is pure removal — no behavior change, since the function is never called.
Expect no `parity:api` movement (the class override is what is scored) and one
fewer `@nie` row.

## Acceptance criteria

- [ ] The module-level `aliasedTypes` stub is gone from
      `connection-adapters/postgresql/schema-definitions.ts`.
- [ ] `pnpm parity:api` delta non-negative; `pnpm parity:api:extra --package activerecord`
      does not grow.
- [ ] No unused-import lint error left behind.
