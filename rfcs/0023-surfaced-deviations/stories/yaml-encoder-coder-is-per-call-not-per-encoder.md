---
title: "YAMLEncoder takes the codec at construction; Rails passes coder per call"
status: draft
updated: 2026-08-14
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

`packages/activemodel/src/attribute-set/yaml-encoder.ts` is now Rails-matched
against `vendor/rails/activemodel/lib/active_model/attribute_set/yaml_encoder.rb`
(PR #6511), but two signature deviations survive the rename.

Rails passes the serializer at the METHOD boundary:

- `def encode(attribute_set, coder)` — `yaml_encoder.rb:12`; the body writes
  `coder["concise_attributes"] = ...` (`:13`).
- `def decode(coder)` — `yaml_encoder.rb:22`; it reads `coder["attributes"]`
  (`:23`) and falls back to `coder["concise_attributes"]` (`:26`).
- `def initialize(default_types)` — `yaml_encoder.rb:8` — stores nothing else.

trails instead takes the serializer at CONSTRUCTION, as a `codec` option
alongside `registry` and `silenceDriftWarnings`, and `encode`/`decode` trade a
serialized string:

- `encode(attributeSet: AttributeSet): string`
- `decode(input: string): AttributeSet`

The constructor options are justified in the class JSDoc (Psych dumps a `Type`
object inline; JSON cannot, so a non-default type travels as a registry key),
and `parity:api` scores the file 4/4 100% today because it does not compare
parameter lists here. But the SHAPE is still Rails-divergent: the `coder` is
per-call in Rails, not per-encoder.

## Converged shape

Move the codec to the method boundary so the call sites read like Rails:

```ts
encode(attributeSet: AttributeSet, coder: AttributeSetCodec): string
decode(coder: AttributeSetCodec, input: string): AttributeSet
```

...or, closer still, give `AttributeSetCodec` the `coder["key"]` accessor shape
Psych's `Psych::Coder` has, so `encode` writes `coder.set("concise_attributes", ...)`
and the codec owns serialization entirely. Then `constructor(defaultTypes)` matches
`yaml_encoder.rb:8` exactly and the `registry` / `silenceDriftWarnings` options move
onto the codec that actually needs them.

Callers to update: `activerecord/src/model-schema.ts#yamlEncoder` (the sole
production caller), `activemodel/src/attribute-set/yaml-encoder.test.ts`.

## Acceptance criteria

- [ ] `YAMLEncoder`'s constructor takes only `defaultTypes` (`yaml_encoder.rb:8`).
- [ ] The codec reaches `encode`/`decode` as a per-call argument, mirroring
      Rails' `coder` (`yaml_encoder.rb:12`, `:22`).
- [ ] `pnpm parity:api:extra --package activemodel` does not grow; call and
      call-arg gates stay green.
