---
title: "Fan out the serialization, conversion, access and naming surface from model.ts"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-attribute-methods-and-registration-surface
deps-rfc: []
est-loc: 280
priority: 18
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

The last block of `model.ts` shadows, 90 code lines across 19 members:

| trails                                                                                                                            | Rails home                           |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| `model.ts:1850` `includeRootInJson`, `:2485` `fromJson` (23L)                                                                     | `serializers/json.rb`                |
| `model.ts:2453` `serializableHash`, `:2660` `readAttributeForSerialization`                                                       | `serialization.rb:38`, `:191`        |
| `model.ts:2457` `asJson` (9L)                                                                                                     | `serializers/json.rb:96`             |
| `model.ts:2598` `toModel`, `:2675` `toParam`, `:2683` `toPartialPath`, `:2734` `toKey`, `:319` `_toPartialPath`                   | `conversion.rb`                      |
| `model.ts:2589` `modelName`, `:1614` `modelName` (static, 11L)                                                                    | `naming.rb`                          |
| `model.ts:1580` `i18nScope`, `:1573` `humanAttributeName`, `:1596` `lookupAncestors`                                              | `translation.rb`, `naming.rb`        |
| `model.ts:2771` `slice` (7L), `:2784` `valuesAt`                                                                                  | `access.rb`                          |
| `model.ts:2632` `sanitizeForMassAssignment`, `:2642` `sanitizeForbiddenAttributes`                                                | `forbidden_attributes_protection.rb` |
| `model.ts:2113` `freeze`, `:2151` `initializeDup`, `:1651` `initInternals`, `:1701` constructor (18L), `:1672` `_mergeAttributes` | mixed                                |

`conversion.ts` is one of the package's 14 already-clean files, and
`access.rb` / `translation.rb` / `forbidden_attributes_protection.rb` all have
TS counterparts that define these names. `ActiveModel::Model` legitimately gets
`slice` and `values_at` from `include ActiveModel::Access` (`model.rb:44`) —
that is the _one_ thing `model.rb` actually does, and after this story
`model.ts` should be doing exactly it.

The `Model` constructor (`:1701`, 18 lines) is `ActiveModel::API#initialize`
(`api.rb:82-85`) — three lines: `assign_attributes(attributes) if attributes`,
`super()`. `call-mismatches-exclude/activemodel/model.json` carries a row
saying this body omits `assign_attributes`; converge it here and delete the row
by hand.

## Acceptance criteria

- `model.ts` retains only what `model.rb` + `api.rb` + `access.rb` define, and
  is **≤ 200 code lines**.
- The constructor calls `assignAttributes` per `api.rb:82-85`; the
  `initialize` → `assign_attributes` row in
  `scripts/api-compare/call-mismatches-exclude/activemodel/model.json` is
  hand-deleted (never reseeded) and
  `pnpm parity:api:calls:tighten activemodel/model.json` run.
- `pnpm parity:api:extra --package activemodel` reports `model.ts` at
  **0 novel / 0 moved**.
- `pnpm lint --fix` after `pnpm parity:api`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Definition of done

Not done if `model.ts` still defines a member whose Rails counterpart is in
another file, however thin the wrapper.

## Verification

```bash
pnpm vitest run packages/activemodel/src/model.test.ts packages/activemodel/src/api.test.ts packages/activemodel/src/access.test.ts packages/activemodel/src/conversion.test.ts packages/activemodel/src/serialization.test.ts
pnpm parity:api:extra --package activemodel
```
