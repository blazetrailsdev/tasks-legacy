---
title: "no-freeform-comments: JSDoc is an unconditional bypass of the autofix"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "arel"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`blazetrails/no-freeform-comments` (PR #6822) keeps every JSDoc block
unconditionally — `eslint/no-freeform-comments.mjs`, keep-rule 1. That is a
real hole in an autofix-backed rule: reformatting a doomed `//` comment as
`/** ... */` bypasses the fix by changing two characters, and no static check
separates a genuine contract note from narration wearing that syntax.

Raised in review of #6822. That PR closed its own exposure — all 12 JSDoc
blocks it adds carry either a `@noRailsEquivalent` / `@internal` tag or a Rails
citation, so none of them rests on bare JSDoc — but the rule still permits the
shape for the next author.

The obvious tightening is "JSDoc must carry a JSDoc tag or a Rails reference to
be kept". **Measured before filing: that flags 94 pre-existing blocks across
arel and activemodel**, and a sample shows they are overwhelmingly ordinary
public-API documentation:

    * Set the FROM table.
    * Add GROUP BY.
    * UNION with another manager.
    * Wrap as EXISTS(subquery).
    * PostgreSQL @> contains operator

Those are JSDoc doing exactly its job. Forcing a tag onto each — or worse, an
invented Rails citation — would be noise and would damage the packages' API
docs, so the blanket form of the rule is not the answer as-is.

Two narrower shapes were probed and both misfire:

- **"JSDoc must attach to a declaration"** — only 7 blocks in both packages
  attach to a statement instead, and all 7 are legitimate: four `describe(...)`
  file headers in tests, three `include(...)` mixin-wiring notes.
- **"JSDoc must carry a tag"** alone — same 94-block problem.

## Acceptance criteria

Pick one and justify it against the measurement above:

- [ ] A discriminator is found that catches narration-as-JSDoc without flagging
      ordinary API docs, is implemented in `eslint/no-freeform-comments.mjs`,
      and is covered by tests; **or**
- [ ] The blanket tag-or-citation rule is adopted deliberately, and the 94
      pre-existing blocks are swept in the same change (they are listed by the
      probe described above); **or**
- [ ] The story is closed as "not statically closable", with the limitation
      left documented in the rule's own doc where it already lives, and the
      convention enforced in review instead.
