---
title: "converge-from-json-body-onto-assign-attributes"
status: done
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7015
claim: "2026-08-25T01:16:44Z"
assignee: "fan-out-model-json-serializer-surface"
blocked-by: null
closed-reason: null
---

## Context

`fan-out-model-json-serializer-surface` (shipped by #7010) relocated the JSON
surface into `packages/activemodel/src/serializers/json.ts` and mixed it into
`Model`. Two divergences rode along in the relocated `from_json` body and were
not part of that story's table:

1. **Invented shape guards.** `from_json` carried two hand-rolled type checks
   plus their `isPlainJsonObject` / `describeJsonShape` module helpers, raising
   a fabricated `fromJson expected a JSON object, got number` /
   `fromJson root payload must be a JSON object, got …`. Rails has neither.
   `json.rb:148`'s `self.attributes = hash` routes through `assign_attributes`,
   which already raises `ArgumentError` with Rails' own message —
   `"When assigning attributes, you must pass a hash as an argument, Integer
passed."` (`attribute_assignment.rb:29-30`).

2. **`include_root`'s default evaluated on an explicit `nil`.** The body read
   `includeRoot ?? ctor.includeRootInJson`. `def from_json(json, include_root =
include_root_in_json)` (`json.rb:146`) is a Ruby optional parameter, so the
   default applies only when the argument is OMITTED; an explicit `nil` stays
   `nil` and `:147`'s `if include_root` is false. Nullish coalescing substitutes
   the class setting for an explicitly-passed `nil` and unwraps when
   `include_root_in_json` is true — the TS-default-parameter trap CLAUDE.md
   § "Ruby idioms that do not translate literally" (kwargs) names.

## Acceptance criteria

- `from_json` is the `json.rb:146-150` body with no guard arms and no
  `isPlainJsonObject` / `describeJsonShape` helpers; a non-hash payload raises
  `ArgumentError` from `assign_attributes` with Rails' message.
- The second parameter distinguishes "omitted" from "explicitly nil", with a
  regression test that fails on the nullish-coalescing body.
- `Model`'s type surface matches the converged signature.
- The trails-only tests that pinned the fabricated message assert the Rails one.
- `pnpm vitest run packages/activemodel/src/serializers packages/activerecord/src/json-serialization.test.ts`
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
