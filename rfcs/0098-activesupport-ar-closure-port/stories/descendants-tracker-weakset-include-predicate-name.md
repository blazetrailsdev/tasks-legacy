---
title: "DescendantsTracker::WeakSet's include? reader is the last in-closure missing member"
status: done
updated: 2026-08-24
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: ["activesupport"]
deps: []
deps-rfc: []
est-loc: 30
priority: 1
pr: 6999
claim: "2026-08-24T18:04:22Z"
assignee: "descendants-tracker-weakset-include-predicate-name"
blocked-by: null
closed-reason: null
---

## Context

This is the last member between RFC 0098 and its "Done means". Measured on
`origin/main` (`API_COMPARE_ALLOW_STALE_BUILD=1 pnpm parity:api --package
activesupport --closure --missing`):

```text
  descendants_tracker.rb    descendants-tracker.ts    6   1   7   86%
      - include? → isInclude

  AR closure: 1077/1078 methods (99.9%)  |  files: 98/102
```

Every other in-closure activesupport file reports 0 missing members.

Rails' `ActiveSupport::DescendantsTracker::WeakSet` defines `[]` and then
`alias_method :include?, :[]`
(`vendor/rails/activesupport/lib/active_support/descendants_tracker.rb:38-41`).
trails' port spells that reader `includes`
(`packages/activesupport/src/descendants-tracker.ts:34`), which is neither the
Ruby name nor the name the compare's `?`-predicate convention expects. The
convention is already settled elsewhere in the tree — `AttributeSet#isInclude`
(`packages/activemodel/src/attribute-set.ts:105`) is the port of the same
`include?` shape, and `packages/activemodel/src/validations/exclusion.ts:38`
calls it.

The rename is contained: `git grep -n "includes(" packages/activesupport/src/descendants-tracker.ts` finds
only the definition, and no other file calls `DescendantsTracker.WeakSet`'s
reader (`Railtie.subclasses.includes(...)` in `railtie.ts:116` is `Array#includes`
on the array `subclasses()` returns, not this method).

## Acceptance criteria

- [ ] `DescendantsTracker.WeakSet`'s `include?` reader is named the way the
      compare's predicate convention and `AttributeSet#isInclude` name it, with
      any call sites updated.
- [ ] `pnpm parity:api --package activesupport --closure` reports
      `descendants_tracker.rb` at 7/7 and the AR-closure line at 1078/1078
      (100%).
- [ ] The `0092/ar-closure-rollup-in-parity-summaries` rollup reads 100%, which
      is RFC 0098's stated "Done means".
