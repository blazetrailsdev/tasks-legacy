---
title: "Converge attribute-set/builder.ts's residue onto attribute_set/builder.rb"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - retire-attribute-set-map-adapter-surface
deps-rfc: []
est-loc: 180
pr: 7028
claim: "2026-08-25T12:34:50Z"
assignee: "converge-attribute-set-builder-residue"
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/attribute_set/builder.rb` is 152
code lines; `packages/activemodel/src/attribute-set/builder.ts` is 269. The 61
code lines with no Rails counterpart:

`dupAttribute` (`:30`), `blockOrValue` (`:157`), `get` (`:246`), `set`
(`:251`), `has` (`:342`), `cloneAttr` (`:350`), `assignDefault` (`:362`).

`get`/`set`/`has` are the same Map-facade duplication as the parent story —
`LazyAttributeSet` / `LazyAttributeHash` in Rails answer `[]`, `[]=`, `key?`,
`fetch`, `each_key`, `transform_values`, `marshal_dump`. `dupAttribute` and
`cloneAttr` are two spellings of the same `Attribute#dup` / `deep_dup`
(`attribute.rb`), and `assignDefault` duplicates the default-materialisation
`LazyAttributeHash#assign_default_value` (`builder.rb:132`).

`pnpm parity:api:extra --package activemodel` scores it 1 novel / 2 moved, and
`call-mismatches-exclude/activemodel/attribute-set/builder.json` holds 6
baseline rows — the third-largest shard in the package.

Sequence after `retire-attribute-set-map-adapter-surface`; they share the
`AttributeSet` contract but not the files, so the PRs do not overlap.

## Acceptance criteria

- The Map facade (`get`, `set`, `has`) is gone; callers use `[]`/`[]=`/`key?`'s
  Rails-named TS spellings.
- `dupAttribute` and `cloneAttr` converge to one, at the Rails name from
  `attribute.rb`.
- `assignDefault` converges onto `assign_default_value` (`builder.rb:132`), or
  is deleted if that method already exists in the file.
- `builder.ts` carries no member without a `builder.rb` counterpart.
- `activemodel/attribute-set/builder.json`'s 6 rows shrink; converged rows
  hand-deleted then
  `pnpm parity:api:calls:tighten activemodel/attribute-set/builder.json`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Verification

```bash
pnpm vitest run packages/activemodel/src/attribute-set packages/activemodel/src/attribute-set.test.ts
```
