---
title: "assign_nested_attributes_for_collection_association still skips an unloaded collection"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

# assign_nested_attributes_for_collection_association skips an unloaded collection

## Context

`nested_attributes.rb:510-515`:

```ruby
existing_records = if association.loaded?
  association.target
else
  attribute_ids = attributes_collection.filter_map { |a| a["id"] || a[:id] }
  attribute_ids.empty? ? [] : association.scope.where(association.klass.primary_key => attribute_ids)
end
```

Rails **queries the DB** for the existing records when the association is not
loaded. trails only marks already-loaded records — see the `KNOWN LIMITATION`
block at `packages/activerecord/src/nested-attributes.ts:893-901`, whose stated
reason is that the sync setter cannot perform trails' async load.

Note `nested-attr-collection-existing-records-query-when-unloaded`
(0023-surfaced-deviations) is marked `done`, but this arm is still divergent in
`main` as of PR #6887 — re-check what that story actually landed before
starting, and fold this into it if it turns out to be the same gap.

The comment claims the case is "not exercised by any test", so a regression test
that fails on the baseline is part of the work (see
`nested-attributes-unloaded-update.trails.test.ts` for the nearest existing
cover).

## Converged shape

`assignNestedAttributesForCollectionAssociation` resolves `existing_records`
through the association scope when the association is unloaded, per
nested_attributes.rb:510-515, so `mark_for_destruction` and the pre-save
size-validation interaction see the same set Rails sees. The async-load
constraint is the real obstacle — the settled trails idiom for a reader that
must await is the park/drain shape already used elsewhere in this file, not a
new sync/async split.

## Acceptance criteria

- [ ] `existing_records` follows nested_attributes.rb:510-515 for both the
      loaded and unloaded arms.
- [ ] A regression test that fails on the current baseline covers the unloaded
      collection with `allow_destroy`.
- [ ] The `KNOWN LIMITATION` comment block is deleted, not reworded.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
