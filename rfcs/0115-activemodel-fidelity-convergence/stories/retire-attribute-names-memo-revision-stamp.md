---
title: "Retire the revision-stamped attribute_names memo for Rails' nil-on-reload"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: api-compare
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails memoizes class-level `attribute_names` with a plain ivar
(`activerecord/lib/active_record/attribute_methods.rb:236-242`):

```ruby
def attribute_names
  @attribute_names ||= if !abstract_class? && table_exists?
    attribute_types.keys
  else
    []
  end.freeze
end
```

and invalidates it in exactly two places: `inherited` nils it on the new
subclass (`attribute_methods.rb:265-272`) and `reload_schema_from_cache` nils
it on self and every descendant (`model_schema.rb:553-570`, `@attribute_names
= nil` at :564).

trails carries a revision-stamped memo instead —
`_attributeNamesMemo = { revision, names }` compared against `_schemaRevision`
(`packages/activerecord/src/attribute-methods.ts`) — PLUS
`clearAttributeNamesMemo` (`packages/activerecord/src/model-schema.ts`), which
already walks `[host, ...descendants]` deleting the own memo, i.e. exactly
Rails' `reload_schema_from_cache` recursion.

The revision stamp was the workaround for two things:

1. no JS `inherited` hook, so a subclass memo the base's reset can't reach
   would go stale (CLAUDE.md, "Module mixins");
2. reset paths that clear `_columns` / `_columnsHash` without going through
   `clearAttributeNamesMemo`.

Since `clearAttributeNamesMemo` walks descendants, (1) is already covered for
every path that calls it, and (2) is a fixable gap in the reset paths rather
than a language limitation — the two-mechanism overlap is the deviation. The
same function also clears `_columnNamesMemo`, so both memos share the shape.

PR #6789 additionally converged `reloadSchemaFromCache` to send
`reset_default_attributes!` (attributes.rb:268-271), which narrows how many
reset paths exist to audit.

## Converged shape

`_attributeNamesMemo` becomes a plain frozen own-property memo — no revision
field, no `_schemaRevision` comparison — and every path that today bumps
`_schemaRevision` to invalidate it instead calls `clearAttributeNamesMemo`,
which is the port of Rails' recursive nil. Audit the bump sites
(`resetColumnInformation`, `applyColumnsHash`, `reloadSchemaFromCache`,
`attribute`, `table_name=`, `ignored_columns=`) and route each through the one
invalidation.

Keep the `exists === undefined` fail-open arm unmemoized as it is today: that
is the genuine async `table_exists?` deviation and is separate from this.

## Acceptance criteria

- [ ] `_attributeNamesMemo` carries names only; no revision stamp and no
      `_schemaRevision` read in `classAttributeNames`.
- [ ] Every reset path that must invalidate it calls `clearAttributeNamesMemo`
      (the port of model_schema.rb:553-570), with a test per path.
- [ ] `_columnNamesMemo` converges the same way or the story states why it
      cannot.
- [ ] `PersistenceTest > reset column information resets children` and the
      schema-reload suites stay green — that test is the existing guard for a
      subclass reading a stale memo.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
