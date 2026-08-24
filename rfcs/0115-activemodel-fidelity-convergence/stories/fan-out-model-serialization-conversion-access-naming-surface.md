---
title: "Fan out the serialization, conversion, access and naming surface from model.ts"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-attribute-methods-and-registration-surface
deps-rfc: []
est-loc: 280
pr: 7010
claim: "2026-08-24T22:42:07Z"
assignee: "fan-out-model-serialization-conversion-access-naming-surface"
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

- `model.ts` defines **no member body** whose Rails counterpart lives in another
  `.rb` — every one of the 19 in the table above is reached through the
  `include` / `extend` its Rails counterpart writes.
- The constructor calls `assignAttributes` per `api.rb:82-85`. (There is no
  `scripts/api-compare/call-mismatches-exclude/activemodel/model.json` shard and
  no `initialize` → `assign_attributes` row in the tree, so nothing is deleted
  or tightened; `pnpm parity:api:calls` is clean either way.)
- `pnpm parity:api:extra --package activemodel` reports `model.ts` at
  **0 novel**.
- `pnpm lint --fix` after `pnpm parity:api`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

### Narrowed 2026-08-24 (PR #7010, review round 2)

Two criteria as originally written measure something a fan-out cannot move, and
are re-scoped onto `split-model-mixin-surface-to-active-model-model`:

- **`0 moved`.** `extra-surface` scores a NAME against the allow-set
  `model.rb` + its Ruby include chain builds. `model.ts`'s 61 `moved` names are
  the Attributes / AttributeMethods / Dirty / Callbacks / Serializers::JSON
  surface trails' `Model` mixes in and `ActiveModel::Model` does not, and they
  score against `model.ts` whether they are spelled as a `declare static`, an
  `interface Model extends Dirty`, or nothing at all beyond an
  `include(Model, Dirty)` call — `include()` of a class module pushes the module
  onto the host's `extends` (`extract-ts-api.ts:1191-1227`) and
  interface-`extends` members carry no `declaredIn` so they are counted
  (`extract-ts-api.ts:14-27`). Relocating bodies cannot change the number.
- **`≤ 200 code lines`.** Same fact. After the fan-out `model.ts` is 378 code
  lines, of which 111 are `declare` statements and 45 are `interface Model`
  members — 156 type-only lines that emit nothing — plus 27 imports and the 29
  `include()`/`extend()`/`prepend()` calls that ARE the Rails `include` lines.
  What is left is the constructor, `dup`, `isPersisted` and
  `isAttributeMethod`. The type-only bulk exists because TS cannot type a
  runtime `include(Model, X)` without a declaration; deleting it does not shrink
  the mixin set, it makes `Model.validates` untyped for every caller.

Both numbers fall out of `Model` being the AR-shaped god class rather than
`model.rb`'s two-line concern, which is that story's job, not this one's.

## Definition of done

Not done if `model.ts` still defines a member BODY whose Rails counterpart is in
another file, however thin the wrapper.

## Verification

```bash
pnpm vitest run packages/activemodel/src/model.test.ts packages/activemodel/src/api.test.ts packages/activemodel/src/access.test.ts packages/activemodel/src/conversion.test.ts packages/activemodel/src/serialization.test.ts
pnpm parity:api:extra --package activemodel
```
