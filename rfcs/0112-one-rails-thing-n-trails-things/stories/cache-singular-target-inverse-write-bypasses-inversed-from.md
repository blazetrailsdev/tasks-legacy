---
title: "_cacheSingularTarget's surviving arm writes the inverse target without set_inverse_instance"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: 6981
claim: "2026-08-24T13:05:17Z"
assignee: "migrator-pending-migrations-must-not-create-schema-table"
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing #6975 (`association-cache-holds-only-association-instances`),
which removed the ad-hoc object literals from `@association_cache`. What is left
in `_cacheSingularTarget`
(`packages/activerecord/src/associations.ts`, the arm after the
`belongsTo`/`hasOne` early return) is a second, non-Rails write path: for an
inverse name whose cached entry already exists, it writes

```ts
existing._setTargetFromLoader(target);
existing._explicitTarget = true;
```

Rails has exactly one inverse write: `set_inverse_instance`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:170-176`)
→ `inversed_from` (`:180-184`), which is what the singular arm of this same
function already calls. There is no `_setTargetFromLoader`, no
`_explicitTarget`, and no `_loadedFromPreload` on Rails' `Association` — those
three are trails-only fields read by `_loadedSingularTarget` and
`_preloadedHolderTarget` in the same file. Rails distinguishes those cases with
`@loaded` alone (`association.rb:`, `loaded?` / `loaded!`), plus
`CollectionAssociation#@target`.

## Converged shape

- `_cacheSingularTarget`'s remaining arm routes through `inversedFrom` (or the
  collection equivalent Rails uses, `association.rb:187-196`), not a bespoke
  loader-write.
- `_explicitTarget` / `_loadedFromPreload` are retired in favour of Rails'
  `loaded?` state, or each surviving one carries a `@noRailsEquivalent` with a
  language-forced reason.

## Acceptance criteria

- [ ] No trails-only write path onto an `Association` target remains in
      `_cacheSingularTarget`.
- [ ] The inverse-of, preload and autosave suites stay green on SQLite,
      PostgreSQL and MySQL/MariaDB.
