---
title: "column-for-attribute-null-column"
status: done
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7046
claim: "2026-08-25T15:54:32Z"
assignee: "converge-association-check-klass-onto-reflection-check-validity"
blocked-by: null
closed-reason: null
---

## Context

Found by the prism-codegen conformance scorer triage (PR #5727). Rails
column_for_attribute falls back to ConnectionAdapters::NullColumn.new(name)
(vendor/rails/activerecord/lib/active_record/model_schema.rb:463-468); the
port returns a bare { name, null: true, type: null } literal
(packages/activerecord/src/model-schema.ts:805-809). Class identity and the
rest of the NullColumn surface differ; Rails tests assert on NullColumn.

## Acceptance criteria

- Port returns a NullColumn instance (ported per Rails layout) from
  columnForAttribute for unknown attributes.
- Rails' column_for_attribute tests for the null case ported/verified.

## Triage note (2026-08-18): the blocker is gone, this is now a small fix

`ActiveRecord::ConnectionAdapters::NullColumn` **is now ported**, at
`packages/activerecord/src/connection-adapters/column.ts:198`
(`export class NullColumn extends Column`, carrying the
`Mirrors: ActiveRecord::ConnectionAdapters::NullColumn` line).

`columnForAttribute` (`packages/activerecord/src/model-schema.ts:807-811`) still
has not been repointed at it:

```ts
export function columnForAttribute(this: SchemaHost, name: string): any {
  loadSchema.call(this);
  const hash = getColumnsHash(this);
  return hash[name] ?? { name, null: true, type: null };
}
```

Rails is `columns_hash.fetch(name) { NullColumn.new(name) }`
(`model_schema.rb:463-468`) — and note `fetch` with a block, not `??`: a stored
`nil`/`false` column would be returned, not replaced (CLAUDE.md, "`fetch` vs `??`").

So this is now a one-line change plus whatever the class-identity assertions in
the Rails tests need. Also check `project_type_for_attribute_never_raises_returns_nil_type`
before assuming the sibling reader should raise.
