---
title: "Delete the dead, nested-through-divergent transaction() in through-association.ts"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/associations/through-association.ts:16-28` defines a
module-local `function transaction(assoc, block)` that is **never exported and
never referenced anywhere in the file or the package** — `grep -rn "transaction"
through-association.ts` returns only its own definition and body. Surfaced while
converging `CollectionProxy#_replaceTransaction` onto the association in PR 6753.

It is dead **and** divergent, which is why it should go rather than be wired up:

- Rails' `ThroughAssociation#transaction`
  (`vendor/rails/activerecord/lib/active_record/associations/through_association.rb:10-12`)
  is `through_reflection.klass.transaction(&block)`, where
  `through_reflection` (`through_association.rb:14-23`) walks **up** the nested
  chain: `refl = reflection.through_reflection; while refl.through_reflection?;
refl = refl.through_reflection; end`.
- The dead trails function skips that walk entirely — it reads
  `reflection.options.through` off the owner class' reflection registry once, so
  a nested-through would resolve the wrong join model.
- The live, correct implementation already exists at
  `packages/activerecord/src/associations/has-many-through-association.ts:140-143`,
  which goes through `throughReflection()` and overrides
  `CollectionAssociation#transaction`
  (`collection_association.rb:321-323`, ported at
  `collection-association.ts:426-430`). That is the single site Rails has, and
  the one every caller now reaches.

Leaving a second, wrong-for-nested-through derivation in the file that _names_
Rails' `ThroughAssociation` invites a future caller to pick it up.

## Converged shape

Delete `function transaction` from `through-association.ts`. Rails has one
`ThroughAssociation#transaction`; trails' one site is the
`HasManyThroughAssociation` override, which already carries the Rails name, the
`through_reflection` walk-up, and the `through_reflection.klass.transaction`
body.

Verify no caller appears in the interim (`grep -rn` across
`packages/activerecord/src`), and confirm `HasOneThroughAssociation` also
resolves its transaction through the same override rather than through this
function — if it does not, that is the one caller to route onto the override
before deleting.

## Acceptance criteria

- [ ] `function transaction` no longer exists in `through-association.ts`.
- [ ] No new `@noRailsEquivalent` tag and no new baseline row.
- [ ] `pnpm parity:api:calls` / `:args` add zero rows; `pnpm parity:api:extra
--package activerecord` gains nothing.
- [ ] `has-many-through-associations.test.ts`,
      `has-one-through-associations.test.ts`,
      `nested-through-associations.test.ts` pass unchanged. No test renamed.
