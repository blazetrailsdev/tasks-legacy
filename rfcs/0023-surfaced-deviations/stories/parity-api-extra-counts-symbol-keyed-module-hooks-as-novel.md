---
title: "parity:api:extra scores symbol-keyed [included]/[extended] hooks as novel surface"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

`include()` / `extend()` fire the module hooks keyed by
`Symbol.for("@blazetrails/activesupport:included")` and `…:extended`
(`packages/activesupport/src/include.ts:121-122`, fired at `:193`, `:272`,
`:371`). That is the ratified TS spelling of Ruby's `included do … end` — e.g.
`ActiveModel::Validations`' block at
`activemodel/lib/active_model/validations.rb:40-50`, ported as
`static [included](base)` in `packages/activemodel/src/validations.ts`, and
`ActiveModel::Attributes`' at `attributes.rb:35`, ported in
`packages/activemodel/src/attributes.ts:224`.

RFC 0115's F0 states these "never surface to `parity:api:extra` and do not
collide with the `SKIP_GROUPS` ban on a string-named `included` member"
(`scripts/parity/conventions.ts:444`, `tsMirrorIsDrift: true`). The second half
holds; the first does not. The TS extractor records the computed key under its
source text and scores it **novel**:

```text
validations.ts — 1 novel, 3 moved
  [included]  name  raiseOnMissingTranslations  toString
```

`validations.ts` went 0 → 1 novel the moment the hook landed (PR #6979), and
`attributes.ts` carries the same phantom. So the ratified idiom is taxed: every
module that ports an `included do` block inflates its novel count by one, which
is noise in the per-file ranking and, for any package inside `GATED_PACKAGES`,
would red `parity:api:extra:gate` for doing the right thing.

## Converged shape

The TS extractor should not emit a member for a **computed** key. It already
excludes whole categories by kind — "Excluded by kind: N novel `interface`
declaration name(s)" — so this is the same move: skip `ComputedPropertyName`
members (or at minimum the two `[included]` / `[extended]` spellings) when
building `ts-api.json`. A Ruby Symbol-keyed method has no string name for
`parity:api` to match on either, so nothing is lost on the Rails side.

After the fix, re-measure and tighten any mark that drops
(`pnpm parity:api:extra:tighten`) — marks only shrink.

## Acceptance criteria

- `pnpm parity:api:extra --package activemodel` no longer lists `[included]` for
  `validations.ts` or `attributes.ts`, and both files' novel counts drop by one.
- No string-named `included` / `extended` member becomes exempt — the
  `SKIP_GROUPS` `tsMirrorIsDrift` entry must keep flagging those.
- `pnpm parity:api:extra:gate` stays green; any mark that drops is tightened,
  never raised.
