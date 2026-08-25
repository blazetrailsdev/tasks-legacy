---
title: "Converge NoTouching.no_touching block onto apply_to (drop duplicate push/pop)"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 30
pr: null
claim: "2026-08-25T17:06:37Z"
assignee: "converge-no-touching-block-onto-apply-to"
blocked-by: null
closed-reason: null
---

## Context

Rails `NoTouching::ClassMethods#no_touching`
(`vendor/rails/activerecord/lib/active_record/no_touching.rb:23`) is a one-line
delegation: `NoTouching.apply_to(self, &block)`. `apply_to` (line 30) pushes the
class onto the `klasses` stack and pops it in an `ensure`.

trails `packages/activerecord/src/no-touching.ts` has both: `noTouching()`
(line 17) reimplements the push/pop against `_noTouchingDepth` inline, and
`applyTo()` (line 62 after PR #5923) is a second, unreferenced copy of the same
push/pop. Nothing imports `applyTo` — `Base.noTouching` (base.ts) calls
`noTouching` directly. So one Ruby method has two TS implementations, one dead.

Surfaced while landing PR #5923 (converge `no_touching?` to an instance
predicate), which removed the sibling dead `touchLater` from the same file.

## Acceptance criteria

- `noTouching(modelClass, fn)` delegates to `applyTo`, matching
  no_touching.rb:23 (one call, no inlined depth bookkeeping).
- Exactly one push/pop implementation remains in `no-touching.ts`.
- `timestamp.test.ts` / `touch-later.test.ts` still pass; parity:api for
  `no_touching.rb` stays at 6/6 matched.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
