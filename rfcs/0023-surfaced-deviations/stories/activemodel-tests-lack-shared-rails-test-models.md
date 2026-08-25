---
title: "activemodel tests redeclare Rails test models inline instead of a shared test-helpers/models dir"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' activemodel test suite shares a small set of test models — `Topic`,
`Person` (+ `Person::Gender`, `Child`), `CustomReader`, `Reply`, `Automobile`,
`Sheep` — under `vendor/rails/activemodel/test/models/`, and every
`test/cases/**` file `require`s them and clears their validators in `teardown`.

trails has no `packages/activemodel/src/test-helpers/models/` at all (activerecord
does). Every activemodel test file therefore defines its own throwaway model
class inline, and before PR #6591 most of them defined an invented `Person` with
a `name` attribute that appears in no Rails model. That is the root cause of the
activemodel assertion-value debt tracked by RFC 0105: a test asserting against
an invented model cannot assert Rails' literal expected values
(`["must be blank"]`, `["hoo 5"]`, `"This string contains 'single' and \"double\" quotes"`).

PR #6591 converged three files by hand-copying the needed subset of
`models/topic.rb` (`title`, `content`, `condition_is_true`, `condition_is_false`),
`models/person.rb` (`karma`) and `models/custom_reader.rb`
(`read_attribute_for_validation` over a backing hash) into each test file — three
near-identical copies, which is exactly the duplication the shared directory
exists to avoid.

Rails source:

- `vendor/rails/activemodel/test/models/topic.rb:1-64`
- `vendor/rails/activemodel/test/models/person.rb:1-19`
- `vendor/rails/activemodel/test/models/custom_reader.rb:1-17`
- consumers, e.g. `vendor/rails/activemodel/test/cases/validations/presence_validation_test.rb:5-13`

## Converged shape

Add `packages/activemodel/src/test-helpers/models/` mirroring
`vendor/rails/activemodel/test/models/` file-for-file (`topic.ts`, `person.ts`,
`custom-reader.ts` first), each a `Model` subclass carrying the full Rails
`attr_accessor` set and helper methods, and have the validation test files
import them instead of redeclaring a local class. `Ruby p[:karma] = x` stays as
the `data` backing hash + `readAttributeForValidation` override (TS has no
`[]=` operator overload); everything else keeps the Rails names.

## Acceptance criteria

- `packages/activemodel/src/test-helpers/models/{topic,person,custom-reader}.ts`
  exist and mirror their Rails counterparts method-for-method.
- `validations/{conditional,presence,absence}-validation.test.ts` import them
  and declare no local model stand-in.
- No test name changes; `pnpm parity:test` percent for activemodel does not drop,
  and `scripts/test-compare/assertion-mismatch-mark.json` is not raised.
