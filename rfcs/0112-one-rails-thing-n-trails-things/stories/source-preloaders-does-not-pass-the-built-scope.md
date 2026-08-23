---
title: "Preloader::ThroughAssociation#source_preloaders derives a scope instead of passing scope"
status: claimed
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: "2026-08-23T02:57:28Z"
assignee: "top-level-function-missing-rails-call-tag-does-not-suppress"
blocked-by: null
closed-reason: null
---

## Context

`Preloader::ThroughAssociation#source_preloaders`
(`activerecord/lib/active_record/associations/preloader/through_association.rb:70-72`)
is one line and hands the source Preloader **this** preloader's built `scope`:

```ruby
def source_preloaders
  @source_preloaders ||= ActiveRecord::Associations::Preloader.new(records: middle_records, associations: source_reflection.name, scope: scope, associate_by_default: false).loaders
end
```

trails' `sourcePreloaders`
(`packages/activerecord/src/associations/preloader/through-association.ts:189-230`)
never calls `scope`. It derives a substitute out of `reflectionScope` /
`preloadScope`, emptying the source `whereClause` when `throughScope` has
already copied the full where onto the through query and eager-loaded the
source there. That is why
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/preloader/through-association.json`
still carries the `source_preloaders` -> `scope` row after PR #6890 migrated
that shard's `any?` and `first` rows to `@missingRailsCall` receipts — the row
was deliberately retained because the deviation is owned convergence work, not
a language shortcoming (RFC 0106 rule at
`scripts/api-compare/missing-rails-call-tags.ts:296-299`).

The ~30-line block comment at the call site is itself the tell: it explains a
policy Rails does not have. Rails does not need it because its `through_scope`
(`through_association.rb:96-125`) does not double-apply the where in the first
place — so the real divergence is probably upstream, in `throughScope`, and
this is where it surfaces.

RFC 0040 (through-association source convergence) is closed, so this is filed
here. Related:
separate `throughScope` divergence, and
`through-source-scope-not-merged-in-eager-preload` (RFC 0040, done) touched the
same seam.

## Converged shape

`sourcePreloaders` passes `this.scope()` — the built scope — the way
`through_association.rb:71` does, with no where-emptying policy of its own. If
that reds the eager-loaded-source cases the comment describes, the fix belongs
in `throughScope` (`through_association.rb:96-125`) so the where is not applied
twice, not in a compensating filter here.

Delete the `source_preloaders` -> `scope` row from
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/preloader/through-association.json`
by hand (only-shrink; do not reseed), and run
`pnpm parity:api:calls:tighten activerecord/associations/preloader/through-association.json`
if the mark goes stale. The two `kind: "args"` `reduce` rows in that shard are
a separate language shortcoming and stay.

## Acceptance criteria

- [ ] `sourcePreloaders` calls `scope`, matching `through_association.rb:70-72`.
- [ ] The where-emptying policy and its block comment are gone from this method;
      any real fix lands in `throughScope`.
- [ ] The `source_preloaders` -> `scope` baseline row is deleted, not reworded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
