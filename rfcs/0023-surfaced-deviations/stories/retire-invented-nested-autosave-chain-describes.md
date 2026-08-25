---
title: "Retire the invented TestHasMany/HasOneAutosaveAssociationWhichItselfHasAutosaveAssociations describes and their Gc*/Gg* models"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/autosave-association.test.ts` contains two describe
blocks whose names do not exist in Rails:

- `TestHasManyAutosaveAssociationWhichItselfHasAutosaveAssociations` (~line 3453)
- `TestHasOneAutosaveAssociationWhichItselfHasAutosaveAssociations` (~line 3796)

`awk '/class Test/' vendor/rails/activerecord/test/cases/autosave_association_test.rb`
lists 22 test classes; neither of these is among them. Their models are
invented too: the first gives Pirate `has_many :ships` and the second gives
Ship `has_one :part`, whereas Rails' canonical `Pirate` has `has_one :ship`
(models/pirate.rb) and `Ship` has `has_many :parts` (models/ship.rb). Because
no canonical model has those shapes, PR #5250 could only rename the classes to
`Gc*` / `Gg*` rather than converge them — the bespoke-model debt is structural,
not cosmetic.

Rails does cover the nested-autosave chain, but through
`TestDestroyAsPartOfAutosaveAssociation`
(`autosave_association_test.rb:1182`) using the real
pirate → ship → parts chain.

## Acceptance criteria

- Read `autosave_association_test.rb:1182+` and identify which Rails tests, if
  any, these invented describes are standing in for.
- Behaviour that Rails covers is moved under the Rails-verbatim describe name
  using the canonical `Pirate`/`Ship`/`ShipPart` chain; the `Gc*` / `Gg*`
  bespoke models are deleted with it.
- Behaviour Rails genuinely does not cover either moves to
  `autosave-association.trails.test.ts` or is dropped with a note saying why.
- No surviving test is renamed to something Rails does not have.
- `pnpm vitest run packages/activerecord/src/autosave-association.test.ts`
  stays green.
