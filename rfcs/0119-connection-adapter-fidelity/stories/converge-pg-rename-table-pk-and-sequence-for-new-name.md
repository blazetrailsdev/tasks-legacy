---
title: "rename_table passes new_name to pk_and_sequence_for, which resolves the schema itself"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6619 (RFC 0096 `wave-4-naming-ar-adapters`). Two naming rows on
`postgresql-adapter.ts` / `renameTable` / `pk_and_sequence_for`
(`ref:newName` -> `ref:renamedName`) are not renames: trails passes a DIFFERENT
value than Rails does.

Rails
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb`,
`rename_table`) passes the caller's `new_name` straight through:

```ruby
pk, seq = pk_and_sequence_for(new_name)
```

trails
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`,
`renameTable`) instead rebuilds a schema-qualified name first, because the
renamed table stays in the OLD schema:

```ts
const renamedName = oldSchema
  ? `${this.quoteColumnName(oldSchema)}.${this.quoteColumnName(unqualifiedNew)}`
  : unqualifiedNew;
const result = await this.pkAndSequenceFor(renamedName).catch(() => null);
```

Rails needs no such rebuild because `pk_and_sequence_for` resolves the name
through PG's `regclass` cast against the session `search_path`. The right
convergence is to make `pkAndSequenceFor` do that resolution itself, so
`renameTable` can pass `newName` as Rails does — the `.catch(() => null)` is a
second, related divergence (Rails does not rescue here).

## Acceptance criteria

- [ ] `pkAndSequenceFor` resolves a schema-qualified or bare name the way
      Rails' `regclass` lookup does, so callers need no pre-qualification.
- [ ] `renameTable` calls `pkAndSequenceFor(newName)`, and the `.catch(() =>
null)` is removed or justified against a Rails rescue.
- [ ] Both `renameTable` naming rows clear in
      `pnpm parity:api:calls:args:report`, with no new `shape` row.
- [ ] PostgreSQL lane green, including a rename of a table in a non-default
      schema.
