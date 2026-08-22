---
title: "NumberConverter#execute gates on the invented isValidFloat, not Rails' valid_bigdecimal"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`NumberConverter#execute` gates on a trails-invented `isValidFloat`, where Rails
gates on `valid_bigdecimal`:

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
  if (this.validateFloat && !this.isValidFloat()) return String(this.number);
  return this.convert();
}
```

`isValidFloat` / `numberAsFloat` have no Rails counterpart at all — Rails has
neither method; the `Float()` gate they implement was removed upstream in favour
of `valid_bigdecimal`. The two are not equivalent: `valid_bigdecimal` answers a
BigDecimal (or nil), and its String arm is `BigDecimal(number, exception: false)`
— a whole-string parse — where `isValidFloat` is `!isNaN(Number(x))`, which
accepts values `BigDecimal()` rejects.

Surfaced in PR #6881, which had to teach `isValidFloat` about `Rational` for the
`convert_to_decimal` Rational arm to be reachable at all. Rails needs no such
patch because `valid_bigdecimal`'s FIRST arm is already
`when Float, Rational then number.to_d(0)` (:179-181) — the arm trails'
`validBigdecimal` also omits, falling a Rational through to the `to_d rescue nil`
else and answering `null`.

## Converged shape

- `execute` gates on `validBigdecimal()`, per number_converter.rb:133.
- `validBigdecimal` gains Rails' `when Float, Rational` arm, `number.to_d(0)`
  (:179-181). `to_d(0)` on a Rational is the BigDecimal default precision
  (`Rational(1,3).to_d(0)` is `0.33333333333333333333333333333333e0` on MRI
  3.3.11), NOT trails' `parseRational(value, 0)`, which throws
  "can't omit precision for a Rational" — the default has to be supplied.
- `isValidFloat` and `numberAsFloat` are then callerless invented surface and go
  away (check `numberAsFloat`'s other callers — the percentage / currency
  converters — first; each should read the value the Rails way).

## Acceptance criteria

- [ ] `execute` calls `validBigdecimal`, not `isValidFloat`.
- [ ] `validBigdecimal` has the `Float, Rational` arm at Rails' position.
- [ ] `isValidFloat` / `numberAsFloat` deleted, or each remaining caller
      converged onto a Rails counterpart.
- [ ] `packages/activesupport/src/number-helper*.test.ts` green, including the
      Rational assertions added by #6881.
