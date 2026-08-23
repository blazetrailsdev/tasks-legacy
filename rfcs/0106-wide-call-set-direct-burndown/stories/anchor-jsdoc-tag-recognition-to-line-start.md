---
title: "Anchor JSDoc tag recognition to line start so prose mentions stop minting phantom tags"
status: draft
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
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

`parity:api:build` migrates a baseline `reason` **verbatim** into a
`@missingRailsCall` JSDoc block. If that reason mentions another tag family by
name — `@noRailsEquivalent`, `@missingRailsArgs`, `@internal` — the TS manifest
extractor reads the bare token as a **real tag on the enclosing declaration**
and mints a phantom one, whose "reason" is whatever prose followed it.

Hit for real in PR #6898 (wave 5f). The
`alias_attribute_method_definition -> has_attribute?` reason ends with

    ... trails generates at the end of every schema load
    (`defineAttributeMethodsAfterLoad`, model-schema.ts:1159, tagged
    @noRailsEquivalent against CLAUDE.md's "Generated attribute readers are
    properties"); ...

Migrating it minted a phantom `@noRailsEquivalent` on
`aliasAttributeMethodDefinition` (`packages/activerecord/src/attribute-methods.ts`),
which reddened the `Rails API/Test Comparison` CI job **twice** — once as a
STALE tag (the method is not extra surface) and once as a tag stating no
permanence claim. Nothing in the local `parity:api:calls` /
`parity:api:calls:args` gates catches it; only `parity:api:extra` does, and only
for the tag family the phantom happened to land in.

The escape today is accidental spelling: the SAME file's pre-existing prose
paragraph writes the token backticked (`` `@noRailsEquivalent` ``) and is fine,
because the token is followed by a backtick rather than whitespace. A reason
author has no way to know that is what saves them.

Note the asymmetry that makes this a real bug rather than a style nit:
`parseJsdoc` in `scripts/api-compare/missing-rails-call-tags.ts:79,180` already
guards its own reason-continuation with a **line-anchored** `ANY_TAG_LINE`
(`/^\s*\*?\s?@\S/`), and `suppressedCallsIn`'s doc comment states the intent
outright — "A line-leading prose `@tag` inside a reason ends that reason at
`parseJsdoc`'s boundary rule, so it can never mint a suppression for a call
nobody tagged." The manifest extractor that feeds `parity:api:extra` does not
apply the same rule, so the two halves of one tag family disagree about what
counts as a tag.

## Converged shape

Make tag recognition line-anchored everywhere, matching `ANY_TAG_LINE`: a tag
is only a tag when it opens a JSDoc line. A mid-line `@noRailsEquivalent` in
prose then reads as prose in BOTH halves, and the backtick spelling stops being
load-bearing.

Where to look:

- `scripts/api-compare/missing-rails-call-tags.ts:79` — `ANY_TAG_LINE`, the
  existing correct rule; the shared constant the other readers should use.
- `scripts/api-compare/extra-surface.ts` — consumes `m.noRailsEquivalent` off
  the TS manifest (`:611`, `:625`, `:642`); `:485` already documents a related
  truncation hazard (`proseTagAfter`).
- The TS manifest builder that populates `noRailsEquivalent` /
  `missingRailsArgs` / `missingRailsCalls` on each member — the actual
  extraction site, and where the anchor is missing.

A guard test belongs with it: a JSDoc block whose reason prose contains a
mid-line `@noRailsEquivalent` must yield **no** `noRailsEquivalent` tag on the
declaration. That test fails on today's extractor.

Not a Rails-fidelity issue — this is parity tooling with no Rails counterpart —
but it is a live hazard for every remaining RFC 0106 wave, since each one copies
reviewed baseline prose into JSDoc, and the reasons most likely to name another
tag family are exactly the ones arguing a language shortcoming.

## Acceptance criteria

- [ ] A mid-line `@noRailsEquivalent` (or `@missingRailsArgs`, `@internal`) in
      reason prose mints no tag on the enclosing declaration.
- [ ] The line-anchored rule is one shared constant, not re-derived per reader.
- [ ] A guard test covers the mid-line case for each tag family, and fails on
      the pre-fix extractor.
- [ ] `pnpm parity:api:extra`, `pnpm parity:api:calls`, `pnpm parity:api:calls:args` green.
