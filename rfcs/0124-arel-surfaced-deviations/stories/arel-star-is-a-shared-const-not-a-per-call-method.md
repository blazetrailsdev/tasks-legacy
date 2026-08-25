---
title: "Arel.star is a shared const; Rails allocates a fresh SqlLiteral per call"
status: draft
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

Surfaced while moving the bare `Arel.*` module functions into
`packages/arel/src/arel.ts` (PR #6357).

Rails (`vendor/rails/activerecord/lib/arel.rb:59-61`):

```ruby
def self.star # :nodoc:
  sql("*", retryable: true)
end
```

It is a METHOD returning a fresh `SqlLiteral` on every call. trails has a
module-level constant:

```ts
export const star = sql("*", { retryable: true });
```

so every `Arel.star` in the codebase hands back the SAME node instance.
PR #6357 converged the `retryable: true` half (it was missing) but left the
const, and noted the deviation in the JSDoc.

Why it is worth converging rather than leaving: `SqlLiteral` is not frozen in
trails, and nodes are mutated in place elsewhere in this package — PR #6357's
own `visitArelNodesSelectStatement` work assigns `o.limit` on the node it is
visiting, and `visit_Arel_Nodes_SelectCore` assigns `o.froms`. A single shared
`star` instance is one in-place mutation away from a cross-query bug that
would be very hard to trace, and Ruby's per-call allocation is precisely what
makes that impossible upstream. It also blocks `parity:api` from ever seeing
`star` as the method Rails defines.

## Converged shape

```ts
export function star(): SqlLiteral {
  return sql("*", { retryable: true });
}
```

and update the call sites to `star()`. Count them first — `Arel.star` is read
across `arel` and `activerecord` — this is a mechanical rename plus a call
suffix, so the LOC is mostly call sites, not logic.

## Acceptance criteria

- `star` is a function matching `arel.rb:59-61`, allocating per call.
- Every `Arel.star` / `star` reader is updated to call it; no lingering
  const re-export from `index.ts`.
- The `@noRailsEquivalent`-style justification in `arel.ts`'s JSDoc for the
  const is DELETED, not reworded.
- `pnpm parity:api --package arel` delta non-negative.
