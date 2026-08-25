---
title: "Privates manifest omits a method-less Rails entity from the entities map"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Follow-up on PR #7057, which gave `eslint/rails-private-methods.json` an
`entities` map so `rails-private-jsdoc` can tell a Rails-mirroring declaration
from a local helper type sharing the TS file.

The map is populated inside the `for (const [rubyFile, names] of fileVis)` loop
in `scripts/build-rails-privates-manifest.ts`, and `fileVis` only gains a key
when `note()` fires — i.e. when some entity contributes at least one method to
that Ruby file. A Rails entity with ZERO methods therefore never reaches
`fileEntities`, and its TS class name is absent from `entities[tsRel]`.

The reviewer traced this on #7057 and it is not a live bug today: if no entity
in the file contributes a method, `files[tsRel]` carries no gated name either,
so `manifestTags` returns false by the second check regardless. It becomes
observable the moment a file mixes a method-less Rails entity with a
method-bearing sibling — the method-less entity's members would be silently
un-gated where the file-wide union still lists the name.

## Converged shape

Populate `fileEntities` from the entity walk itself rather than from the
`fileVis` iteration: `noteEntity(host.file, host.fqn)` already runs
unconditionally per host in `visit()`, so the fix is to emit
`manifest.entities[tsRel]` for every key in `fileEntities`, not only for keys
that also appear in `fileVis`. Keep the intersection semantics of `files`
untouched — this changes which entity NAMES are recognized, never which method
names are gated.

Add a builder-level regression that a method-less entity sharing a file with a
method-bearing one still appears in `entities`.

## Acceptance criteria

- [ ] A Rails entity contributing no methods still appears in its file's
      `entities` list.
- [ ] `scripts/api-compare/config.test.ts` and the `rails-private-jsdoc` rule
      tests pass, with a new case covering the method-less entity.
- [ ] `Rails API/Test Comparison` green; no package's `@internal` requirements
      change as a side effect.
