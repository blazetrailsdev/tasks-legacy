---
title: "Base re-declares ActiveModel::Access's slice/values_at"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

`packages/activerecord/src/base.ts:4480-4481` re-declares `slice` and
`valuesAt` on `interface Base`:

```ts
slice(...keys: string[]): HashWithIndifferentAccess<unknown>;
valuesAt(...keys: string[]): unknown[];
```

Both come from `ActiveModel::Access` (access.rb:8, :12), which `Base` already
has through `Model` — Rails' `ActiveRecord::Base` declares neither, and gets
them from `include ActiveModel::Access` in `model.rb:44` by way of
`ActiveModel::API`. The re-declaration is not a port of anything in
`activerecord/lib/active_record/base.rb`.

It is also actively harmful: because it restates the signature rather than
inheriting it, it drifts. PR #7010 changed `Access#slice`'s return type to
`HashWithIndifferentAccess` (access.rb:9's `.with_indifferent_access`) and the
stale `Record<string, unknown>` here broke the build across 46 files until it
was retyped by hand — a change confined to `activemodel` should not have
required an `activerecord` edit at all.

`base.ts`'s `interface Base` carries a long run of such re-declarations
(`assignAttributes`, `updateAttribute`, …); only the two `Access` members are
in scope here, since the rest have counterparts in `base.rb`'s own include
chain and need checking one at a time.

## Converged shape

Delete both lines. `Base extends Model` and `interface Model extends … Access`,
so the members and their types arrive by inheritance, and the next change to
`access.rb`'s signature lands in one file.

## Acceptance criteria

- `slice` / `valuesAt` are gone from `base.ts`'s `interface Base`, and
  `pnpm typecheck` is clean without them.
- A call site that reads a slice off an `ActiveRecord::Base` subclass still
  types (`packages/activerecord/src/validations/uniqueness-validation.trails.test.ts`
  and `attribute-methods.trails.test.ts:594` exercise it).
- `pnpm parity:api:extra --package activerecord` does not regress.
- `pnpm vitest run packages/activerecord/src/attribute-methods.trails.test.ts`

## Definition of done

Not done if `base.ts` still restates a signature it inherits from
`ActiveModel::Access`.
