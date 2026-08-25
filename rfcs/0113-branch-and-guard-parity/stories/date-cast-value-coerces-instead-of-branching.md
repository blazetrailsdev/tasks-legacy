---
title: "Date/DateTime cast_value coerces every input to a trimmed string instead of Rails' three-arm class branch"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::Type::Date#cast_value` and `DateTime#cast_value` branch on the
value's CLASS. trails instead coerces every non-Temporal value to a string and
trims it, which silently admits inputs Rails routes down a different arm.

Rails (`activemodel/lib/active_model/type/date.rb:39-48`):

```ruby
def cast_value(value)
  if value.is_a?(::String)
    return if value.empty?
    fast_string_to_date(value) || fallback_string_to_date(value)
  elsif value.respond_to?(:to_date)
    value.to_date
  else
    value
  end
end
```

`DateTime#cast_value` (`activemodel/lib/active_model/type/date_time.rb`) has the
same three-arm shape against `fast_string_to_time` / `fallback_string_to_time`.

trails today (`packages/activemodel/src/type/date.ts:53-55`,
`packages/activemodel/src/type/date-time.ts:44-46`):

```ts
const str = String(value).trim();
if (str === "") return null;
return this.fastStringToDate(str) ?? this.fallbackStringToDate(str);
```

Two divergences fall out of this:

1. **`String(value)` swallows the other two arms.** Rails' `else value` returns a
   non-String, non-date-able value UNCHANGED; trails stringifies it and tries to
   parse. A value Rails hands back untouched becomes `null` here.
2. **`.trim()` has no Rails counterpart.** Rails checks `value.empty?` on the raw
   String — `" "` is NOT empty in Ruby, so Rails proceeds to
   `fast_string_to_date(" ")` and gets `nil` from the regex; trails short-circuits
   on the trimmed empty check. Same answer today, different route, and the trim
   also mutates what reaches `fast_string_to_*` for padded input.

Surfaced by the RFC 0096 activemodel naming burndown (PR #6350) as four `naming`
rows (`type/date.ts` ×2, `type/date-time.ts` ×2) where Rails passes `value` and
trails passes `str`. Deliberately NOT renamed there: the local is a symptom of
the missing branch, and renaming it would have hidden the real gap.

## Converged shape

Port the three-arm `if/elsif/else` in both files, passing the raw `value` to
`fastStringTo*` / `fallbackStringTo*` as Rails does, with the string arm gated on
an actual string check rather than a coercion. Keep the existing Temporal /
`Date` / multiparameter-hash pre-branches that precede it — those are the trails
analogue of Ruby's `respond_to?(:to_date)` arm and are not in question here.

## Acceptance criteria

1. Both `castValue` bodies branch on the value's type in Rails' order and arms;
   a value matching neither the string arm nor the date-able arm is returned
   unchanged.
2. `String(value).trim()` is gone from both; `fastStringTo*` /
   `fallbackStringTo*` receive Rails' `value`.
3. The four `naming` rows for `type/date.ts` / `type/date-time.ts` in
   `pnpm parity:api:calls:args:report` are gone; report before/after.
4. A regression test covers the `else value` arm — it must FAIL on baseline.
5. `pnpm vitest run packages/activemodel` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
