---
title: "Retire the ad-hoc ThroughAssociation duck hosts; call the members on the real association"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR 6766.

Rails only ever calls `ThroughAssociation`'s members on an association
instance — the module is `include`d into `HasManyThroughAssociation` /
`HasOneThroughAssociation`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:8`,
`has_one_through_association.rb:7`) and `self` is always that association.

trails has two call sites with no association instance in hand, which fake a
`self` by spreading the module onto an owner/reflection pair:

- `packages/activerecord/src/associations.ts`,
  `_associationForeignKeyPresent` —
  `ThroughAssociation.foreignKeyPresent.call({ ...ThroughAssociation, owner: record, reflection })`
- `packages/activerecord/src/associations/has-many-through-association.ts`,
  `buildThroughInverseFor` — builds
  `{ owner, reflection, _throughScope, ...throughAssociationMethods }` and
  casts it to `HasManyThroughAssociation`.

Both are trails-only shapes: the module gets a duck `self` that is not an
`Association`, so any member that later reaches real association state
(`target`, `loaded`, `scope`) silently breaks at a distance. Rails reaches
`foreign_key_present?` off `owner.association(name)`
(`associations/association.rb`), which is available at both sites.

## Acceptance criteria

- [ ] Both sites obtain the real association (`owner.association(name)`) and
      call the member on it; the `{...ThroughAssociation, ...}` duck hosts and
      the `as unknown as HasManyThroughAssociation` cast are deleted.
- [ ] No new `parity:api:extra` surface; `parity:api:calls` / `:args` stay
      green.
- [ ] `associations/` suites green on SQLite, PostgreSQL and MySQL/MariaDB.
