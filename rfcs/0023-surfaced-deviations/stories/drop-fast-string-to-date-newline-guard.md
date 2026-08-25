---
title: "Drop fastStringToDate's newline guard, which Rails has no counterpart for"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activemodel/lib/active_model/type/date.rb:51-55` is:

```ruby
ISO_DATE = /\A(\d{4})-(\d\d)-(\d\d)\z/
def fast_string_to_date(string)
  if string =~ ISO_DATE
    new_date $1.to_i, $2.to_i, $3.to_i
  end
end
```

`packages/activemodel/src/type/date.ts#fastStringToDate` opens with a guard
Rails does not have:

```ts
if (s.includes("\n")) return null;
```

It was presumably added on the assumption that JS `$` behaves like Ruby's `$`
(matching before a trailing newline). It does not: without the `m` flag, JS `$`
matches only at the very end of input, exactly like Ruby's `\z`. Measured on
both runtimes:

```console
node -e 'console.log(/^(\d{4})-(\d\d)-(\d\d)$/.exec("2024-01-01\n"))'  # null
ruby -e 'p("2024-01-01\n" =~ /\A(\d{4})-(\d\d)-(\d\d)\z/)'             # nil
```

So the guard is unreachable-as-a-difference: it can only ever return `null` for
inputs the regex was already going to reject. It is extra surface inside a
ported body, and it invites the reader to believe the two anchorings differ.

Surfaced while converging `cast_value` in PR #6528; explicitly left out of
scope there.

## Converged shape

Delete the line. `fastStringToDate` becomes the three-line `if regex → newDate`
Rails has, with `ISO_DATE` (date.ts:199, already `/^(\d{4})-(\d\d)-(\d\d)$/`)
doing all the anchoring.

## Acceptance criteria

- [ ] `fastStringToDate` has no `includes("\n")` guard and is a line-for-line
      mirror of `date.rb:51-55`.
- [ ] A test pins `cast("2024-01-01\n")` → `null` so the behaviour the guard was
      protecting is covered by the regex alone.
