---
title: "activemodel/attribute-methods.ts is out of Rails source order on main"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`packages/activemodel/src/attribute-methods.ts` on `main` is not in the order
`blazetrails/rails-file-structure-method-order` requires, so running
`pnpm lint --fix` on a pristine checkout of that file rewrites it:

````console
git show origin/main:packages/activemodel/src/attribute-methods.ts > <file>
npx eslint <file> --fix
# 32 insertions(+), 47 deletions(-)
```console

Verified on `origin/main` during trails#6818 (no local edits involved). The rule
is autofixable and non-fixing lint passes, so CI is green and the drift is
invisible — until someone adds a top-level function to the file, at which point
lint-staged autofixes the *whole* file and the PR picks up ~80 lines of
unrelated block movement (the "Class-level Rails privates" and "Instance-level
Rails privates" banners plus `NAME_COMPILABLE_REGEXP`, `InstanceHost` and
`InstanceMethods` all relocate). That churn was backed out of #6818 by hand.

The rule's expected order comes from
`vendor/rails/activemodel/lib/active_model/attribute_methods.rb` via the
manifest `pnpm parity:api` builds (`eslint/rails-file-structure-method-order.json`),
so the fix is to bring the file to that order in one dedicated commit that
changes nothing else.

## Converged shape

`packages/activemodel/src/attribute-methods.ts` matches Rails' member order, so
`npx eslint packages/activemodel/src/attribute-methods.ts --fix` is a no-op on a
clean tree. Pure movement — no body edits.

Worth checking whether other files carry the same latent drift:
run `pnpm lint --fix` on a clean checkout and see what moves.

## Acceptance criteria

- [ ] `eslint --fix` on `attribute-methods.ts` from a clean tree produces no
      diff.
- [ ] The commit is movement-only: no member body changes, no renames.
- [ ] Any other files found with the same latent drift are listed (fixed here if
      small, filed as siblings if not).
- [ ] `pnpm parity:api` deltas non-negative; ActiveModel attribute-method tests
      green.
````
