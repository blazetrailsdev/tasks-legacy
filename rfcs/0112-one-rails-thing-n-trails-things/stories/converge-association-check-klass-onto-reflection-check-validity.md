---
title: "converge-association-check-klass-onto-reflection-check-validity"
status: blocked
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-25T15:54:32Z"
assignee: "converge-association-check-klass-onto-reflection-check-validity"
blocked-by: "Depends on unmerged PR #7039 (converge-association-klass-to-reflection-klass-delegate): the story's premise — Association#klass being the bare reflection.klass delegate, and the constructor backing the rich registered reflection — is not in origin/main yet, so checkKlass cannot drop its own re-resolve/derive+constantize fallback without duplicating that PR's work, and editing the same method region would overlap an open sibling branch. Re-ready once #7039 merges."
closed-reason: null
---

## Context

Surfaced while landing `converge-association-klass-to-reflection-klass-delegate`
(PR #7039), which made `Association#klass` the bare `reflection.klass` delegate
Rails writes (`vendor/rails/activerecord/lib/active_record/associations/association.rb:36-38`).

The sibling divergence in the same file is untouched:
`Association#checkKlass`
(`packages/activerecord/src/associations/association.ts`, `protected checkKlass()`)
still carries the shape `klass` just shed — a `_reflectOnAssociation` re-resolve
off the owner's class, wrapped in a `try`/`catch` that swallows `NameError`, then
a `camelize(singularize(name))` derivation + `autoloadModel` + `constantize`
fallback:

```ts
try {
  if (ctor._reflectOnAssociation?.(name)?.klass) return;
} catch (e) {
  if (!(e instanceof NameError)) throw e;
}
const className =
  opts.className ?? camelize(this.reflection.macro === "hasMany" ? singularize(name) : name);
autoloadModel(className);
constantize(className);
```

Rails has no `check_klass` at all. `Association#initialize` runs
`reflection.check_validity!` (association.rb:39), and the NameError for an
unknown class surfaces from the reflection's own `compute_class`
(`reflection.rb:495-508`) reached through `klass` — one derivation, on the
reflection, not a second copy at the holder. The `deriveClassName` helper and
the `autoloadModel`/`constantize` imports in association.ts survive only to feed
this method and `raiseOnTypeMismatchBang`.

PR #7039 also made the constructor back a plain-object definition with the
registered reflection (`_richReflectionFor`), so by the time `checkKlass` runs
`this.reflection` is the rich one — `this.klass` alone should now raise the same
faithful NameError, without the re-resolve or the fallback.

## Acceptance criteria

- [ ] `checkKlass` no longer re-resolves the reflection off the owner's class
      and no longer derives/constantizes a class name; the NameError comes from
      the reflection, as it does in Rails.
- [ ] The `NameError`-only rescue is preserved or shown unnecessary — Rails
      re-raises everything else, notably the "not an ActiveRecord::Base
      subclass" `ArgumentError` (reflection.rb:495-508).
- [ ] `deriveClassName` and the `autoloadModel`/`constantize` imports are
      dropped from association.ts if `raiseOnTypeMismatchBang` no longer needs
      them, or the remaining need is justified at that call site.
- [ ] AR association suites green on all three adapter lanes.
