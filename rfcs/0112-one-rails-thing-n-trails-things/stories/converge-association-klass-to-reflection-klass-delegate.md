---
title: "Make Association#klass the plain reflection.klass delegate Rails has"
status: in-progress
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 100
pr: 7039
claim: "2026-08-25T14:34:31Z"
assignee: "converge-association-klass-to-reflection-klass-delegate"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in review of PR #6428 (RFC 0084,
`converge-has-many-delete-records-reflection-klass-fallback`).

Rails' `Association#klass` is a plain delegate:

```ruby
def klass
  reflection.klass
end
```

(`vendor/rails/activerecord/lib/active_record/associations/association.rb:36-38`)

The trails port (`packages/activerecord/src/associations/association.ts`,
`get klass()`) is not that delegate. It re-resolves the reflection off the
owner's class and then falls back to a name derivation:

```ts
get klass(): typeof Base {
  const richKlass = ctor._reflectOnAssociation?.(this.reflection.name)?.klass;
  if (richKlass) return richKlass;
  const className = this.reflection.options.className ?? this.deriveClassName();
  autoloadModel(className);
  return constantize(className) as typeof Base;
}
```

Both arms are machinery Rails does not have here — Rails' namespace-relative
`compute_type` walk lives on the reflection (`reflection.rb:495-508`), reached
through `reflection.klass`. The re-resolve exists because an ad-hoc holder
built by `_buildAssociationInstance` carries the thin macro definition rather
than the registered reflection; the `deriveClassName`/`constantize` arm is a
second copy of the reflection's own derivation.

PR #6428 removed the last call-site fallback that leaned on this getter
(`HasManyAssociation#deleteRecords` now reads `reflection.klass` literally),
so the getter itself is the remaining divergence.

## Converged shape

`Association#klass` is `return this.reflection.klass;` and nothing else, with
the derivation left to the reflection. Rides on the ad-hoc holders carrying a
rich reflection — the same root cause as
[[converge-association-reflection-type-drop-association-definition]].

## Acceptance criteria

- [ ] `Association#klass` is a bare `reflection.klass` delegate matching
      `association.rb:36-38`.
- [ ] No call site re-adds a derivation fallback in its place.
- [ ] AR association suites green on all three adapter lanes.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
