---
title: "respondToMissing drops respond_to_missing?'s self == Base arm and Method#valid?'s reflect_on_aggregation"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/dynamic-matchers.ts` (created by PR #6118, which
relocated `Base.respondToMissingFinder` there as `respondToMissing`) carries a
body that pre-dates the move and drops two arms of
`DynamicMatchers::ClassMethods#respond_to_missing?`
(`vendor/rails/activerecord/lib/active_record/dynamic_matchers.rb:6-13`):

```ruby
def respond_to_missing?(name, _)
  if self == Base
    super
  else
    match = Method.match(self, name)
    match && match.valid? || super
  end
end
```

1. **The `self == Base` arm is missing.** trails runs the finder match on
   `Base` itself too. It happens to answer `false` today only because `Base`
   has no attributes; the branch Rails takes is not there.
2. **`Method#valid?` (`dynamic_matchers.rb:57`) is half-ported.** Rails is
   `model.columns_hash[name] || model.reflect_on_aggregation(name.to_sym)`;
   trails checks `_attributeDefinitions.has(resolved)` only, so a `findBy` over
   a `composed_of` aggregation is not recognised.

Also unported in that file: `Method` / `FindBy` / `FindByBang`
(`dynamic_matchers.rb:26, :93, :105`) and `method_missing` (`:15`).
`dynamic_matchers.rb` is registered in `scripts/parity/unported-files/`
(its reason now notes the partial `respond_to_missing?` port), so none of this
is scored by `parity:api` today.

## Acceptance criteria

- `respondToMissing` takes the `self === Base` arm before matching, mirroring
  dynamic_matchers.rb:7-8.
- The validity check mirrors `Method#valid?` (dynamic_matchers.rb:57) —
  attribute **or** `reflectOnAggregation(name)`.
- Regression tests for both arms in `finder-respond-to.test.ts`, failing on
  baseline. No test renames.
- Decide (and record in the PR) whether the `Method` / `FindBy` / `FindByBang`
  classes are portable at all in TS; if they are not, the
  `scripts/parity/unported-files/` reason is the place that says so, not a new
  register row.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
