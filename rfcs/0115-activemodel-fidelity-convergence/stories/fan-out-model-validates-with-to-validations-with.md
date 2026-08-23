---
title: "Fan out validates_with from model.ts to validations/with.ts"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-validates-macro-to-validations-validates
deps-rfc: []
est-loc: 280
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/validations/with.rb` has exactly two
methods: the class-level `validates_with(*args, &block)` at `:88` (10 code
lines) and the instance-level one at `:144` (7).

trails has both on `Model`, at 114 code lines combined:

- `model.ts:787` `validatesWith` (class, **91 code lines**)
- `model.ts:2049` `validatesWith` (instance, 23)

That is a 6.7x inflation of a 17-line pair, and the file
`packages/activemodel/src/validations/with.ts` is where `parity:api` looks.

Rails' class body is:

```ruby
options = args.extract_options!
options[:class] = self
args.each do |klass|
  validator = klass.new(options.dup, &block)
  validator.check_validity! if validator.respond_to?(:check_validity!)
  validate(validator, options)
end
```

Read the trails body against those five lines before moving it. Three known
sources of the extra 74 lines to classify while porting: block-validator
construction (Rails passes `&block` to `klass.new`), the
`respond_to?(:check_validity!)` guard (trails has `checkValidity` as a `novel`
name on six `validations/*.ts` files per
`pnpm parity:api:extra --package activemodel` — that is its own smell), and
argument normalisation that Rails gets from `extract_options!`.

`model.ts:1485` `_registerValidator` (27 lines) and `model.ts:1520`
`_buildValidateConditions` (34 lines) are trails-invented helpers with no Rails
counterpart that this pair calls; they are in scope for this story only insofar
as they must not survive the move as new invented surface in `with.ts`. If they
cannot be inlined here, the next story (the validation-runner fan-out) owns
them — say which in the PR body.

## Acceptance criteria

- Both `validatesWith` arms are defined in
  `packages/activemodel/src/validations/with.ts` and reach `Model` through
  `include()` / `Included<>`.
- Each body matches its Ruby counterpart branch for branch, including the
  `check_validity!` `respond_to?` guard and the `options[:class] = self`
  assignment.
- `pnpm parity:api:extra --package activemodel` shows `with.ts` at no worse
  than its current novel/moved counts, and `model.ts` loses both rows.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed.

## Verification

```bash
pnpm vitest run packages/activemodel/src/validations/with.test.ts packages/activemodel/src/validations.test.ts
```
