---
title: "Burn down internalSchemaCache readers onto the bound schema reflection"
status: draft
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AbstractAdapter#internalSchemaCache`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`, the
`internalSchemaCache` getter) is a trails-only accessor. Its own JSDoc says so:
Rails has no adapter-level accessor for the raw `SchemaCache` — `@schema_cache`
holds the pool-bound `BoundSchemaReflection`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:298`),
and every read goes through that one-arg handle.

`converge-column-for-attribute-onto-bound-schema-cache` (PR #6369) took
`abstract-adapter.ts` itself to zero raw-cache call sites, which was the scope
of that story. The accessor survives because ~33 non-test references remain
across:

- `packages/activerecord/src/model-schema.ts`
- `packages/activerecord/src/attribute-methods/primary-key.ts`
- `packages/activerecord/src/connection-adapters/schema-cache.ts`
- `packages/activerecord/src/connection-adapters/pool-config.ts`
- `packages/activerecord/src/support/schema-cache-dump.ts`

The JSDoc's stated reason is "sync peeks and pool-arg-taking reads that our
async BoundSchemaReflection can't serve" — that is the thing to converge, not a
permanent licence. Rails' equivalents are all one-arg reads on the bound handle
(`schema_reflection.rb` / `bound_schema_reflection`).

## Converged shape

Readers take `this.schemaCache.<x>(tableName)` — the pool-bound handle, one
argument, no pool plumbing. Where a caller genuinely needs a synchronous peek,
that call site is the finding: converge it or record it with a Rails cite
rather than reaching for the raw cache. The accessor is deleted once its last
reader is gone.

## Acceptance criteria

- [ ] Non-test `internalSchemaCache` readers reduced; each converted call site
      uses the bound one-arg form.
- [ ] Any reader that cannot convert is justified AT THE CALL SITE with a Rails
      cite naming the sync constraint.
- [ ] `internalSchemaCache` is deleted when the last reader goes (may be a
      follow-up story if the burndown does not fit one PR).
- [ ] `pnpm parity:api:calls` / `parity:api:calls:args` green.
