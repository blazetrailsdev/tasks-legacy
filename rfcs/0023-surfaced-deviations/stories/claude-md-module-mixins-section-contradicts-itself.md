---
title: "CLAUDE.md's Module mixins section contradicts itself after #6746"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
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

PR #6746 corrected CLAUDE.md's "Module mixins" section to say that Ruby's
`included` / `extended` hooks **do** have a TS equivalent (symbol-keyed
callbacks fired by `packages/activesupport/src/include.ts:193,272,371`), and
added positive pointers to `extend()` / `Extended<>` and `classAttribute()`.

It left the section's opening premise in place, and the two now contradict each
other within one screen. `CLAUDE.md:373-374` still reads:

> Rails uses `include`/`extend` to mix module methods into a class. TS has no
> equivalent, so we use **`this`-typed functions assigned directly to the
> class**.

and its worked example is `static aliasAttribute = aliasAttribute` — which is
literally one of the 22 `static X = X` lines at
`packages/activemodel/src/model.ts:312-319,373,1571-1591` that RFC 0115's F0
identifies as a hand-rolled `extend ClassMethods` and tells implementers to
stop writing. A reader following the top of the section produces exactly what
the bottom of the section forbids.

The nuance the section needs (and #6746 also got too absolute about, saying
"reach for it instead of hand-assigning `static x = x`"): assigning a single
`this`-typed function as a static is still correct where Rails defines one
method and no module. `extend()` is for a Ruby `extend SomeModule` — a
module's worth of class methods. Both are right, for different Ruby.

**The fix already exists** as commit `03d45d62d` on branch
`claude-md-mixin-hooks-wording` (pushed, but the #6746 squash-merge raced it
and took only the first commit). It replaces the opener with a Ruby→TS
selection table, scopes the `static X = X` example to "one method, no module",
adds a worked `[included]` example citing the covering tests
(`activesupport/src/include.test.ts:111` "fires the included callback after
methods are copied", `:128` "does not copy the included symbol onto the
prototype"), and splits `inherited` out as the one hook with no equivalent.

## Acceptance criteria

- `CLAUDE.md`'s "Module mixins" section no longer claims TS has no equivalent
  for `include`/`extend`, and no longer presents `static X = X` as the general
  answer.
- The section states which Ruby construct maps to `include()` / `Included<>`,
  `extend()` / `Extended<>`, the `included` / `extended` hooks,
  `classAttribute()`, and a bare `this`-typed static — and says to pick by
  what the Ruby does.
- The `static X = X` guidance is scoped to a single method with no Ruby module
  behind it, so the existing `aliasAttribute` example stays correct.
- `inherited` is called out separately as the one hook with no TS equivalent.
- Docs-only; `npx prettier --check CLAUDE.md` clean.
- Recovering `03d45d62d` is sufficient and is the cheapest path — verify it
  still applies before rewriting by hand.
