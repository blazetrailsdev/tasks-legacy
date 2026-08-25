---
title: "Base#inspect / pretty_print render unqualified class name for namespaced models"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #4865 (`relation-inspect-wrapper-qualified-class-name`), which
fixed the same bug one level up: `Relation#inspect` rendered the wrapper class
via JS `constructor.name` (unqualified) where Rails renders `self.class.name`
(the namespace-qualified constant path).

The identical divergence remains on the **record** inspect path, for
module-namespaced models:

- `packages/activerecord/src/core.ts:791` — `inspectWithAttributes` does
  `const ctor = this.constructor as { name: string }` and renders
  `` `#<${ctor.name} ...>` ``. trails declares namespaced models as flat JS
  classes (`AdminUser`, `ClothingItemUsed`), so `ctor.name` is `"AdminUser"`
  where Rails' `self.class.name` is `"Admin::User"`.
  Rails: `core.rb:782-784` (`inspect` → `inspect_with_attributes`).
- `packages/activerecord/src/pretty-print.ts:82` — `objectAddressGroup` uses
  `(obj as {constructor?: {name?: string}}).constructor?.name ?? "Object"`.
  Rails' `Base#pretty_print` (`core.rb:798-800`) calls `pp.object_address_group(self)`,
  and Ruby's `PP#object_address_group` derives the label from
  `Kernel.instance_method(:to_s).bind_call(obj)` → `#<Admin::User:0x...>` —
  also qualified.

The fix mechanism already exists and is proven: `qualifiedName()` in
`inheritance.ts` returns `"ClothingItem::Used"` / `"Admin::User"`, asserted by
`inheritance-namespaced.test.ts:48`. Non-namespaced models are unaffected
(`qualifiedName(Post)` === `"Post"`), so `core.test.ts`'s pinned
`#<Topic id: 1, ...>` literals stay green.

Note this is NOT covered by the done stories `core-inspect-attributes-for-inspect`
(which attributes are listed) or `module-namespaced-sti-polymorphic-name`
(sti_name / polymorphic_name columns) — neither touches the inspect class label.

## Acceptance criteria

- `Base#inspect` renders the qualified constant path for namespaced models:
  `#<Admin::User id: 1>`, not `#<AdminUser id: 1>`.
- `Base#pretty_print` / `objectAddressGroup` render the same qualified label.
- Source the name from the existing `qualifiedName()` rather than a new
  mechanism or `constructor.name`.
- Non-namespaced models are unchanged; existing pinned literals in
  `core.test.ts` stay green.
- Cover with a test on the canonical namespaced models (`AdminUser` /
  `ClothingItemUsed`); if a like-named Rails test exists use its name verbatim,
  otherwise place it in a `*.trails.test.ts` sibling.
- parity:api / parity:test deltas non-negative.
