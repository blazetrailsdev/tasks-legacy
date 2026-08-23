---
title: "split-attribute-registration-attribute-from-attributes-attribute"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 130
priority: 55
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/attributes.rb:59-62` is three lines:

```ruby
def attribute(name, ...)
  super
  define_attribute_method(name)
end
```

The registration half — name resolution, type lookup, `hook_attribute_type`,
the `pending_attribute_modifications <<` appends and `reset_default_attributes`
— lives in `AttributeRegistration::ClassMethods#attribute`
(`vendor/rails/activemodel/lib/active_model/attribute_registration.rb:12-20`),
a different module in a different file.

trails inlines the whole registration body into
`packages/activemodel/src/attributes.ts` `attribute()`, so
`attribute-registration.ts` has no `attribute` at all, and attributes.ts has to
import `PendingType` / `PendingDefault` / `pendingAttributeModifications` /
`resetDefaultAttributes` across a file boundary Rails does not have. That
import list was raised in review on PR #6781; the split was prepared and then
backed out of that PR because the relocation plus the
`rails-file-structure-method-order` reflow it triggers costs ~130 LOC and put
the PR over its ceiling.

The move is mechanical and was verified green once already: lift the body of
`attributes.ts` `attribute()` (everything up to the closing
`_defineAttributeMethod.call(...)`) into `attribute-registration.ts` as
`export function attribute`, move the file-private `typeOptions` helper with
it, and leave attributes.ts with the two-line Rails body that imports the
registration one under an alias (TS has no `super` for a module outside the
prototype chain).

## Acceptance criteria

- `attribute-registration.ts` exports `attribute`, mirroring
  `attribute_registration.rb:12-20`, including the `typeOptions` helper it uses.
- `packages/activemodel/src/attributes.ts` `attribute()` is the Rails
  two-liner: call the registration `attribute`, then `defineAttributeMethod`
  (attributes.rb:59-62). The `super`-substitute is cited at the call site.
- attributes.ts no longer imports `PendingType`, `PendingDefault`,
  `pendingAttributeModifications`, or `resetDefaultAttributes`.
- `pnpm parity:api --package activemodel` stays at 719/719, and
  `pnpm parity:api:extra --package activemodel` shows no new novel row for
  either file.
- Parity deltas non-negative for activemodel and activerecord.
