---
title: "Strict-loading violation check lives in the singular reader, not find_target"
status: blocked
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
deps: []
deps-rfc: []
est-loc: 120
pr: null
claim: "2026-08-19T12:59:52Z"
assignee: "days-into-week-duplicated-in-date-calculations"
blocked-by: "Prerequisite inline-has-many-module-private-find-target-loader (0023, est 350 LOC, status draft, unclaimed as of 2026-08-19) is unlanded. Its triage note says land it first: the trails-only module-private findTarget(record, assocName, options, queryExecutor, violatesStrictLoading) calling convention is why the collection gate cannot see the association instance's _skipStrictLoading, so moving the gate onto Association#findTarget and deleting CollectionProxy._checkStrictLoading's ~13 call sites cannot be done until that loader is folded in. Bundled with days-into-week-duplicated-in-date-calculations; the other four stories shipped without it."
closed-reason: null
---

## Context

Rails raises strict-loading violations from `Association#find_target`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:248-251`,
guard at `:284-291`), so **every** path that issues the association's query
raises — the reader, `load_target`, and anything else that reaches `find_target`.

trails instead puts the check in the singular _reader_
(`packages/activerecord/src/associations/singular-association.ts:210-222`,
`_isStrictOnOwner`) and leaves `findTarget` / `loadTarget` unguarded. Any caller
that loads without going through the reader silently lazy-loads a strict-loaded
owner's association.

Surfaced concretely in PR #5643: the nested-attributes one-to-one writer called
`assoc.loadTarget()` directly and bypassed the check entirely, lazy-loading and
then mutating the child of a strict-loaded owner. That call site was fixed by
routing through `assoc.reader` (which is also the faithful port of Rails'
`existing_record = send(association_name)`), but the underlying placement
divergence is untouched and the next non-reader caller will reproduce it.

Note the placement is not cosmetic: Rails' `violates_strict_loading?` also
consults `owner.validation_context.nil?` and the `@skip_strict_loading` flag,
neither of which the trails reader-side check models.

## Acceptance criteria

- [ ] The strict-loading violation check moves to (or is additionally enforced
      at) trails' `findTarget`, matching `association.rb:248-251`, so a
      non-reader load path cannot bypass it.
- [ ] `violates_strict_loading?`'s `validation_context` and `skip_strict_loading`
      factors are ported or their absence explicitly justified at the call site.
- [ ] A regression test loads a strict-loaded owner's association via a
      non-reader path and asserts it raises; verified failing on the baseline.
- [ ] `strict-loading-sync-reader.test.ts` still passes (the reader-side
      behavior is preserved, not relocated out from under it).

## Absorbed: `collection-strict-loading-gate-cannot-see-skip-flag`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Collection strict-loading gate lives in a loader that cannot see @skip_strict_loading"

### Context

Rails puts the strict-loading gate on the association instance:
`Association#find_target` calls `violates_strict_loading?`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:248-251`,
guard at `:284-291`), which reads the instance's `@skip_strict_loading`.
`Association#skip_strict_loading` (`association.rb:276-282`) raises that flag
for the duration of a block.

trails has BOTH: an instance-level `isViolatesStrictLoading()` on
`Association` (`packages/activerecord/src/associations/association.ts`, which
does honor `_skipStrictLoading`) and a second, independent check inside the
_functional_ has_many loader
(`packages/activerecord/src/associations/has-many-association.ts:~550`,
`_violatesStrictLoading(record, options) && _findTargetReachable(...)`). The
functional loader takes the trails-only `(owner, assocName, options)` triple
rather than an association instance, so it cannot see `_skipStrictLoading` — and
the instance-level check is not the one that fires on the collection load path.

PR #5767 hit this porting `concat`'s `skip_strict_loading { load_target }`:
wrapping the call and widening `skipStrictLoading` to `protected` was NOT
enough, because the raise came from the functional loader. The fix threaded the
flag through as a 5th positional parameter
(`has-many-association.ts:169-174` → `:517` → `:550`). That parameter is pure
trails surface with no Rails counterpart, and it only covers the one caller that
remembered to pass it — every other functional-loader entry point still bypasses
`@skip_strict_loading` silently.

Related but distinct from
[[strict-loading-check-in-reader-not-find-target]], which covers the _singular_
side (check in the reader instead of `find_target`). This one is the collection
side: the check exists in a loader that structurally cannot see the association
instance's flag.

### Acceptance criteria

- [ ] The collection strict-loading gate consults the association instance's
      `_skipStrictLoading` without a threaded parameter — either by moving the
      check onto `HasManyAssociation#findTarget` (the Rails-shaped entry point)
      or by giving the functional loader access to the holder.
- [ ] The trails-only `skipStrictLoading` parameter on the functional
      `findTarget` (`has-many-association.ts:517`) is deleted, not merely
      defaulted.
- [ ] `strict-loading.test.ts` still passes in full, including
      `strict loading with new record on concat is ignored` (the #5767
      regression) — verified failing on a baseline that drops the param.
- [ ] Any other functional-loader entry point that should honor
      `skip_strict_loading` is covered by the same mechanism rather than
      case-by-case.

## Absorbed: `audit-collection-proxy-strict-loading-call-sites`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "Audit CollectionProxy.\_checkStrictLoading call sites against Rails find_target"

### Context

`CollectionProxy._checkStrictLoading()` (`collection-proxy.ts:1130`) raises
`StrictLoadingViolationError` from ~13 proxy methods (1766, 1851, 1867, 1883,
1899, 3141, 3151, 3163, 3175, 3489, 3512, 4009 and, until PR #5910, `find`).

Rails raises strict-loading violations from exactly one place on this path:
`Association#find_target` (`activerecord/lib/active_record/associations/association.rb`),
which checks `violates_strict_loading?` before querying. Methods that go
through `scope.<something>` rather than `find_target` — `scope.find`,
`scope.pluck`, `scope.pick`, and friends — do NOT raise in Rails.

PR #5910 removed the `find` call site while converging `CollectionProxy#find`
into Rails' one-line delegation to `@association.find`: Rails' `find` reaches
`scope.find(*args)`, never `find_target`, and neither Rails' nor trails' test
suite covered `proxy.find` under strict loading (checked `strict_loading_test.rb`
and all of `strict-loading*.test.ts`). The remaining call sites were out of
that PR's scope and are unexamined — each is either a genuine Rails path
through `find_target` or the same invention `find` had.

### Acceptance criteria

- Each remaining `_checkStrictLoading()` call site in `collection-proxy.ts` is
  audited against Rails: either the Rails method reaches `find_target` (keep,
  and note which Rails frame does the raising), or it does not (remove).
- Any site whose removal changes observable behaviour is covered by a test
  named after the Rails test that pins it, if one exists; if Rails has no test,
  the behaviour follows the Rails source and the trails-only assertion says so.
- If the audit finds the checks are load-bearing for a trails-specific reason
  (e.g. a sync reader with no Rails analogue), that reason is recorded at the
  call site rather than left implicit.

## Triage note (2026-08-18)

Merged story (~300 LOC est.). The three absorbed rows are one deviation seen
from three angles: the gate belongs on `Association#find_target`
(`association.rb:247-273`, `violates_strict_loading?` at `:284-291`), reading
the instance's `@skip_strict_loading` (`:276-282`).

**Prerequisite, not part of this story:**
`inline-has-many-module-private-find-target-loader` (0023, ~350 LOC) folds the
module-private `findTarget(record, assocName, options, queryExecutor,
violatesStrictLoading)` loader into `HasManyAssociation#findTarget`. That
trails-only owner/name/options calling convention is _why_ the gate cannot see
`@skip_strict_loading` today. Land it first; this story then becomes a move plus
the deletion of `CollectionProxy._checkStrictLoading`'s ~13 call sites.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
