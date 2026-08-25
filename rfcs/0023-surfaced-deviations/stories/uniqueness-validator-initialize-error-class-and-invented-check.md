---
title: "UniquenessValidator#initialize raises the wrong error class and adds a validation Rails does not have"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# UniquenessValidator#initialize raises the wrong error class and adds a validation Rails does not have

## Context

Surfaced while converging the `validations/uniqueness.ts` call-set rows in
PR #6723 (`wave-4c-ar-core-residue-config`), which fixed that file's
`error_options` denylist but left the constructor alone.

`activerecord/lib/active_record/validations/uniqueness.rb:6-18`

```ruby
def initialize(options)
  if options[:conditions] && !options[:conditions].respond_to?(:call)
    raise ArgumentError, "#{options[:conditions]} was passed as :conditions but is not callable. " \
                         "Pass a callable instead: `conditions: -> { where(approved: true) }`"
  end
  unless Array(options[:scope]).all? { |scope| scope.respond_to?(:to_sym) }
    raise ArgumentError, "#{options[:scope]} is not supported format for :scope option. " \
      "Pass a symbol or an array of symbols instead: `scope: :user_id`"
  end
  super
  @klass = options[:class]
  @klass = @klass.superclass if @klass.singleton_class?
end
```

`packages/activerecord/src/validations/uniqueness.ts` diverges in four ways:

1. **Wrong error class.** The `:conditions` arm throws bare `new Error(...)`
   where Rails raises `ArgumentError` (`:8`). The scope arm, via
   `validateScopeOption`, does throw `ArgumentError` — so the two arms of one
   Ruby method disagree with each other.
2. **A validation Rails does not have.** The constructor rejects a
   non-boolean `:caseSensitive` with an invented message
   (`"... is not a supported value for :caseSensitive option."`). No such check
   exists at `:6-18` or anywhere in `uniqueness.rb`.
3. **An extracted helper Rails inlines.** `validateScopeOption` is a
   module-level function called from three places — the constructor plus both
   `validatesUniqueness` / `validatesUniquenessOf` registrars, which run it at
   declaration time. Rails validates once, in `initialize`.
4. **A dropped line.** `@klass = @klass.superclass if @klass.singleton_class?`
   (`:17`) has no port; trails stops at `this._klass = options.class ?? null`.

The scope message also reads "Pass a string or an array of strings" against
Rails' "Pass a symbol or an array of symbols". That one is a defensible
consequence of Ruby Symbols being JS strings, but it should be justified at the
call site rather than left silently different.

## Converged shape

- Both arms raise `ArgumentError`, with Rails' message strings.
- The `:caseSensitive` type check is deleted; if some trails call site depends
  on it, it moves behind a `@noRailsEquivalent` tag with a stated reason rather
  than sitting unmarked in a ported constructor.
- `validateScopeOption` is inlined into the constructor. If eager
  declaration-time validation is worth keeping, the registrars call the
  constructor's path rather than a parallel helper.
- The `singleton_class?` arm is ported or its absence justified at the call
  site with `:17`.

## Acceptance criteria

- [ ] `new UniquenessValidator({ conditions: "not callable" })` raises
      `ArgumentError`, matching Rails' class and message; a test covers it and
      fails on baseline.
- [ ] No `:caseSensitive` type validation remains untagged in the constructor.
- [ ] `pnpm parity:api:extra --package activerecord` shows no novel surface for
      `validations/uniqueness.ts` beyond what is tagged.
- [ ] `pnpm parity:api:calls` / `:args` clean; uniqueness suites green on all
      three lanes.
