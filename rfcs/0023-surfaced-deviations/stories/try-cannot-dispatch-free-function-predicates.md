---
title: "Object#try cannot dispatch the Ruby-core predicates trails spells as free functions"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`XMLConverter#process_hash` carries a `@missingRailsCall try` tag
(`packages/activesupport/src/core-ext/hash/conversions.ts`), filed in
trails#6818. Rails writes
(`vendor/rails/activesupport/lib/active_support/core_ext/hash/conversions.rb:192`):

```ruby
if entries.nil? || value["__content__"].try(:empty?)
```

trails spells the nil-guard inline (`value["__content__"] != null &&
isEmpty(value["__content__"])`) because `tryCall`
(`packages/activesupport/src/try.ts:21-33`) dispatches a **method name** on the
receiver, and Ruby's `empty?` — a real method on String/Hash/Array — is a free
function in trails (`isEmpty`, `packages/activesupport/src/ruby-empty.ts`).
A JS string has no `.isEmpty` for `tryCall` to find, so the faithful call
cannot be written today.

This is the general shape, not a one-off: every Ruby-core predicate trails
spells as a free function (`isEmpty`, `isBlank`, `isPresent`, …) is unreachable
through `Object#try`, so any Rails body doing `x.try(:empty?)` /
`x.try(:blank?)` must deviate. Grep for `@missingRailsCall try` before starting
— this call site is likely not the only one.

## Converged shape

`tryCall` resolves a Ruby-core predicate name that trails spells as a free
function, so `tryCall(content, "isEmpty")` answers for a JS string exactly as
`content.try(:empty?)` does in Ruby, and the call site becomes the Ruby line
with its `@missingRailsCall` tag deleted. `try.rb`'s own semantics are
unchanged: `nil` in, `nil` out; a receiver that does not answer the name still
returns `nil`.

Rails reference: `vendor/rails/activesupport/lib/active_support/core_ext/object/try.rb`.

## Acceptance criteria

- [ ] `tryCall` dispatches the free-function predicate spellings; `try.ts`'s
      existing `nil`/non-responder arms are unchanged.
- [ ] `conversions.rb:192`'s call site calls it, and its `@missingRailsCall try`
      tag is deleted.
- [ ] Any sibling `@missingRailsCall try` sites found by grep are converged too,
      or left with a note saying why they differ.
- [ ] `pnpm parity:api:calls` / `:args` green; the call-gate row count does not
      grow.
