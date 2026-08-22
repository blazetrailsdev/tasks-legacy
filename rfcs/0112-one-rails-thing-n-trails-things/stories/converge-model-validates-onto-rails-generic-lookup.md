---
title: "Converge Model.validates onto Rails' generic validator lookup"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while measuring `assertions-activemodel-validates-test` (PR #6625).

`Model.validates` (`packages/activemodel/src/model.ts:561-655`) is a hardcoded
chain — `if (rules.presence) … if (rules.length) … if (rules.comparison) …` —
one arm per built-in validator, with the shared `allowNil`/`allowBlank` merge
open-coded inside each arm. Rails' `validates`
(`vendor/rails/activemodel/lib/active_model/validations/validates.rb:105-124`)
is generic and much shorter:

```ruby
def validates(*attributes)
  defaults = attributes.extract_options!.dup
  validations = defaults.slice!(*_validates_default_keys)
  ...
  validations.each do |key, options|
    key = "#{key.to_s.camelize}Validator"
    begin
      validator = const_get(key)
    rescue NameError
      raise ArgumentError, "Unknown validator: '#{key}'"
    end
    next unless options
    validates_with(validator, defaults.merge(_parse_validates_options(options)))
  end
end
```

Three behaviours follow from that shape and are absent in trails:

- **Lookup by option key.** `validates :karma, email: true` resolves
  `EmailValidator`; `'namespace/email': true` resolves
  `Namespace::EmailValidator`. trails has no registry, so a model-defined or
  app-defined validator cannot be named by key at all — only `validatesWith`
  works.
- **`ArgumentError` on an unknown key** — `"Unknown validator: 'UnknownValidator'"`,
  raised for a falsy option value too (`unknown: false` raises before the
  `next unless options` guard). trails silently ignores the key.
- **`_validates_default_keys`** (validates.rb:126-128) and its model-level
  override, plus `_parse_validates_options` (validates.rb:158-169) — the
  shorthands `format: /re/`, `inclusion: %w(a b)`, `length: 6..20`,
  `numericality: true`.

## Converged shape

Port `validates` as Rails writes it, with `_validates_default_keys` and
`_parse_validates_options` extracted as Rails extracts them. The one piece with
no literal TS twin is `const_get("#{key.to_s.camelize}Validator")` — Ruby
constant lookup off the model's namespace. Settle that spelling first (a
registry keyed by the camelized name, populated by the built-in validators and
extensible by a model, is the likely answer) and justify it at the call site.

## Acceptance criteria

- `validates` mirrors validates.rb:105-124 — same locals, same branch order,
  same `ArgumentError` class and message string.
- `_validates_default_keys` and `_parse_validates_options` exist at the Rails
  names and are overridable by a model, as `test/models/topic.rb:11-13` does.
- A validator named by option key resolves, including a namespaced one.
- `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green; no new baseline
  rows.
- Unblocks `assertions-activemodel-validates-test`.

## Absorbed: `validates-does-not-route-through-parse-validates-options`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "validates-does-not-route-through-parse-validates-options"

### Context

`ActiveModel::Validations::ClassMethods#validates`
(`vendor/rails/activemodel/lib/active_model/validations/validates.rb:107-115`)
normalizes every rule value through one helper:

```ruby
defaults = ...
validations.each do |key, options|
  ...
  validator.new(defaults.merge(_parse_validates_options(options)), &block)
```

so `length: 6..20`, `inclusion: %w(m f)` and `presence: true` all reach their
validator as the option hash `_parse_validates_options` produces
(`validates.rb:166-177`: `TrueClass -> {}`, `Hash -> itself`,
`Range, Array -> { in: options }`, else `{ with: options }`).

trails' `Model.validates` (`packages/activemodel/src/model.ts:540-650`) never
calls `_parseValidatesOptions`. It is a hand-rolled dispatcher with one `if`
per known validator key, each re-implementing only the `true -> {}` arm
inline (`model.ts:573-645`, e.g. `rules.presence === true ? {} : rules.presence`)
and spreading the value otherwise. The ported helper exists and is correct
(`packages/activemodel/src/validations.ts:_parseValidatesOptions`, exposed as
`Model._parseValidatesOptions` at `model.ts:1611`) — it is simply not on the
path.

Consequences:

- The `Range, Array -> { in: ... }` arm is unreachable through `validates`.
  Rails' `test_validates_with_range`
  (`vendor/rails/activemodel/test/cases/validations/validates_test.rb:114-121`,
  `Person.validates :karma, length: 6..20`) cannot be ported as written; the
  trails test of that name
  (`packages/activemodel/src/validations/validates.test.ts` "validates with
  range") currently asserts an unrelated `numericality` shape instead.
  Likewise `validates_test.rb:106` (`inclusion: %w(m f)`) is ported as
  `{ inclusion: { in: [...] } }` rather than the bare array Rails passes.
- The `else -> { with: options }` arm is unreachable, so
  `validates :name, format: /\A[a-z]+\z/` (a bare Regexp) does not work.
- Every new validator key needs another `if` block in `validates` rather than
  registering by name, which is why the dispatcher only knows a fixed list.

Surfaced in PR #6219 review while converging `_parseValidatesOptions`'s Range
arm (`Range` became a real class in that PR, so the `instanceof Range` test the
helper needed is now available — the helper is fixed and unit-covered in
`validates.trails.test.ts`, but nothing reaches it through `validates`).

### Acceptance criteria

- [ ] `Model.validates` resolves each rule value through
      `_parseValidatesOptions`, as `validates.rb:113` does, instead of the
      per-key `=== true ? {} : value` inline normalization.
- [ ] `validates :karma, length: <Range>` and `validates :gender, inclusion:
<Array>` reach their validators as `{ in: ... }`.
- [ ] The ported `validates with range` test is converged to
      `validates_test.rb:114-121` (`length: 6..20`, asserting the
      "is too short (minimum is 6 characters)" error), and `validates with
array` to `validates_test.rb:104-112`. Test names unchanged.
- [ ] `pnpm parity:api:calls` clean — the `validates` call-set gains
      `_parse_validates_options`.

## Absorbed: `validates-raises-argumenterror-when-no-validation-supplied`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "validates raises ArgumentError when no validation is supplied"

### Context

Surfaced while fixing PR #5192 (`save-through-record-uses-bang-save`).

A test registered a join-row validator as `(this as any).validates((r) => {
r.errors.add("base", "Join always invalid") })` — the block form, whose real
API is `validate(fn)`. Our `validates` silently registered **nothing**: the
test passed vacuously for as long as it existed, and only surfaced when the
bang-save change made the expected raise mandatory.

`validates` (`packages/activerecord/src/validations.ts:263`, and
ActiveModel's `Model.validates` at `packages/activemodel/src/model.ts:549`)
takes `(attribute, rules)`. Called with one argument, `rules` is `undefined`;
`{ ...undefined }` spreads to `{}`, every `if (arRules.x)` arm is skipped, and
the call returns having registered no validator at all.

Rails raises instead
(`vendor/rails/activemodel/lib/active_model/validations/validates.rb:116`):

```ruby
raise ArgumentError, "You need to supply at least one validation" if validations.empty?
```

No other caller currently passes a function (checked repo-wide), so this is a
latent footgun rather than an active bug — but it silently converts a
mis-typed validation into a no-op, which is exactly how the #5192 test rotted.

### Acceptance criteria

- [ ] `validates` raises `ArgumentError("You need to supply at least one
validation")` when no recognized validation rules remain, matching
      `validates.rb:116`. Cover both the AR override
      (`activerecord/src/validations.ts`) and ActiveModel's
      (`activemodel/src/model.ts`).
- [ ] Shared-only options (`on`/`if`/`unless`/`allowNil`/`allowBlank`) do not
      count as validations, per Rails' `_validates_default_keys` exclusion.
- [ ] Test covering the no-rules call and the mistaken block form
      `validates(fn)`.
- [ ] No existing caller regresses (repo-wide check showed none passing a
      function today).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
