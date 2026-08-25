---
title: "through-cant-associate-error-takes-owner-and-reflection"
status: draft
updated: 2026-08-18
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

# `ThroughCantAssociateThroughHasOneOrManyReflection` family takes `(owner, reflection)`

## Context

PR #6684 converged `ThroughNestedAssociationsAreReadonly` and its two
subclasses onto Rails'
`initialize(owner = nil, reflection = nil)`
(`activerecord/lib/active_record/associations/errors.rb:224-238`), deriving
Rails' message `Cannot modify association '#{owner.class.name}##{reflection.name}'
because it goes through more than one other association.` from the pair, with
the argument-less fallback `Through nested associations are read-only.`

Its sibling family in the same file was left as-is and still takes two strings
with a trails-invented message
(`packages/activerecord/src/associations/errors.ts`, around the
`ThroughCantAssociateThroughHasOneOrManyReflection` /
`HasManyThroughCantAssociateThroughHasOneOrManyReflection` /
`HasOneThroughCantAssociateThroughHasOneOrManyReflection` block):

```ts
constructor(owner: string, association: string) {
  super(
    `Cannot modify association '${association}' on ${owner} because the source reflection is through a has_one or has_many reflection.`,
  );
}
```

Rails (`associations/errors.rb:177-185`):

```ruby
class ThroughCantAssociateThroughHasOneOrManyReflection < ActiveRecordError
  def initialize(owner = nil, reflection = nil)
    if owner && reflection
      super("Cannot modify association '#{owner.class.name}##{reflection.name}' because the source reflection class '#{reflection.source_reflection.class_name}' is associated to '#{reflection.through_reflection.class_name}' via :#{reflection.source_reflection.macro}.")
    else
      super("Cannot modify association.")
    end
  end
end
```

Callers: `associations/collection-proxy.ts` (`_ensureThroughWritable`),
plus any `ensure_mutable` caller once
`ensure-mutable-raises-bare-error-instead-of-the-through-error-class` lands.

## Converged shape

- The base class takes `(owner?, reflection?)` and derives Rails' message,
  including the `source_reflection.class_name` / `through_reflection.class_name`
  / `source_reflection.macro` interpolations and the `"Cannot modify association."`
  fallback; the two subclasses are empty (Rails declares them with no body).
- Call sites pass the owner record and the reflection, as PR #6684's
  `Has{One,Many}ThroughNestedAssociationsAreReadonly` sites now do.
- The trails tests asserting the old message text are updated to the Rails
  message (test NAMES unchanged).

## Acceptance criteria

- [ ] The three classes match `associations/errors.rb:177-185, 221-222` in
      arguments and message, fallback arm included.
- [ ] Every call site passes `(owner, reflection)`.
- [ ] `pnpm parity:api:calls:args` stays green, and no new baseline row.
