---
title: "BatchEnumerator#touch_all guards a call Rails makes unconditionally"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found while writing the per-site reasons for the `BatchEnumerator` `sum` rows
(PR #6721, `wave-4a-relation-family-residue`).

Rails (`vendor/rails/activerecord/lib/active_record/relation/batches/batch_enumerator.rb:84-88`):

```ruby
def touch_all(...)
  sum do |relation|
    relation.touch_all(...)
  end
end
```

`relation.touch_all` is called unconditionally — every batch is a Relation and
Relation responds to `touch_all`.

trails (`packages/activerecord/src/relation/batches/batch-enumerator.ts`,
`touchAll`) guards it:

```ts
for await (const batchRelation of this) {
  if (typeof batchRelation.touchAll === "function") {
    total += await batchRelation.touchAll(...args);
  }
}
```

The guard exists only because the local host interface declares
`touchAll?(...)` optional (`batch-enumerator.ts:19`). Rails has no such arm, and
the silent-skip semantics differ: a batch whose relation somehow lacked
`touch_all` contributes 0 to the total in trails and raises `NoMethodError` in
Rails. That is a swallowed error, not a language shortcoming.

## Converged shape

Make the host interface's `touchAll` required (it is a real `Relation` method)
and drop the `typeof` guard, so the body is Rails' unconditional
`relation.touch_all(...)` accumulation:

```ts
for await (const batchRelation of this) {
  total += await batchRelation.touchAll(...args);
}
```

## Acceptance criteria

- [ ] `touchAll` on the `BatchEnumerator` host interface is required, not optional.
- [ ] The `typeof ... === "function"` guard is gone; the call is unconditional
      per batch_enumerator.rb:84-88.
- [ ] `pnpm parity:api:calls` / `:args` green; the `touch_all -> sum` row keeps
      its (still accurate) Enumerable#sum reason.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
