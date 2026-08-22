---
title: "NumberConverter#execute stringifies the nil and invalid arms Rails returns unconverted"
status: claimed
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-22T22:33:40Z"
assignee: "converge-remaining-activerecord-copy-on-write-stores-onto-class-attribute"
blocked-by: null
closed-reason: null
---

## Context

`NumberConverter#execute` returns a `string` in every arm, where Rails returns
`nil` or the original `number` object:

```ruby
# activesupport/lib/active_support/number_helper/number_converter.rb:130-137
def execute
  if !number
    nil
  elsif validate_float? && !valid_bigdecimal
    number
  else
    convert
  end
end
```

trails (`packages/activesupport/src/number-helper/number-converter.ts`):

```ts
execute(): string {
  if (this.number === null || this.number === undefined) return String(this.number);
  if (this.validateFloat && !this.validBigdecimal()) return String(this.number);
  return this.convert();
}
```

So a `nil`/`null` number answers the STRING `"null"` where Rails answers `nil`,
and an unparseable number answers `String(number)` where Rails answers the
number itself (`number_helper.rb`'s `number_to_human("x")` is the String `"x"`
only because the input already was one — `number_to_human(Object.new)` returns
the object). Surfaced while converging the `valid_bigdecimal` gate in PR #6885.

## Converged shape

- `execute(): string | unknown` — the first arm returns `null`, the second
  returns `this.number` unchanged, per number_converter.rb:130-137.
- `NumberConverter.convert`'s return type and the `number_helper.rb` wrappers
  (`numberToDelimited` etc., number_helper.rb:29-232) follow, since Rails' public
  helpers return whatever `execute` returned.

## Acceptance criteria

- [ ] `execute`'s three arms return `nil`, `number`, `convert` respectively.
- [ ] Callers in `number-helper.ts` and the ActionView/ActiveModel consumers
      typecheck against the widened return without a `String()` coercion added
      to paper over it.
- [ ] `packages/activesupport/src/number-helper*.test.ts` green.
