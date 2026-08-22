---
title: "addConstraints guards reflection.constraints() and the lambda type where Rails guards neither"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`AssociationScope#addConstraints`
(`packages/activerecord/src/associations/association-scope.ts`) mirrors
`vendor/rails/activerecord/lib/active_record/associations/association_scope.rb:124-159`.
PR #6886 inlined its three invented helpers back into the one Rails loop body,
so the decomposition now matches. Two invented GUARDS survived that inlining
and are the remaining divergence in the same loop:

```ts
const constraints =
  (reflection as { constraints?: () => Array<...> }).constraints?.() ?? [];
for (const scopeChainItem of constraints) {
  if (typeof scopeChainItem !== "function") continue;
```

Rails writes `reflection.constraints.each do |scope_chain_item|`
(association_scope.rb:132-133) with neither guard:

- `constraints` is declared on every reflection Rails puts in a chain —
  `AbstractReflection#constraints` (`reflection.rb`), and
  `PolymorphicReflection#constraints` / `RuntimeReflection` alike — so the
  optional call plus `?? []` covers a shape Rails says cannot occur, and would
  silently produce an EMPTY constraint list (no scope lambdas applied at all)
  where Rails would raise `NoMethodError`.
- Every element of `constraints` is a scope lambda, so the
  `typeof !== "function"` filter likewise silently skips a chain item Rails
  would `instance_exec`.

Both are the classic "silently drop what Rails raises on" shape: they cannot
make a green suite red, but they turn a wiring bug in `constraints()` into a
missing WHERE clause instead of an exception.

The same optional-call idiom is on the `DisableJoinsAssociationScope` copy
(`disable-joins-association-scope.ts`, mirroring
disable_joins_association_scope.rb:41), and should converge with it.

## Converged shape

Type `constraints()` as required on the reflection interface the loop reads
(the union `AbstractReflection | ReflectionProxy` — `ReflectionProxy` already
declares it, association-scope.ts:175), then write Rails' two lines:

```ts
for (const scopeChainItem of reflection.constraints()) {
  const item = this.evalScope(reflection, scopeChainItem, owner);
```

If a chain member genuinely lacks `constraints()`, that is a bug in `getChain`
/ the reflection port to fix at its source, not to absorb here.

## Acceptance criteria

- [ ] `addConstraints` calls `reflection.constraints()` unconditionally, with no
      `?? []` fallback and no `typeof === "function"` filter.
- [ ] Same for the `DisableJoinsAssociationScope#addConstraints` loop.
- [ ] The reflection type the loop reads declares `constraints()` as required,
      so the removal is type-checked rather than cast away.
- [ ] `packages/activerecord/src/associations/` suite green on SQLite, PG and
      MySQL/MariaDB.
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
