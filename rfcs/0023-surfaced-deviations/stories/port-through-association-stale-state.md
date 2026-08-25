---
title: "Port ThroughAssociation#stale_state so through collections detect a stale target"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ThroughAssociation#stale_state`
(`activerecord/lib/active_record/associations/through_association.rb:82-87`):

```ruby
def stale_state
  if through_reflection.belongs_to?
    Array(through_reflection.foreign_key).filter_map do |foreign_key_column|
      owner[foreign_key_column]
    end.presence
  end
end
```

trails ports `stale_state` only on `BelongsToAssociation`
(`packages/activerecord/src/associations/belongs-to-association.ts:207`); the
through override is missing, so `Association#isStaleTarget`
(`packages/activerecord/src/associations/association.ts:222`) is never true for a
`has_many :through` / `has_one :through` whose through reflection is a
`belongs_to`. Writing the through FK on a loaded owner leaves the stale target in
place.

Surfaced while converging `CollectionAssociation#reader`'s stale-reload arm
(PR #6673): the arm is now in Rails' place, but no collection association can
reach it, so the regression pin has to force `isStaleTarget` with a spy rather
than write the FK.

## Converged shape

Port the override on the through-association mixin at the Rails name, with the
`through_reflection.belongs_to?` guard, `Array(...)`/`filter_map` over the
composite FK, and `presence` (blank → nil, so an all-nil FK is not "stale").

## Acceptance criteria

- [ ] `staleState` ported on the through association, Rails' guard and `presence` included.
- [ ] A test writes the through FK on a loaded owner and asserts the reader reloads.
