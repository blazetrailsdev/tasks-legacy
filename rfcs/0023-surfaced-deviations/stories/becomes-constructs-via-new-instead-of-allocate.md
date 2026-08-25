---
title: "becomes constructs via new with suppression flags where Rails uses klass.allocate"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `becomes` (activerecord/lib/active_record/persistence.rb:487-500) allocates without
running `new`:

```ruby
def becomes(klass)
  became = klass.allocate
  became.send(:initialize) do |becoming|
    @attributes.reverse_merge!(becoming.instance_variable_get(:@attributes))
    ...
  end
  became
end
```

`klass.allocate` skips `Inheritance::ClassMethods#new` entirely, so neither the abstract/Base
guard (inheritance.rb:56-59) nor the STI `subclass_from_attributes` dispatch (inheritance.rb:61-75)
runs.

trails' `becomes` (packages/activerecord/src/persistence.ts:~1430-1480) has no `allocate`, so it
constructs with `new klass({}, block)` and suppresses the two behaviours `new` would add via
class-level flags it sets and restores around the call: `_suppressStiNewDispatch` and
`_suppressAbstractCheck` (read at packages/activerecord/src/base.ts:991). Both flags are trails
inventions with no Rails counterpart, and they are class-level mutable state on a hot path.

Surfaced by trails#6926: the flags mask the difference only partially. `becomes(Base)` "worked" in
trails while Rails raises `TableNotSpecified` (allocate → `send(:initialize)` seeds `@attributes`
from `_default_attributes`, which loads the schema for a table-less Base), because trails'
`descends_from_active_record?` was answering `true` for Base. Fixing the predicate to Rails'
`if self == Base then false` (inheritance.rb:83-84) exposed it.

## Converged shape

Construct the instance without entering the `new` override — a JS analogue of `allocate` —
e.g. `Object.create(klass.prototype)` followed by the same initialization body `new` runs, so
the two suppression flags and their save/restore dances can be deleted from `persistence.ts`
and `base.ts:991` can drop its `_suppressAbstractCheck` term. Check whether the identical
`_suppressStiNewDispatch`/`_suppressAbstractCheck` save-and-restore at base.ts:2873-2889 is the
same pattern and can go with it.

Related: `decompose-instantiate-onto-allocate-init-with-attributes` (0023-surfaced-deviations)
covers the sibling `instantiate` path and may share the mechanism.

## Acceptance criteria

- [ ] `becomes` no longer routes through `new`, mirroring persistence.rb:487-489.
- [ ] `_suppressAbstractCheck` and `_suppressStiNewDispatch` are deleted (or reduced to whatever
      genuinely has a Rails counterpart), including the `base.ts:991` guard term.
- [ ] `parity:api:extra --package activerecord` does not regress.
- [ ] `persistence.trails.test.ts` → "becomes bypasses the abstract-instantiation guard" and the
      Rails `becomes` cases in `persistence.test.ts` stay green.
