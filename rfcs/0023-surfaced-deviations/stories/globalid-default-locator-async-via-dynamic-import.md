---
title: "GlobalID.defaultLocator is async; converge via a zero-import Locator slot"
status: draft
updated: 2026-08-08
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`GlobalID.default_locator` is a plain synchronous setter in Rails —
`def default_locator(default_locator); Locator.default_locator = default_locator; end`
(`vendor/globalid/lib/global_id/global_id.rb:34-36`).

trails ships it `async` (`packages/globalid/src/global-id.ts`, ported in
PR #6221) because `global-id.ts` reaches `Locator` through a **dynamic import**
— the same device `GlobalID#find` uses (`global-id.ts`, ~:196) — to avoid a
runtime edge into the `global-id ↔ signed-global-id ↔ locator` cycle that would
evaluate `class SignedGlobalID extends GlobalID` with `GlobalID` still in TDZ.
An awaited setter is a real API divergence: Ruby callers write
`GlobalID.default_locator(x)` and read `Locator.default_locator` on the next
line.

## Converged shape

CLAUDE.md's settled answer for exactly this shape is the **zero-import slot
module** (the two existing instances: `activerecord/src/encryption/
configurable-slot.ts`, `activerecord/src/associations/collection-proxy-slot.ts`)
— a file with no runtime imports exporting a mutable binding plus a `_setX()`
setter, which `locator.ts` calls at the bottom of its own body. `global-id.ts`
then imports the slot (which joins no cycle) and both `defaultLocator` and
`find` become synchronous-at-the-Rails-shape reads resolved at call time,
exactly where Ruby resolves the constant.

Verify both directions with a plain-node import of the **built** `dist/**.js`
modules as entry modules — a vitest run enters the funnel module first and masks
the TDZ, so a green suite proves nothing here.

## Acceptance criteria

- [ ] `GlobalID.defaultLocator(locator)` is synchronous, matching
      `global_id.rb:34-36`.
- [ ] `GlobalID#find`'s dynamic import is retired by the same slot, or the
      story says explicitly why it must stay.
- [ ] TDZ verified by plain-node import of `packages/globalid/dist/global-id.js`
      and `dist/signed-global-id.js` as entry modules, both orders.
- [ ] globalid suite green; `parity:api --package globalid` stays 80/80.
