---
title: "Port a minimal Enumerator so ported no-block to_enum arms stop being deleted"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Three ported methods drop Ruby's no-block `to_enum` arm because trails has no
`Enumerator` analogue, each documented at its own call site:

- `Array#extract!` — `activesupport/lib/active_support/core_ext/array/extract.rb:11`
  (`return to_enum(:extract!) { size } unless block_given?`) →
  `packages/activesupport/src/array-utils.ts` `extractBang`, predicate required (PR #6637).
- `Enumerable#index_with` — `activesupport/lib/active_support/core_ext/enumerable.rb:75-87`
  → `packages/activesupport/src/enumerable-utils.ts` `indexWith`, block-or-default required.
- `Deprecators#each` — `activesupport/lib/active_support/deprecation/deprecators.rb:41`
  → `packages/activesupport/src/deprecation/deprecators.ts` `each`, block required.

The divergence is visible in the assertion ratchet: Rails'
`core_ext/array/extract_test.rb:19-32` asserts `assert_instance_of Enumerator`
and drives `extract_enumerator.each(&:odd?)`; the port asserts the no-predicate
call raises and extracts with a second call, matching count/kind but not shape.

## Converged shape

A minimal `Enumerator` port (`size` from the block-form arity hint, `each(block)`
re-invoking the originating method with the block) that the three no-block arms
return, so each method's Ruby branch order and return value are mirrored rather
than deleted. Requires deciding where a core-Ruby class lives in trails and how
`parity:api:extra` accounts for it (core Ruby has no vendor/rails counterpart).

## Acceptance criteria

- The three call sites above restore their no-block arm and drop the "there is no
  Enumerator to return" deviation note.
- `core_ext/array/extract_test.rb` and `core_ext/enumerable_test.rb` assert the
  Enumerator shape as Rails does; `assertion-mismatch-mark.json` lowered, never raised.
- Any new surface is traced to a Ruby counterpart or carries `@noRailsEquivalent`.
