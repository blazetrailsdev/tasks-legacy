---
title: "Neither _assignAttribute models Rails' respond_to?(setter) re-raise arm"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced in PR #6738, which converged ActiveModel's `_assignAttribute` onto
`attribute_assignment.rb:67-75` (story
`activemodel-assign-attribute-still-writes-through-write-attribute`).

Rails:

```ruby
def _assign_attribute(k, v)
  setter = :"#{k}="
  public_send(setter, v)
rescue NoMethodError
  if respond_to?(setter)
    raise
  else
    attribute_writer_missing(k.to_s, v)
  end
end
```

Both trails copies —
`packages/activemodel/src/attribute-assignment.ts` `_assignAttribute` and
`packages/activerecord/src/persistence.ts` `_assignAttribute` — resolve the
setter first and branch on whether it exists, so they never enter a rescue and
never reach the `raise` at `:70-71`. In the common case that is equivalent: a
`NoMethodError` thrown from inside a setter that DOES exist propagates out of
the call, which is what the re-raise does.

It is NOT equivalent where the two differ on what "the setter exists" means.
Ruby's `respond_to?(setter)` consults the receiver's full method table
(including `method_missing`-backed names via `respond_to_missing?`) at the
moment of the rescue; trails asks its own resolution ladder (own accessor →
generated `name=` key → `matchedAttributeMethod`) BEFORE dispatching. A name
that trails' ladder cannot see but that the receiver would in fact answer goes
to `attribute_writer_missing` where Rails would re-raise the original
`NoMethodError`.

`ar-assign-attribute-bypasses-attribute-writer-missing` (0023) was closed on
2026-08-09 with `closed-reason: "Already done: persistence.ts:1004
_assignAttribute now calls self.attributeWriterMissing(key, value) on the
no-setter arm"` — that covers only the `attribute_writer_missing` call, not
this arm, so this story carries the residue and is what the two call sites
should cite.

## Acceptance criteria

- [ ] Both `_assignAttribute` copies model the `respond_to?(setter)` re-raise
      arm, or the reason they cannot is recorded at each call site with the
      Ruby semantics it diverges from.
- [ ] The citation in `packages/activemodel/src/attribute-assignment.ts`'s
      `_assignAttribute` JSDoc points at this story rather than the closed one.
- [ ] activemodel + activerecord attribute-assignment suites green.
