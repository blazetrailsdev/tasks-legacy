---
title: "_createRecord narrows its attribute_names default where Rails narrows in attributes_for_create"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
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

Surfaced in PR #7029 (`converge-values-readers-drop-initialized-filter`) and
confirmed by that PR's reviewer as pre-existing, not introduced there.

`vendor/rails/activerecord/lib/active_record/persistence.rb:918-942`:

```ruby
def _create_record(attribute_names = self.attribute_names)
  attribute_names = attributes_for_create(attribute_names)
  ...
```

`self.attribute_names` is `@attributes.keys` verbatim
(`attribute_methods.rb:333-336`) — no further narrowing. The narrowing Rails
does have lives one call later, inside `attributes_for_create`
(`attribute_methods.rb:519-524`), which is `attribute_names & self.class.column_names`.

trails' `_createRecord` (`packages/activerecord/src/persistence.ts`, the
`selfNames` default) applies its own
`.filter((k) => Object.hasOwn(ctor.attributeTypes(), k))` directly to the
default argument. It is an extra narrowing at a site Rails does not narrow,
against `attributeTypes()` rather than `columnNames`, and it is functionally
inert given the downstream intersection — which is exactly why it has survived
unnoticed.

## Converged shape

The default argument is `this.attributeNames()` alone, matching
`_create_record(attribute_names = self.attribute_names)`. Whatever narrowing is
still needed belongs in `attributesForCreate`, spelled as Rails spells it
(`attribute_names & column_names`), so the two methods divide the work the way
`persistence.rb` / `attribute_methods.rb` do.

Check `attributesForCreate` in trails first: if it already intersects with
`columnNames`, deleting the filter is the whole change; if it does not, the
intersection moves there.

## Acceptance criteria

- `_createRecord`'s default is `this.attributeNames()` with no trailing filter.
- `attributesForCreate` performs Rails' `& column_names` intersection.
- `persistence.test.ts`, `dirty.test.ts`, `timestamp.test.ts` and the locking
  suites green on sqlite3, PostgreSQL and MySQL — the partial-insert and
  optimistic-locking paths are the ones that read this list.
