---
title: "PG renameTable passes an invented renamedName to pkAndSequenceFor"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
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

Surfaced by the RFC 0096 wave-2 burndown (PR #6386), which renamed the
`rename_table` locals but could not converge this row.

Rails (`activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:436-455`)
looks the primary key and sequence up under the caller's `new_name` directly:

```ruby
def rename_table(table_name, new_name, **options)
  ...
  execute "ALTER TABLE #{quote_table_name(table_name)} RENAME TO #{quote_table_name(new_name)}"
  pk, seq = pk_and_sequence_for(new_name)
```

trails (`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`,
`renameTable`) instead builds a `renamedName` local and passes that:

```ts
const renamedName = oldSchema
  ? `${this.quoteColumnName(oldSchema)}.${this.quoteColumnName(unqualifiedNew)}`
  : unqualifiedNew;
const result = await this.pkAndSequenceFor(renamedName).catch(() => null);
```

That is an invented conversion — the port re-qualifies the new name with the
OLD schema because a PG rename leaves the table in place — plus a swallowed
error (`.catch(() => null)`) Rails does not have. Rails gets the same effect
from `pk_and_sequence_for`'s own `extract_schema_qualified_name` handling, so
the extra local is covering for a gap somewhere below rather than for a
PostgreSQL behavior. It also reports as a standing `naming` row
(`ref:newName` vs `ref:renamedName`) that no rename can close.

The same body also quotes with `quoteColumnName` where Rails uses
`quote_table_name` on the RENAME TO target and the index/sequence names
(`:441`, `:446`, `:452`); worth checking in the same pass.

## Acceptance criteria

- [ ] `renameTable` passes `newName` to `pkAndSequenceFor`, as
      `schema_statements.rb:440` does, with no `renamedName` local.
- [ ] Whatever `pkAndSequenceFor` needs to resolve an unqualified post-rename
      name is fixed inside `pkAndSequenceFor`, not at this call site.
- [ ] The `.catch(() => null)` is removed or replaced by the Rails control
      flow (`pk_and_sequence_for` returns nil; it does not raise).
- [ ] The `naming` row `renameTable` → `pk_and_sequence_for`
      (`ref:newName` vs `ref:renamedName`) is gone from
      `API_COMPARE_FORCE=1 pnpm parity:api --calls`.
- [ ] PG rename tests still pass, including the schema-qualified cases.
