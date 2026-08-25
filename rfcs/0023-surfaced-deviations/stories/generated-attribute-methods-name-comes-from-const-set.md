---
title: "GeneratedAttributeMethods takes its name from const_set, not a post-construction stamp"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Surfaced in PR #6427 (`call-args-ar-host-param-core`) while converging
`GeneratedAttributeMethods.new` to Rails' zero-argument call.

Rails `activerecord/lib/active_record/attribute_methods.rb:42-50`:

```ruby
def initialize_generated_modules # :nodoc:
  @generated_attribute_methods = const_set(:GeneratedAttributeMethods, GeneratedAttributeMethods.new)
  private_constant :GeneratedAttributeMethods
  ...
```

`const_set` is what names the module — `Topic::GeneratedAttributeMethods` — and
`Module#inspect` reads that name off the constant binding. JS has no constant
binding, so PR #6427 kept the owner name as a writable `ownerName` field the
owner stamps immediately after construction, one statement where Rails has
none (`attribute-methods.ts` `initializeGeneratedModules`). The behaviour is
pinned by `attribute-methods.test.ts` "generated attribute methods ancestors
have correct module" (`mod.inspect()` === `"Topic::GeneratedAttributeMethods"`).

Also unported alongside it: `private_constant :GeneratedAttributeMethods`
(`attribute_methods.rb:44`).

## Converged shape

Look for a spelling that derives the name from the assignment instead of a
separate write — e.g. a `constSet(owner, name, mod)` helper in
`@blazetrails/activesupport` mirroring Ruby's `const_set` (it is a real Ruby
`Module` method, not invented surface), used by both this site and
`delegation.rb:65`'s `const_set(:GeneratedRelationMethods, mod)`, which trails
also elides. Then `initialize_generated_modules` is one statement per Rails
statement, and `ownerName` becomes an implementation detail of the helper.

## Acceptance criteria

1. `initializeGeneratedModules` has one TS statement per Ruby statement in
   `attribute_methods.rb:43-49`; no post-construction stamp.
2. `inspect()` still returns `"<Owner>::GeneratedAttributeMethods"` — the
   existing test name stays verbatim and green.
3. If a shared `constSet` lands, `delegation.rb:65` uses it too, or the story
   says why it does not.
