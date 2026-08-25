---
title: "Retire attribute-set.ts's JS Map adapter surface"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - converge-dirty-tracker-onto-rails-mutation-trackers
deps-rfc: []
est-loc: 320
pr: 7021
claim: "2026-08-25T00:42:02Z"
assignee: "retire-attribute-set-map-adapter-surface"
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/attribute_set.rb` is 91 code lines
and exposes `[]`, `[]=`, `key?`, `keys`, `fetch_value`, `write_from_database`,
`write_from_user`, `write_cast_value`, `deep_dup`, `reset`, `accessed`, `map`,
`reverse_merge!`, `except`, `each_value`, `cast_types`,
`values_before_type_cast`, `values_for_database`, `to_hash`, `freeze`.

`packages/activemodel/src/attribute-set.ts` is 280 code lines. 143 map onto
those Rails members. The other **124 are a second, JS-shaped spelling of the
same object** plus assorted sidecars:

- Map facade: `get` (`:301`), `set` (`:307`), `has` (`:332`), `entries`
  (`:442`), `forEach` (`:436`), `getAttribute` (`:63`) — duplicates of
  `fetchValue`/`writeCastValue`/`isKey`/`keys`/`eachValue`, which the file also
  has (`:117`, `:172`, `:95`, `:109`, `:21`).
- Freeze sidecar: `isFrozen` (`:286`), `assertNotFrozen` (`:290`) beside the
  Rails `freeze` (`:280`).
- Snapshot/clone sidecar: `snapshotValues` (`:382`), `resolveSnapshotValue`
  (`:401`), `cloneAttribute` (`:414`), `narrowTo` (`:355`).
- Write sidecars: `overrideFromDatabase` (`:246`), `rebindFromDatabaseValue`
  (`:269`), `forgetAttributeAssignment` (`:453`), `forgetAssignmentsBang`
  (`:470`).

`pnpm parity:api:extra --package activemodel` scores the file 6 novel / 6
moved — `getAttribute`, `has`, `overrideFromDatabase`,
`rebindFromDatabaseValue`, `resolveSnapshotValue`, `snapshotValues` novel;
`delete`, `entries`, `forEach`, `get`, `isFrozen`, `set` moved.

The snapshot/restore sidecars exist to serve `DirtyTracker`; land
`converge-dirty-tracker-onto-rails-mutation-trackers` first, which should
remove their only callers.

## Acceptance criteria

- The Map facade is deleted; every caller uses the Rails-named member
  (`fetchValue`, `writeCastValue`, `isKey`, `keys`, `eachValue`).
- The snapshot/clone and freeze sidecars are deleted, or each names the Rails
  method it implements.
- `attribute-set.ts` carries no member without a counterpart in
  `attribute_set.rb`, except any tagged `@noRailsEquivalent` with a reason that
  is a genuine TypeScript language shortcoming.
- `pnpm parity:api:extra --package activemodel` shows `attribute-set.ts` at
  ≤ 1 novel / ≤ 1 moved.
- `activemodel/attribute-set.json`'s 7 baseline rows shrink; converged rows
  hand-deleted then `pnpm parity:api:calls:tighten activemodel/attribute-set.json`.
- Parity deltas non-negative for activemodel **and** activerecord.

## Verification

```bash
pnpm vitest run packages/activemodel/src/attribute-set.test.ts packages/activemodel/src/attribute.test.ts
pnpm vitest run packages/activerecord/src/attribute-methods
```
