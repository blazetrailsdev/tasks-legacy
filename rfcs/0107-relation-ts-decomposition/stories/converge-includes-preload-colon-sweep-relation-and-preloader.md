---
title: "Sweep includes/preload call sites onto the colon spelling: relation, preloader and test-helpers"
status: ready
updated: 2026-08-22
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Cluster split of `sweep-includes-preload-call-sites-onto-the-colon-symbol-spelling`,
which is ~800 literal call sites across 131 files in `packages/activerecord/src` —
far past the PR ceiling as one change. `sweep-joins-call-sites-onto-the-colon-symbol-spelling`
(PR #6704) did the same job for `joins` / `leftOuterJoins` at ~660 LOC across 52
files, which is the size one cluster should land at.

Rails passes association names to `includes` / `preload` / `eager_load` /
`references` as Symbols (`relation/query_methods.rb`, e.g. `:88-101`), and
CLAUDE.md spells a Ruby Symbol as a colon-prefixed string. trails' `joins` values
now carry that spelling while the includes family does not, so the two value sets
disagree where Rails has Symbols on both sides.

This story covers the `relation/`, `associations/preloader/` and `test-helpers/` trees under `packages/activerecord/src` (~120 literal call sites).

Depends on `converge-preloader-branch-colon-symbol-entry-point`, which adds the
one colon strip at `Preloader::Branch#_normalizeAssociationName` (mirroring
`associations/join-dependency.ts:933`) that makes the colon spelling resolve.

## Converged shape

Every `includes` / `preload` / `eagerLoad` / `references` call site in scope that
passes an association NAME passes it colon-prefixed — including nested-hash and
array forms, keys and values alike. Raw strings naming a TABLE for `references`
stay bare: Rails passes those as Strings (`query_methods.rb`, `references!`).

## Acceptance criteria

- [ ] Every in-scope association-name call site uses the colon spelling.
- [ ] No new normalization site is introduced anywhere; the strip stays at the
      `JoinDependency` / `Preloader::Branch` entry points.
- [ ] Generated SQL unchanged on all three adapters; no test name touched.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative;
      `parity:api:calls` / `:args` clean.
