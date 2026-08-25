---
title: "validates_uniqueness_of is _merge_attributes + validates_with, with no third call"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing `drop-validates-uniqueness-for-validates-uniqueness-of`
(PR #7060), which deleted the novel `validatesUniqueness` registrar so
uniqueness has the one registrar Rails has
(`vendor/rails/activerecord/lib/active_record/validations/uniqueness.rb:291-293`):

```ruby
def validates_uniqueness_of(*attr_names)
  validates_with UniquenessValidator, _merge_attributes(attr_names)
end
```

Two lines, two calls. trails' `validatesUniquenessOf`
(`packages/activerecord/src/validations/uniqueness.ts`) makes a third:

```ts
const merged = this._mergeAttributes(attrNames);
validateScopeOption(merged.scope); // <- no Rails counterpart
this.validatesWith(UniquenessValidator, merged);
```

`validateScopeOption` is a file-local helper called from BOTH the registrar
(declaration time) and `UniquenessValidator#initialize` (instantiation time).
Rails validates `:scope` in exactly one place — the validator's constructor
(`uniqueness.rb:14-21`, `Array(options[:scope]).all? { |s| s.respond_to?(:to_sym) }`)
— so the eager arm is a trails-only second seat for the same check. PR #7060's
story text flagged it as "likely droppable" and left it in place to keep that PR
scoped to deleting the duplicate registrar.

The reason it looked load-bearing is a timing difference: `validatesWith`
buckets options before reaching the constructor, so the ArgumentError arrives
later than Rails' does. Whether that is observable is the thing to establish —
if it is not, the eager call and the shared helper both go.

## Converged shape

- Drop the `validateScopeOption(merged.scope)` call from `validatesUniquenessOf`
  so the registrar is the two-call body Rails writes.
- Keep the check in `UniquenessValidator`'s constructor, inlined at Rails' site
  and shape if the shared helper then has one caller.
- If a test genuinely depends on the ArgumentError landing at declaration time,
  cite the Rails test that pins that timing before keeping the eager arm.

Rails cite: `validations/uniqueness.rb:291-293` (registrar), `:14-21`
(constructor validation).

## Acceptance criteria

- [ ] `validatesUniquenessOf`'s body is `_merge_attributes` + `validates_with`,
      with no third call.
- [ ] A non-string `:scope` still raises `ArgumentError` with the same message.
- [ ] `pnpm parity:api:calls` clean; AR uniqueness suites green on all three lanes.
