---
title: "stub_any_instance covers only Klass.new(), not new Klass()"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`stub_any_instance`
(activesupport/lib/active_support/testing/method_call_assertions.rb:63-65) is
`klass.stub(:new, instance) { yield instance }` — because Ruby's `new` is an
ordinary class method, stubbing it redirects EVERY construction of `klass` for
the block's duration.

trails' port (`packages/activesupport/src/testing/method-call-assertions.ts`, PR #6454)
assigns a `new` property on the class object, which only covers the
literal `Klass.new()` spelling: `new Klass()` is an operator on a binding
importing modules already hold, so it still constructs normally. The limitation
is documented at the call site, but it means the helper does not deliver the
Rails behavior for the call shape trails code actually uses.

## Converged shape

Investigate whether a construction-intercepting shape exists that AR tests can
use — e.g. routing construction under test through a factory the stub can
replace, or a Proxy-wrapped class installed on the module registry — and either
converge to it or, if nothing works, record the language shortcoming with the
experiment that proved it.

## Acceptance criteria

- Either `stubAnyInstance` intercepts the construction shape trails code uses,
  or the story is blocked with the specific evidence.
