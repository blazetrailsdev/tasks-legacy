---
title: "Fan out the validates macro from model.ts to validations/validates.ts"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - retire-model-set-callback-skip-callback-run-callbacks-passthrough
deps-rfc: []
est-loc: 300
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/validations/validates.rb` defines
`validates` (`:111`, ~30 code lines), `validates!` (`:153`),
`_validates_default_keys` (`:162`) and `_parse_validates_options` (`:166`).

trails defines all four on `Model`:

- `model.ts:577` `validates` — **95 code lines**, 3.2x the Ruby
- `model.ts:689` `validatesBang` (5)
- `model.ts:1683` `_validatesDefaultKeys` (1)
- `model.ts:1692` `_parseValidatesOptions` (1)

102 code lines with `validations/validates.rb` as their Rails home. The
destination `packages/activemodel/src/validations/validates.ts` is where
`parity:api` will look for them.

The 95-line `validates` body is the file's single largest divergence after the
callback block. Rails' body is a `each_validator` loop over
`attributes.extract_options!.symbolize_keys` with `defaults`/`validations`
split, a `key.to_s.camelize` const lookup, and a
`validates_with(validator, defaults.merge(_parse_validates_options(options)))`
tail. The trails body hand-dispatches a fixed table of validator classes
(`PresenceValidator`, `AbsenceValidator`, … imported at `model.ts:104-113`),
which is why the `validators/*` imports sit at the top of `model.ts` at all.

Converge the dispatch onto Rails' shape (a registry lookup, the trails
equivalent of `const_get("#{key.to_s.camelize}Validator")`) as part of the
move; a literal 12-branch table is a bucket-(c) inlined-delegation deviation,
not a language shortcoming.

## Acceptance criteria

- `validates`, `validatesBang`, `_validatesDefaultKeys` and
  `_parseValidatesOptions` are defined in
  `packages/activemodel/src/validations/validates.ts` at those names and reach
  `Model` via `include()` / `Included<>`.
- The body follows `validates.rb:111-151` branch for branch: options
  extraction, `defaults`/`validations` split, the `_validates_default_keys`
  merge, per-key validator resolution, the `validates_with` tail.
- The hard-coded validator-class import block at `model.ts:104-113` is gone
  from `model.ts`.
- `pnpm parity:api:extra --package activemodel` loses `validates` and
  `validatesBang` from `model.ts`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
  `call-mismatches-exclude/activemodel/model.json`'s
  `validates_each` → omits `validates_with` row is expected to converge here or
  in the next story — hand-delete it, then
  `pnpm parity:api:calls:tighten activemodel/model.json`.

## Verification

```bash
pnpm vitest run packages/activemodel/src/validations.test.ts packages/activemodel/src/validations
```
