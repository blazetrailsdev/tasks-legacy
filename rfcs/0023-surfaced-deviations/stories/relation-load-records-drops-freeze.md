---
title: "relation-load-records-drops-freeze"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`pnpm codegen:score` scores `active_record/relation.rb :: loadRecords` as
divergent, and the triage under `conformance-triage-burndown` verified it as a
real deviation, not a tooling artifact.

Rails (`vendor/rails/activerecord/lib/active_record/relation.rb:1331-1334`):

```ruby
def load_records(records)
  @records = records.freeze
  @loaded = true
end
```

trails (`packages/activerecord/src/relation.ts:6804-6807`) copies instead of
freezing:

```ts
protected loadRecords(records: T[]): void {
  this._records = [...records];
  this._loaded = true;
}
```

The frozen array is what makes a loaded relation's `records` immutable to its
callers — Rails raises `FrozenError` on an in-place mutation of `relation.records`,
where trails silently accepts it and the mutation sticks in the relation's cache.
The copy also diverges the other way: Rails keeps the caller's array identity.

## Acceptance criteria

- `loadRecords` assigns the frozen argument (`Object.freeze(records)`), matching
  Rails' two statements and their order.
- A regression test pins that mutating a loaded relation's records throws
  rather than silently editing the relation's cache.
- The `active_record/relation.rb::loadRecords::divergent` baseline row in
  `scripts/prism-codegen/convergence-baseline.json` is deleted (only-shrink; do
  not reseed).
