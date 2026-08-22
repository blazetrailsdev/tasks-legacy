---
title: "findPrototypeSetter's last caller: store accessors aren't reachable by name like Rails' define_method ones"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 180
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`findPrototypeSetter` (`packages/activerecord/src/persistence.ts`, ~1037-1053)
walks the prototype chain looking for `Object.getOwnPropertyDescriptor(proto,
key).set`. Rails never introspects descriptors — `_assign_attribute` dispatches
by name:

```ruby
def _assign_attribute(k, v)
  raise UnknownAttributeError.new(self, k.to_s) unless respond_to?("#{k}=")
  public_send("#{k}=", v)
end
```

(`vendor/rails/activemodel/lib/active_model/attribute_assignment.rb:44-48`.)

PR #6162 removed the nested-attributes caller of this walk
(`_reapplyNestedAttrSetters` now names `set#{Name}Attributes` directly). **One
caller remains**: `_assignAttribute`'s store-accessor arm
(`persistence.ts`, ~1095). It is cited at the call site, and it exists because
Rails' `store_accessor` generates ordinary methods with `define_method`
(`vendor/rails/activerecord/lib/active_record/store.rb:130-148`) that
`public_send` reaches by name, whereas trails generates _property setters_ —
and `record[k] = v` cannot distinguish "call the store accessor" from "write a
plain attribute slot", hence the descriptor lookup.

That asymmetry is the actual deviation. The walk is a symptom.

## Converged shape

Make trails' store accessors reachable by name, the same way the association
writers now are: generate the writer as a named method rather than (or in
addition to) a property setter, so `_assignAttribute` can dispatch through the
same by-name path as every other attribute and `findPrototypeSetter` has no
caller left. Then delete `findPrototypeSetter` outright.

Note the store-accessor _reader/writer pair_ must stay ergonomic
(`record.color = "red"` is how the Rails API reads), so the property setter
likely stays as the sugar while a name-dispatchable method becomes what
assignment routes through — the same shape RFC 0087 settled on for association
writers.

## Acceptance criteria

- `_assignAttribute`'s store-accessor arm dispatches by name, not by
  `Object.getOwnPropertyDescriptor`.
- `findPrototypeSetter` is deleted from `persistence.ts`.
- Existing store-accessor tests (including the dirty-tracking ones) stay green;
  `record.color = "red"` still works.
- `pnpm parity:api:extra --package activerecord` delta non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
