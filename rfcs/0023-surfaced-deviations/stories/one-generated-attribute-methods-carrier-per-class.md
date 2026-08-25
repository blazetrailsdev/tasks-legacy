---
title: "AR's initializeGeneratedModules splices a second carrier; ActiveModel's stays in the chain and keeps answering"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while debugging PR #6788. A model can end up with TWO generated-method
carriers spliced into its prototype chain, and the older one keeps answering.

Ruby has exactly one module per class. ActiveModel's accessor is

```ruby
def generated_attribute_methods                     # activemodel/lib/active_model/attribute_methods.rb:400-402
  @generated_attribute_methods ||= Module.new.tap { |mod| include mod }
end
```

and ActiveRecord replaces the whole thing before anything can call it, in
`initialize_generated_modules` off the `inherited` hook
(`activerecord/lib/active_record/attribute_methods.rb:265-272`), which
`const_set`s `GeneratedAttributeMethods` and includes that.

trails has no `inherited` hook, so ordering inverts: ActiveModel's
`generatedAttributeMethods` (`packages/activemodel/src/attribute-methods.ts`)
builds a bare `Module` and `include()`s it first — at the first
`attribute()`/`aliasAttribute()` in the class body — and generates its accessors
into it. When AR's `initializeGeneratedModules`
(`packages/activerecord/src/attribute-methods.ts`) later runs, its gate sees a
module that is not a `GeneratedAttributeMethods` and builds a second one.
`include()` splices the new carrier directly below the class prototype and
never unsplices the old, so both sit in the chain holding the same names.

Observable consequence: `undefineAttributeMethods` clears only the AR carrier,
so the AM carrier still answers. `"nameChanged" in Employee.prototype` is true
after an undefine, and only the (weaker) fresh-instance probe reads as cleared,
because nothing had driven AR generation in that scenario.

The existing test `initializeGeneratedModules replaces a module ActiveModel
built first` (`attribute-methods.trails.test.ts`) asserts the intent the code
does not yet deliver — it checks the ivar, not the ancestry.

## Converged shape

One carrier per class, as in Ruby. AR's `initializeGeneratedModules` removes the
ActiveModel-built module from the prototype chain (or empties it) when it
replaces it, so `undefine_attribute_methods` really undefines and no stale
generation shadows a later one.

## Acceptance criteria

- [ ] After AR's `initializeGeneratedModules`, exactly one generated-method
      carrier is reachable from the class prototype.
- [ ] `undefineAttributeMethods` leaves no generated name resolvable from
      `Klass.prototype` (prototype-level assertion, not a fresh instance).
- [ ] `initializeGeneratedModules replaces a module ActiveModel built first`
      is extended to assert the ancestry, not just the ivar.
