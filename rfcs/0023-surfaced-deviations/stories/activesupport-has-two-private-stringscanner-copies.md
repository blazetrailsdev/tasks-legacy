---
title: "Two private StringScanner copies model Ruby's strscan; fold into one Ruby-core module"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby's `strscan` (`StringScanner`) is stdlib, and Rails ports reach for it
directly: `activesupport/lib/active_support/duration/iso8601_parser.rb:3`
(`require "strscan"`, then `scanner.scan` / `scanner.eos?` / `scanner[1]` /
`scanner.matched` / `scanner.string`) and
`activesupport/lib/active_support/core_ext/erb/util.rb:159-207`
(`ERB::Util.tokenize`, which also uses `scan_until` / `exist?` / `rest` /
`terminate` / `pos`).

trails now carries TWO hand-rolled private copies of the same class:

- `packages/activesupport/src/core-ext/tse/util.ts:134-179` (added first)
- `packages/activesupport/src/duration/iso8601-parser.ts:12-43` (added by
  PR #6909, which could not import the first — it is not exported, and the two
  need different member subsets: this one needs `scanner[n]` group access,
  which the tse one has no equivalent for)

Two divergent models of the same Ruby stdlib class is exactly the shape
`ruby-empty.ts` was created to avoid for `empty?` — its own header says it
exists so "an `empty?` in an activesupport body spells the same call an
activerecord one does". `pos` semantics (UTF-16 code units vs Ruby bytes) are
reasoned about independently in each copy today, so a third port would re-derive
them again.

## Converged shape

One `string-scanner.ts` in activesupport modelling Ruby's `StringScanner` at
its Ruby names — the union of what the two call sites use: `scan`,
`scan_until`, `matched`, `[n]`, `eos?`, `exist?`, `rest`, `terminate`, `pos`,
`string` — the sibling of `ruby-empty.ts` / `ruby-truthy.ts` (a Ruby-core
model, not a Rails class, so it carries a `@noRailsEquivalent` header the way
`ruby-empty.ts` does). Both call sites import it and delete their copy; the
UTF-16-vs-bytes note lives in one place.

Do NOT invent members neither Ruby call site uses.

## Acceptance criteria

- [ ] `core-ext/tse/util.ts` and `duration/iso8601-parser.ts` both import the
      shared scanner; both local classes deleted.
- [ ] `pnpm vitest run packages/activesupport/src/core-ext/tse/util.test.ts
packages/activesupport/src/core-ext/duration.test.ts` green.
- [ ] `pnpm parity:api:extra --package activesupport` shows no new novel
      surface (the module is `@noRailsEquivalent`-tagged Ruby-core modelling).
- [ ] No import cycle introduced — the module has no runtime imports.
