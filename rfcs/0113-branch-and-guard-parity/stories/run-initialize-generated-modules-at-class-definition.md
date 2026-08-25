---
title: "Run initializeGeneratedModules at class-definition time and drop the instanceof gate arm"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
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

Rails runs `initialize_generated_modules` from the `included do` block —
`vendor/rails/activerecord/lib/active_record/attribute_methods.rb:10-11`:

```ruby
included do
  initialize_generated_modules
  include Read
  ...
end
```

so every AR class has its named `GeneratedAttributeMethods` module built and
included before its class body runs. trails has no `included` hook, so
`initializeGeneratedModules` is called lazily from `defineAttributeMethods`
(`packages/activerecord/src/attribute-methods.ts:342-360`). A class-body
`aliasAttribute` gets there first — `aliasAttribute` →
`eagerlyGenerateAliasAttributeMethods` → `generateAliasAttributeMethods` →
`aliasAttributeMethodDefinition` → `generatedAttributeMethods.call(host)`
(`packages/activemodel/src/attribute-methods.ts:329`) — and ActiveModel builds a
bare `Module` under the same `_generatedAttributeMethods` name
(activemodel/lib/active_model/attribute_methods.rb:400-402).

PR #6389 handled the inversion with an extra arm on the lazy-init gate:

```ts
if (
  !Object.prototype.hasOwnProperty.call(this, "_generatedAttributeMethods") ||
  !(this._generatedAttributeMethods instanceof GeneratedAttributeMethods)
) {
```

The `instanceof` arm has no Rails counterpart, and the replacement it triggers
strands whatever ActiveModel already generated in the superseded carrier (still
in the prototype chain, so it still resolves, but `undefine_attribute_methods`
no longer reaches it).

## Converged shape

Run `initializeGeneratedModules` at class-definition time — the trails analogue
of `included do` — so the module exists and is included before any class body
can generate a method, and drop the `instanceof` arm, leaving the plain own-
property gate Rails' per-class ivar gives for free. `registerSubclass` /
`registerModel` and the `Base` static wiring are the candidate hook points.

## Acceptance criteria

- `initializeGeneratedModules` runs before a class body can reach
  `generated_attribute_methods`; ActiveModel never builds the bare `Module` for
  an AR class.
- The `instanceof GeneratedAttributeMethods` arm is deleted from
  `defineAttributeMethods`, along with its call-site comment.
- `attribute-methods.trails.test.ts` "initializeGeneratedModules replaces a
  module ActiveModel built first" is retired or rewritten to assert the module
  is the AR one from the start (no replacement happens).
- `AttributeMethodsTest > generated attribute methods ancestors have correct
module` still passes.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
