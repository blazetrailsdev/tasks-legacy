---
title: "@association_cache holds ad-hoc object literals Rails has no counterpart for"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 40
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two trails call sites seed `@association_cache` with a minimal object literal
that is not an `Association` at all — it implements only `target`,
`_explicitTarget`, `isLoaded()` and `setTarget()`:

- `packages/activerecord/src/associations.ts:599-614` (`_cacheSingularTarget`'s
  fallback for an inverse name with no declared singular reflection)
- `packages/activerecord/src/support/seed-association-cache.ts:35-46` (the
  `catch` arm for an undeclared name)

Rails' `@association_cache` only ever holds `Association` instances built by
`Base#association` from a reflection
(`vendor/rails/activerecord/lib/active_record/associations.rb:290-296`), and
`set_inverse_instance` (`association.rb:170-176`) writes through
`inversed_from` on such an instance. There is no "ad-hoc holder" shape.

These holders are why callers must probe defensively —
`syncAssociationInstance` (`associations/instance-methods.ts:53-64`) calls
`isCollection?.()` optionally "because `_associationInstances` also holds
minimal ad-hoc holders", and PR #6685's `CollectionProxy` constructor does the
same. They were also named as the root cause in
`retire-collection-proxy-own-seat`.

## Converged shape

- Every `@association_cache` entry is an `Association` instance built from a
  reflection, as `associations.rb:290-296` builds them.
- An inverse name with no declared reflection is not cached at all (Rails
  cannot reach that state: `inverse_of` resolution yields a reflection or
  nothing — `reflection.rb`'s `inverse_of`), rather than cached under an
  invented shape.
- The optional `isCollection?.()` probes at the call sites above become plain
  `isCollection()` calls once no non-Association can be in the map.

## Acceptance criteria

- [ ] No object literal is written into `_associationInstances`.
- [ ] `isCollection?.()` optional probes over `_associationInstances` entries
      become unconditional.
- [ ] Association, inverse, preload, autosave and nested-attributes suites stay
      green on SQLite, PostgreSQL and MySQL/MariaDB.

## Absorbed: `homogenize-association-instances-cache`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Stop caching ad-hoc non-Association holders in \_associationInstances"

### Context

`_associationInstances` is documented as the canonical `Association` cache
(Rails' `@association_cache`), but trails also stores non-`Association` literals
in it for undeclared inverses — an ad-hoc holder exposing only
`target` / `_explicitTarget` / `isLoaded` / `setTarget`:

- `packages/activerecord/src/associations.ts` (the undeclared-inverse branch of
  the inverse-seeding helper)
- `packages/activerecord/src/support/seed-association-cache.ts`

Rails has no such shape: `@association_cache` holds `Association` instances
only, and an ad-hoc inverse simply is not cached. The heterogeneous map forces
every consumer to probe defensively (`isCollection?.()`, `_mergeLoaderResults?.()`,
`isLoaded?.()`, `reset?.()`), and a single missing `?.` is a runtime
`TypeError` on a path with no local test coverage — exactly what broke CI on
PR #5461 (`instance.isCollection is not a function`, reached indirectly through
validation error-message generation).

### Acceptance criteria

- [ ] Undeclared inverses either get a real `Association` instance or are not
      cached in `_associationInstances` at all — the map holds one type.
- [ ] The defensive optional-call probes on that map
      (`isCollection?.()`, `_mergeLoaderResults?.()`, `isLoaded?.()`, `reset?.()`)
      become plain calls, or the sites are documented as needing them for a
      different reason.
- [ ] `validations/absence-validation.test.ts` (the suite that caught the
      original TypeError) and the FakeTopic/FakeReply fixture paths stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
