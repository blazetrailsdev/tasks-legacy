---
title: "to_key drops a partially-populated composite key where Rails keeps it"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# `to_key` drops a partially-populated composite key where Rails keeps it

## Context

Rails' `AttributeMethods::PrimaryKey#to_key`
(`activerecord/lib/active_record/attribute_methods/primary_key.rb:11-14`) is:

```ruby
def to_key
  key = id
  Array(key) if key
end
```

The guard is on `id` itself, so a composite key whose parts are `[1, nil]` is
truthy (a non-empty Array) and `to_key` answers `[1, nil]`.

`packages/activerecord/src/attribute-methods/primary-key.ts:24-30` adds a second
guard Rails does not have:

```ts
const arr = Array.isArray(pk) ? pk : [pk];
if (arr.some((v) => v == null)) return null;
```

so a half-populated composite key answers `null` where Rails answers the tuple.
Callers that treat `to_key` as "is this record keyed" get the same answer either
way for a scalar key; a composite one diverges. Rails routes the "all parts
present" question through `primary_key_values_present?`
(`composite_primary_key.rb:16-22`, `id.all?`), which trails now has on the two
modules after PR #6840 — so the extra guard is answering the wrong question in
the wrong method.

Pre-existing; surfaced while splitting the readers in PR #6840 (RFC 0112,
`restore-composite-primary-key-module-split`), which deliberately left `to_key`
alone as out of scope.

## Converged shape

```ts
toKey(): unknown[] | null {
  const key = this.id;
  return key != null ? (Array.isArray(key) ? key : [key]) : null;
}
```

— Ruby's `Array(key) if key`, with the truthiness rule from CLAUDE.md ("Ruby
idioms that do not translate literally"): `if key` is false only for
`nil`/`false`, never for a non-empty Array. Then check the call sites that
relied on the dropped guard and point them at `isPrimaryKeyValuesPresent()`.

## Acceptance criteria

- [ ] `toKey` mirrors `primary_key.rb:11-14` with no extra all-parts-present guard.
- [ ] A composite-PK record with one part written answers the tuple, not `null`.
- [ ] Callers wanting "all parts present" read `isPrimaryKeyValuesPresent()`.
- [ ] `primary-keys.test.ts` and the composite-PK suites pass on all three adapters.
