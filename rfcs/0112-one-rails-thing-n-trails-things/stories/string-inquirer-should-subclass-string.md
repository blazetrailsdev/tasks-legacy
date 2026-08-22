---
title: "StringInquirer wraps a _value where Rails subclasses String"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: wrapper-vs-subclass
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activesupport/lib/active_support/string_inquirer.rb:21` is
`class StringInquirer < String` — a StringInquirer **is** a String, so every
String method works on it directly, `is_a?(String)` is true, and it can be passed
anywhere a String is.

trails wraps a private `_value` instead
(`packages/activesupport/src/string-inquirer.ts`). PR #6649 narrowed the gap by
having the `get` trap resolve String's own methods through the wrapped string
(`Object(target._value)`) and the `has` trap answer for them, which is what makes
`str.length` and `str.upcase`-equivalents work and what
`test_respond_to_fallback_to_string_respond_to` exercises. But the deviation
itself remains, with three observable consequences:

1. `inquirer instanceof String` is false, and `typeof inquirer` is `"object"`,
   so a call site that type-tests a String rejects one.
2. `valueOf()` exists only to serve JS primitive coercion and carries a
   `@noRailsEquivalent PERMANENT` tag citing this very line — the tag is the
   receipt for this deviation, not an independent one.
3. `_value` is invented state with no Ruby counterpart (Ruby's String _is_ the
   state).

`ArrayInquirer` does not have this problem: it already `extends Array<T>`, per
`array_inquirer.rb:14`'s `class ArrayInquirer < Array`. That asymmetry is the
evidence that the String side is a port gap rather than a language wall — JS
subclassing of `Array` works, and the String primitive is the specific
obstruction.

## Converged shape

Converge as far as the language allows and record precisely where the wall is.
Two candidates, in order of preference:

1. `class StringInquirer extends String` — JS `String` **is** subclassable
   (`class S extends String {}` gives a String exotic object whose `length`,
   indexing and prototype methods all work, and `instanceof String` is true).
   Investigate this first: if it holds, the Proxy shrinks to just the
   `?`-predicate / `NoMethodError` arms, `_value` and `valueOf` both disappear,
   and the `@noRailsEquivalent PERMANENT` tag is deleted rather than reworded.
   The known catch is that a `String` subclass instance is not `===`-comparable
   to a primitive, so audit every `Trails.env === "..."`-shaped comparison and
   the `arel` `visitActiveSupportStringInquirer` dispatch
   (`packages/arel/src/visitors/to-sql.ts:1470`) before committing.
2. If (1) genuinely does not hold, keep the wrapper but narrow the tag to the
   one primitive-coercion fact that blocks it, with the failing construct quoted.

Do not close this by rewording the existing tag — per CLAUDE.md a deviation
register is a burndown ledger, and "already documented" is not convergence.

## Acceptance criteria

- Either `StringInquirer` subclasses `String` (and `_value` / `valueOf` / the
  `@noRailsEquivalent PERMANENT` tag are gone), or the story is
  `pnpm tasks block`ed with the specific construct that fails and its output.
- `string_inquirer_test.rb` and `environment_inquirer_test.rb` stay at 0
  assertion-count / 0 kind / 0 value.
- Every existing inquirer call site still passes: `delegated-type.ts:136`,
  `trailties/src/rails.ts` (`Trails.env`), `finisher.ts:111`, and the
  `arel` visitor above.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
