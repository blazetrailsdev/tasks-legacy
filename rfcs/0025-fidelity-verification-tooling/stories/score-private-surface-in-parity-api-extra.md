---
title: "Score private/protected TS surface in parity:api:extra"
status: draft
updated: 2026-08-22
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: ["activerecord", "activesupport"]
deps: []
deps-rfc: []
est-loc: 420
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`parity:api:extra` is asymmetric about visibility, and the asymmetry leaves a
whole class of invented surface unmeasured.

**Ruby side (the allowed set) already includes private and protected methods**,
deliberately. `scripts/api-compare/extra-surface.ts:1199-1207`:

> Private/protected Ruby methods (internal) still count: a TS method mirroring a
> Rails-private method isn't _extra_ surface, it's a visibility divergence — the
> method exists in Rails. Excluding them here would mislabel every
> public-port-of-a-private-method as drift.

Same reasoning at `:1197-1199` for the novel-vs-moved split. That is correct and
should not change.

**TS side (the scored population) excludes them.** `walkTsFileSurface`
(`extra-surface.ts:935-946`) drops every member with `internal: true` — TS
`private`/`protected`, `#`-prefixed fields, `@internal` JSDoc on a top-level
exported function — and separately drops every `_`-prefixed name
(`extra-surface.ts:885-896` documents why: the extractor keeps `_`-exports as
public, and the repo's Rails-private convention means they should not count).

Net effect: **a trails-private helper with no Rails counterpart is invisible to
the report.** A `private computeFoo()` on a ported class, or an
`_normalizeArgs()` top-level function, costs nothing in the extra-surface
number however invented it is.

CLAUDE.md's "No extra abstraction" rule carves out no exception for privates —
"Do not add a helper, wrapper, indirection layer, or 'cleaner' rewrite that
Rails does not have" applies to a private helper exactly as it does to a public
one. The rule and its measurement currently disagree.

### Two constraints that shape the design

**This is a different question, not a bigger version of the same one.** Public
extra surface is an API-compatibility claim — a trails user can call it, so it
is part of the port's contract. A private helper is an internal-structure claim
— Rails decomposed a method one way and we decomposed it another. Both are
fidelity debt, but folding privates into the existing `Novel` / `Moved` /
`Total` columns would break the public series and make the public trend
unreadable. The private population needs its own column(s) and its own
baseline.

**The `_`-prefix filter is doing real work and cannot simply be dropped.** That
convention exists _because_ the extractor treats `_`-exports as public; it is
the repo's marker for "Rails-private, spelled in TS". Score privates naively and
every one of those names becomes extra. The honest version asks "is there a
Rails private method this mirrors?" — which needs the Ruby-side EFFECTIVE
visibility that `scripts/build-rails-privates-manifest.ts` already resolves
through the `include`/`extend` graph.

### Sequencing

That last point is why this depends on the two sibling 0025 stories: TS-side
`visibility` must be accurate before a private population can be scored at all.
`extract-ts-api.ts:791` derives visibility from TS syntax, so mixin-section
members currently extract as public regardless of their Rails visibility
(`extract-ts-api-stamp-mixin-section-visibility` fixes that), and
`add-visibility-parity-gate` builds the Ruby-vs-TS visibility comparison this
story needs to decide whether a given private helper mirrors a Rails private.

## Acceptance criteria

- `parity:api:extra` scores a SEPARATE private/protected population alongside
  the public one, reported in its own column(s) — the existing `Novel` /
  `Moved` / `Total` / `Allowed` / `NoCntrp` numbers for the public population
  are unchanged, so the historical series stays comparable.
- A TS private/protected member (or `_`-prefixed name) is scored as extra only
  when no Rails private or protected method on the mapped entity mirrors it,
  resolved through the effective-visibility model in
  `build-rails-privates-manifest.ts` — not by name-matching against public
  Rails methods only.
- `--json` output carries the private population so it can be triaged the same
  way the public one is.
- A seeded only-shrink baseline for the private population, on the same
  contract as the call ratchets (NEW fails, STALE fails, partial-scope guard),
  with a per-row reviewed `reason`; rows written through `serializeBaseline` so
  hand-added rows stay sorted.
- The PR body states the private-population totals per package on first
  measurement, so the size of the newly-visible debt is on the record.
- Documented in CONTRIBUTING.md alongside the existing `parity:api:extra`
  guidance, including the explicit statement that a private helper Rails does
  not have is extra abstraction under CLAUDE.md and not exempt.

## Notes

Do NOT change the Ruby-side allowed-set behaviour at `extra-surface.ts:1199-1207`.
Including Ruby privates there is correct and load-bearing; the comment explains
what breaks without it.
