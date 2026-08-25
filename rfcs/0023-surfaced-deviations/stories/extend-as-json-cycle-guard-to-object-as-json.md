---
title: "as_json's cycle guard misses Object#as_json — instanceValues returns a fresh hash each frame"
status: draft
updated: 2026-08-20
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

The cycle guard PR #6787 added to `as_json` covers the two Rails recursion sites
that recurse into a _caller-owned_ container, but not the third that builds a
fresh one each frame — so an object-graph cycle through instance values still
recurses unbounded and dies on a JS `RangeError`.

`packages/activesupport/src/core-ext/object/json.ts`:

- `Array#as_json` (Ruby `json.rb:163-172`) and `Hash#as_json` (`json.rb:174-197`)
  enter a frame keyed on the value the caller passed, so a repeat visit is
  detectable.
- `Object#as_json` (`json.rb:58-66`) is
  `Hash.asJson(instanceVariables.Object.instanceValues(value), options)` — Ruby's
  `instance_values.as_json`. `instanceValues` returns a **new** object per call,
  so the `Hash` frame keys on a value never seen before and the guard never
  fires. `a.ref = b; b.ref = a` recurses until the stack is gone.

Rails has the same unbounded recursion (a cycle is `SystemStackError` there), so
this is a gap in the trails-side guard rather than a divergence from Rails
behaviour. It is filed because the guard's stated contract — a cycle answers
`nil` rather than felling the encode — is not actually true for the arm most
likely to carry a cyclic graph, which is the model-to-model one.

## Acceptance criteria

- A cycle reached through `Object#as_json` answers `nil` at the repeat visit,
  the same as one reached through `Hash`/`Array`, instead of raising
  `RangeError`.
- The frame keys on the identity of the value being serialized — the receiver
  `Object#as_json` was called with — not on the fresh `instanceValues` result.
- The `@noRailsEquivalent PERMANENT` reason at the guard is updated to describe
  the arm it now covers.
- A test in `packages/activesupport/src/core-ext/object/json.trails.test.ts`
  covers a two-object `a.ref = b; b.ref = a` cycle serialized through
  `Object#as_json` (i.e. class instances with no own `asJson`).
