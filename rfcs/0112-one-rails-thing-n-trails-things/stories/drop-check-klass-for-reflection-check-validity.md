---
title: "Association#initialize makes one check_validity! call, not check_klass + validateReflectionValidity"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
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

Surfaced while landing `converge-association-check-klass-onto-reflection-check-validity`
(PR #7060), which reduced `Association#checkKlass` to a bare `this.klass`
(`packages/activerecord/src/associations/association.ts`, `protected checkKlass()`).

What remains is that **Rails has no `check_klass` at all**.
`Association#initialize` (`vendor/rails/activerecord/lib/active_record/associations/association.rb:35-45`) is:

```ruby
def initialize(owner, reflection)
  reflection.check_validity!
  @owner, @reflection = owner, reflection
  reset
  reset_scope
end
```

One call. trails' constructor makes two — `this.checkKlass()` followed by
`validateReflectionValidity(...)` — because the class resolution was split out
of the reflection's own validity check when the NameError had to be raised at a
specific point. Now that `checkKlass` is nothing but `this.klass`, the
`reflection.klass` access that raises it belongs inside `checkValidityBang`,
where `check_validity!` reaches it (`reflection.rb:618-628`,
`klass.composite_primary_key?`).

The method also carries a trails-only early return for
polymorphic/through/anonymous-class associations, and a return type
(`typeof Base | undefined`) that exists only so the getter access is a statement
`@typescript-eslint/no-unused-expressions` accepts.

## Converged shape

- Delete `Association#checkKlass` and its constructor call; the constructor
  makes the single `reflection.check_validity!` call Rails makes.
- The polymorphic/through/anonymous skips move into (or are shown already
  covered by) the macro-specific `checkValidityBang` overrides, which is where
  Rails decides what a given macro validates.
- Verify the NameError still surfaces from `record.association(:name)`
  synchronously — that is the behaviour
  `assoc-check-validity-raises-at-load-not-constructor` and
  `check-validity-in-association-initialize` already pinned.

Rails cite: `associations/association.rb:35-45`, `reflection.rb:618-628`.

## Acceptance criteria

- [ ] No `checkKlass` in the tree; `Association#initialize` calls
      `check_validity!` once and nothing else resolves the class.
- [ ] The unknown-class `NameError` still raises from the constructor, with the
      same message and class.
- [ ] AR association suites green on all three adapter lanes.
