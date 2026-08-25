---
title: "Barrel renames Hash#compact_blank to compactBlankObj to dodge the Enumerable collision"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/index.ts` exports Hash's `compact_blank`
(`core_ext/enumerable.rb:222-224`, in `hash-utils.ts`) under the invented
name `compactBlankObj`:

```ts
compactBlank as compactBlankObj,
```

The bare `compactBlank` is already taken at the barrel by Enumerable's
`compact_blank` (`core_ext/enumerable.rb:184-186`, in `enumerable-utils.ts`).
Ruby has both — they are distinct methods on distinct receivers — but one flat
ESM namespace cannot hold two exports of one name, so the barrel renames one of
them. `Obj` is not a suffix any Ruby name or `docs/ruby-ts-conventions.md` rule
produces.

PR #6556 converged the SOURCE file (`hash-utils.ts` now declares
`compactBlank`, which is what `parity:api` matches on) and left the barrel
alias, since removing a public export is a separate call. The alias is the
residue.

This is one instance of a file-wide question: the same collision will recur for
every Ruby name that exists on two receivers and is ported as a free function in
two modules.

## Converged shape

Decide the barrel's rule for receiver-collided names and apply it here. The two
candidates:

1. **Drop the flat alias** and require callers to import from the module that
   matches the receiver (`hash-utils.js` vs `enumerable-utils.js`) — the barrel
   then exports only one `compactBlank`, and the other has a real import path.
2. **Namespace the barrel by receiver** (a `Hash` / `Enumerable` export object),
   which is closer to how Ruby actually disambiguates them.

Either way the invented `Obj` suffix goes. Whichever is chosen should be written
down so the next collision does not invent a third convention — check whether
any other `*Obj`/`*Hash`/`*Arr`-suffixed barrel alias already exists and fold
those into the same move.

## Acceptance criteria

- [ ] No `compactBlankObj` (or equivalently-suffixed) name in the barrel.
- [ ] The chosen disambiguation rule is stated where the next porter will find
      it (`docs/ruby-ts-conventions.md` or `scripts/parity/conventions.ts`).
- [ ] `pnpm parity:api:extra --package activesupport` shows no new surface.
