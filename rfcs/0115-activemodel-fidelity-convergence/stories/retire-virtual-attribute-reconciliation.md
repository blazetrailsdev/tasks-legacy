---
title: "Retire reconcileVirtualAttributes and the virtual-attribute flag (column_names is DB-sourced in Rails)"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`reconcileVirtualAttributes` (`packages/activerecord/src/model-schema.ts`), its
`_virtualAttributesReconciled` one-shot flag, the `virtual` field on an
attribute definition, and the `cachedColumnNames` / `reflectColumnNames` pair
that feed it have **no Rails counterpart**. Rails' `column_names` is always
`columns.map(&:name)` — purely DB-sourced
(`vendor/rails/activerecord/lib/active_record/model_schema.rb:437-441`) — so an
`attribute()` declaration that is not a real column simply never appears in it,
and nothing has to classify anything.

The machinery exists only because trails' schema reflection is async, so
`ensureSchemaLoaded` can short-circuit with a synthesized `columnsHash` that
cannot tell a virtual attribute from a real column.

It is also a repeat source of defects. In #6948 it produced two in one PR:

1. `reflectColumnNames` acquired its connection through `reflectionAdapter`,
   whose last resort is `leaseConnectionSync` — a _permanent_ checkout, which
   tripped `permanent_connection_checkout = :deprecated` on every save. Fixed
   there by routing through `withConnection`.
2. An early return added to skip that lease silently dropped the function's
   _other_ responsibility — `reflectColumnNames` re-warms a cleared schema
   cache and then calls `reloadSchemaFromCache` — reddening
   `base.trails.test.ts`'s "an STI subclass's own columns memo is rebuilt when
   the base re-reflects" on all three adapter lanes. Two unrelated jobs are
   tangled in one function.

## Converged shape

Delete `reconcileVirtualAttributes`, `_virtualAttributesReconciled`,
`cachedColumnNames`, `reflectColumnNames` and the `virtual` flag. `columnNames`
becomes purely DB-sourced off `columns_hash`, as `model_schema.rb:437-441` has
it, and a declared-but-columnless attribute is excluded by construction rather
than by a flag someone has to maintain.

The cache-rewarm/reload side effect currently buried in `reflectColumnNames`
must be re-homed before deletion — it is what keeps a model that synthesized a
minimal `columnsHash` against a cold cache from serving that stale view
forever. It belongs with the schema load, not with attribute classification.

Depends on the synthesized-`columnsHash` fallback going away; see
[[retire-remaining-attribute-definitions-registry]].

## Acceptance criteria

- [ ] `reconcileVirtualAttributes`, `_virtualAttributesReconciled`,
      `cachedColumnNames`, `reflectColumnNames` and the `virtual` flag are gone.
- [ ] `columnNames()` is derived from `columns_hash` only.
- [ ] `base.trails.test.ts`'s STI columns-memo case and
      `column-names-sync-virtual-exclusion.test.ts` still pass, or are replaced
      by coverage of the re-homed reload.
- [ ] `connection-handling.test.ts`'s "common APIs don't permanently hold a
      connection…" still passes on all adapter lanes.
