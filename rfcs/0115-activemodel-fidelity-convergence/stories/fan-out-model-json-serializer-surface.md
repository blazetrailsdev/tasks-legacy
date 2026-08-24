---
title: "fan-out-model-json-serializer-surface"
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`fan-out-model-serialization-conversion-access-naming-surface` relocated the
conversion, access, serialization, naming, translation, forbidden-attributes
and `freeze` members out of `packages/activemodel/src/model.ts` into their Rails
home files. Three members of that story's table are still `model.ts` class
bodies, because moving them needs a decision the fan-out could not make
unilaterally:

- `model.ts` `asJson` — `ActiveModel::Serializers::JSON#as_json`
  (`vendor/rails/activemodel/lib/active_model/serializers/json.rb:96-108`)
- `model.ts` `fromJson` — `json.rb:144-149`
- `model.ts` `get includeRootInJson` — the instance reader of
  `class_attribute :include_root_in_json, instance_writer: false, default: false`
  (`json.rb:15`)

`packages/activemodel/src/serializers/json.ts` already ports the whole module as
`class JSON`, with a MORE faithful `asJson` (it honours the `:root` option,
`json.rb:97-101`, which `Model#asJson` drops). The blocker is `from_json`'s
`self.attributes = hash` (`json.rb:147`):

- For a `Model`, `attributes=` is
  `alias attributes= assign_attributes` (`attribute_assignment.rb:36`), which
  trails spells `setAttributes` (`attribute-assignment.ts:60`) because a TS
  `set` accessor cannot be awaited. `Model` has no `attributes` setter at all —
  `attributes` is a getter installed by `include(Model, Attributes)`.
- `serializers/json.ts`'s own `fromJson` writes `this.attributes = hash`, and
  the standalone host fixtures in `serializers/json.test.ts` (`Person`,
  `Keyed`, `Defaulted`, …) define `attributes` as a `get`/`set` accessor pair
  via `Object.defineProperty`, so they depend on that spelling.

So `include(Model, JSON)` — the direct port of `include
ActiveModel::Serializers::JSON` — cannot land until `JSON#fromJson`'s write
routes through the host's `attributes=` alias (`setAttributes`) and the
standalone fixtures are converged to declare it, the way `json.rb`'s own
docstring host declares `def attributes=(hash)`.

## Acceptance criteria

- `serializers/json.ts`'s `fromJson` writes through the Rails alias —
  `setAttributes` (`attribute_assignment.rb:36`) — not a raw `attributes`
  assignment; `serializers/json.test.ts`'s host fixtures declare `setAttributes`
  instead of an `attributes` setter (test NAMES unchanged).
- `model.ts` no longer defines `asJson`, `fromJson` or `includeRootInJson`
  (static or instance); `Model` gains them from `include(Model, JSON)` plus the
  `class_attribute :include_root_in_json` its `[included]` hook issues.
- `Model#asJson` then honours the `:root` option (`json.rb:97-101`) — today it
  does not; add the Rails test coverage for it.
- `pnpm vitest run packages/activemodel/src/serializers packages/activemodel/src/serialization.test.ts packages/activerecord/src/json-serialization.test.ts`
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Definition of done

`model.ts` defines no member whose Rails counterpart lives in
`serializers/json.rb`.
