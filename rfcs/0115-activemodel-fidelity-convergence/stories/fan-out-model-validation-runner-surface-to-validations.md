---
title: "Fan out the validation-runner surface from model.ts to validations.ts"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-validates-with-to-validations-with
deps-rfc: []
est-loc: 380
pr: 6979
claim: "2026-08-24T12:21:44Z"
assignee: "fan-out-model-validation-runner-surface-to-validations"
blocked-by: null
closed-reason: null
---

## Context

Fourteen `Model` members (93 code lines) have `validations.rb` as their Rails
home, plus two more (35 lines) shared with `validator.rb`:

| trails                                               | Rails                       |
| ---------------------------------------------------- | --------------------------- |
| `model.ts:695` `clearValidatorsBang`                 | `validations.rb:246`        |
| `model.ts:711` `validate` (class, 32L)               | `validations.rb:160`        |
| `model.ts:762` `validatesEach` (18L)                 | `validations.rb:88`         |
| `model.ts:916` `validators` (12L)                    | `validations.rb:204`        |
| `model.ts:947` `validatorsOn`                        | `validations.rb:266`        |
| `model.ts:1920`/`:1924` `_validationContext` get/set | `validations.rb:454`/`:459` |
| `model.ts:1942` `contextForValidation`               | `validations.rb:463`        |
| `model.ts:1952` `runValidationsBang`                 | `validations.rb:473`        |
| `model.ts:1962` `raiseValidationError`               | `validations.rb:478`        |
| `model.ts:1971` `isValid` (27L)                      | `validations.rb:361`        |
| `model.ts:2020` `validate` (instance)                | `validations.rb:361` alias  |
| `model.ts:2030` `isInvalid`                          | `validations.rb:408`        |
| `model.ts:2651` `readAttributeForValidation`         | `validations.rb:433`        |
| `model.ts:2748` `validationContext`                  | `validations.rb:454`        |
| `model.ts:2759` `validateBang`                       | `validations.rb:417`        |

Alongside them sit three trails-invented private helpers with **no Rails
counterpart** (68 code lines) that exist only to support this cluster:

- `model.ts:1453` `_ensureOwnValidators` (7) — copy-on-first-write dup standing
  in for Rails' `inherited(base)` hook at `validations.rb:287-291`. Its JSDoc
  documents a real behavioural divergence: a subclass that never registers a
  validator keeps seeing validators the parent adds later.
- `model.ts:1485` `_registerValidator` (27)
- `model.ts:1520` `_buildValidateConditions` (34)
- `model.ts:2009` `_runValidateCallbacks` (4)

`isValid` at 27 lines against Rails' 6 (`validations.rb:361-366`) is the
per-body hot spot; note `isValid()` returns `Promise<boolean>` in trails
(RFC 0063), which is a settled async-boundary deviation and not this story's
business — keep the async shape, converge the branches.

## The mixin idiom to use (RFC finding F0)

All three mechanisms this story needs are already ported and exported, and
activemodel currently uses none of them — see this RFC's F0. Do not hand-roll a
fourth spelling:

- **`classAttribute()`** — `packages/activesupport/src/class-attribute.ts:70`,
  exported from the package index (`:387`). Its contract is exactly Rails'
  `class_attribute`: _"reads walk the constructor chain; writes are local to the
  class"_. It has **zero** callers in activemodel today.
- **`extend()` / `Extended<>`** — `packages/activesupport/src/include.ts:335`.
  The TS spelling of `extend SomeModule`, i.e. the `ClassMethods` half of a
  Concern. **Zero** callers in activemodel; 65 in activerecord.
- **`include()` / `Included<>`** — `include.ts:184`, plus the symbol-keyed
  `[included]` / `[extended]` hooks fired at `include.ts:193,272,371`, which are
  the TS spelling of an `included do` block. The hooks are keyed by
  `Symbol.for(...)`, so they never surface to `parity:api:extra` and do not
  collide with the `SKIP_GROUPS` ban on a string-named `included` member
  (`scripts/parity/conventions.ts:444`, `tsMirrorIsDrift: true`). CLAUDE.md's
  "Module mixins" section still says these hooks have no TS equivalent; that is
  stale for `included`/`extended` and true only for `inherited`.

## Acceptance criteria

- All sixteen Rails-named members are defined in
  `packages/activemodel/src/validations.ts` and reach `Model` via `include()` /
  `Included<>`; `model.ts` defines none of them.
- Each body matches its `validations.rb` counterpart branch for branch.
- `_registerValidator` and `_buildValidateConditions` are inlined into the
  Rails bodies that call them, or deleted. They must not reappear as new
  invented module surface in `validations.ts`.
- `_ensureOwnValidators` (`model.ts:1453`) is **deleted**, not carried across.
  `validations.rb:50` is
  `class_attribute :_validators, instance_writer: false, default: Hash.new { |h, k| h[k] = [] }`,
  and `classAttribute()` already provides those exact per-subclass semantics —
  so the divergence its own JSDoc admits (a subclass that never registers a
  validator keeps seeing validators the parent adds later) goes away rather
  than being re-documented. It is the fifth of the package's copy-on-first-write
  spellings (F0).
- `validations.rb:41-45`'s `extend ActiveModel::Naming` / `Callbacks` /
  `Translation` and `extend HelperMethods; include HelperMethods` are spelled
  with `extend()` / `include()`, not with `static X = X` assignments.
- `define_callbacks :validate, scope: :name` (`validations.rb:48`) is issued
  from the `[included]` hook, as Rails issues it from `included do`.
- `pnpm parity:api:extra --package activemodel` loses `validators`,
  `validatorsOn`, `clearValidatorsBang`, `validatesEach`, `beforeValidation`,
  `afterValidation` and friends from `model.ts`.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed.

## Verification

```bash
pnpm vitest run packages/activemodel/src/validations.test.ts packages/activemodel/src/validations.trails.test.ts packages/activemodel/src/validator.ts
```
