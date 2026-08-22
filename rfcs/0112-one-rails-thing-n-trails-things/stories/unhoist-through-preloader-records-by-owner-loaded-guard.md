---
title: "Return ThroughAssociation#recordsByOwner's loaded? guard to Rails' per-owner arm"
status: closed
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps:
  - converge-through-preloader-records-by-owner-onto-public-reader
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Delivered by PR #6826 (17faad119, 'converge records_by_owner control flow'). On origin/main packages/activerecord/src/associations/preloader/through-association.ts:56-98 recordsByOwner is a single loop over owners with the per-owner 'if (this.isLoaded(owner)) { result.set(owner, this.targetFor(owner)); continue; }' arm matching through_association.rb:12-15; the hoisted whole-loop early return and the 'owners.length > 0' guard are both gone (git grep 'owners.length > 0' returns nothing in that file). Prereq converge-through-preloader-records-by-owner-onto-public-reader is done."
---

## Context

Surfaced while landing PR #6771.

Rails' `Preloader::ThroughAssociation#records_by_owner`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/through_association.rb:11-37`)
is one `each_with_object` over `owners` with a per-owner `next` for the loaded
case:

```ruby
@records_by_owner ||= owners.each_with_object({}) do |owner, result|
  if loaded?(owner)
    result[owner] = target_for(owner)
    next
  end
  ...
```

trails hoists that per-owner guard into a whole-loop early return
(`packages/activerecord/src/associations/preloader/through-association.ts`,
`recordsByOwner`):

```ts
if (this.owners.length > 0 && this.owners.every((owner) => this.isLoaded(owner))) {
  for (const owner of this.owners) {
    result.set(owner, this.targetFor(owner));
  }
  this._recordsByOwner = result;
  return result;
}
```

The comment at the call site justifies the hoist as a loop-invariant: the
per-owner `next` cannot avoid the _unconditional_ child fetch that trails does
above the loop, so the guard had to move up to the one path that could await.
That premise no longer holds. PR #6771 made the child readers awaitable, so
the fetch can move back inside the loop where Rails has it — and once the merge
helpers force their own children (see
`converge-through-preloader-records-by-owner-onto-public-reader`), there is no
unconditional pre-fetch left to guard against.

The hoist is not behaviour-neutral: an `owners` list that is _partly_ loaded
takes the slow path in both, but a fully-loaded list skips the source-type
filtering and the order/distinct handling that Rails' per-owner `next` also
skips — the shapes agree today only because the two arms happen to coincide.
It also adds an `owners.length > 0` guard Rails does not have.

Note the same body carries a second deviation: `throughLoadedOnFirst` wraps
Rails' `owners.first.association(through_reflection.name).loaded?`
(`through_association.rb:19`) in a `try/catch` returning `false`. That belongs
to `preloader-association-readers-swallow-errors-in-bare-catch` (0023) — do not
duplicate it here, but converging this body is the natural place to retire it.

## Converged shape

`recordsByOwner` is Rails' single loop over `owners` with the per-owner
`loaded?` → `next` arm, no hoisted whole-loop early return and no
`owners.length > 0` guard, with the child records forced by the merge helpers
they are read from rather than by a pre-pass.

## Acceptance criteria

- [ ] `ThroughAssociation#recordsByOwner` has no whole-loop early return; the
      `loaded?` check is the per-owner `continue` at the top of the loop, as in
      `through_association.rb:12-15`.
- [ ] No `owners.length > 0` guard Rails does not have.
- [ ] Depends on / lands after
      `converge-through-preloader-records-by-owner-onto-public-reader`, which
      removes the unconditional pre-fetch this hoist was working around.
- [ ] `pnpm parity:api:calls` / `:args` green; the through-preloader,
      eager-loading and nested-through suites pass with no test renames.
