---
title: "Model.validate takes one filter where Rails takes *args plus a block"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced during PR #6647 and independently flagged by that PR's reviewer as
out-of-scope-but-real.

Rails `ActiveModel::Validations::ClassMethods#validate`
(`vendor/rails/activemodel/lib/active_model/validations.rb:160-185`) is variadic
and takes a block:

```ruby
def validate(*args, &block)
  options = args.extract_options!

  if args.all?(Symbol)
    options.each_key do |k|
      unless VALID_OPTIONS_FOR_VALIDATE.include?(k)
        raise ArgumentError.new("Unknown key: ...")
      end
    end
  end
  ...
  set_callback(:validate, *args, options, &block)
end
```

trails takes exactly one filter (`packages/activemodel/src/model.ts`):

```ts
static validate<T extends ValidatableRecord = ValidatableRecord>(
  methodOrFn: string | ((record: T) => unknown),
  options: ConditionalOptions = {},
): void
```

Consequences:

- `validate :foo, :bar` (several method names in one call) is impossible; callers
  must issue N calls, which also changes registration order semantics relative to
  a single `set_callback(:validate, :foo, :bar, options)`.
- Rails' `args.all?(Symbol)` guard is vacuously true for a block-only
  `validate { }` call, so `validate(if: opts) { }` skips the key check. PR #6647
  approximated this as `typeof methodOrFn === "string"`, which is equivalent only
  because the arity is fixed at one — it stops being equivalent the moment
  `validate` accepts multiple filters.
- Rails passes the block separately from `*args`; trails conflates "the filter"
  and "the block" into one parameter.

Not exercised by `validations_test.rb` (`test_invalid_options_to_validate` uses
the single-symbol form), which is why #6647 left it — but `validates_test.rb` and
`callbacks_test.rb` may reach it.

Note `Model.validates` / `validatesBang` were made variadic by #6647
(`[...attributes: string[], rules]`); this story is the same treatment for
`validate`, whose tuple also has to admit a function filter and an optional
trailing block.

## Converged shape

```ts
static validate(
  ...args: [...filters: Array<string | ((record: T) => unknown)>, options?: ConditionalOptions]
): void
```

with `extract_options!` semantics on the trailing hash, the
`VALID_OPTIONS_FOR_VALIDATE` check gated on _every_ positional arg being a string
(Rails' `args.all?(Symbol)`), and each filter registered through one
`_registerCallbackOnProto` pass so ordering matches
`set_callback(:validate, *args, options)`.

## Acceptance criteria

- `Model.validate` accepts multiple filters plus a trailing options hash,
  mirroring `validations.rb:160-185`.
- The unknown-key `ArgumentError` fires only when all positional args are method
  names, matching `args.all?(Symbol)` including the block-only case.
- Registration order for `validate("a", "b")` matches a single Rails
  `set_callback(:validate, :a, :b, options)`, not two separate calls.
- `validations_test.rb`, `validates_test.rb` and `validations/callbacks_test.rb`
  do not regress in `pnpm parity:test -- --assertions --package activemodel`.
