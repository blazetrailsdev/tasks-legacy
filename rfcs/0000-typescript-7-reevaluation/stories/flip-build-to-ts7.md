---
title: "Flip the pinned typescript to 7.x and drop TypeScript 5.x"
status: blocked
updated: 2026-08-25
rfc: "0000-typescript-7-reevaluation"
cluster: build-infra
packages:
  ["activerecord", "activesupport", "activemodel", "trailties", "trails-tsc", "activerecord-cli"]
deps:
  [
    "port-trails-tsc-to-ts7-api",
    "port-tsc-wrapper-to-ts7-api",
    "fix-anonymous-class-declaration-emit",
  ]
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: "Depends on port-trails-tsc-to-ts7-api, which is itself blocked on TS 7 shipping a programmatic --build and LS plugin hosting. Also requires TS 7.1 stable (scheduled 2026-11-10)."
---

## Context

The terminal story of RFC `0000-typescript-7-reevaluation`. Once no package
needs the TypeScript 5.x programmatic API, the migration is a **single-compiler
swap with no split env** — which is the only form the maintainer has not
rejected.

The spike in the RFC (2026-08-25, `typescript@7.0.2` run over this repo's real
18-project graph) already established what this costs:

- **Diagnostics:** 2, both TS4094 in `trailties/src/application.ts` — retired by
  `fix-anonymous-class-declaration-emit`. Otherwise **zero** across 3,472 files.
- **`types: []`** (TS 7's changed default, RFC #59's headline risk): **zero**
  hits. No package here relies on ambient `@types/*` auto-inclusion.
- **`.d.ts` shape:** 14 of 3,338 files differ (0.42%), in four benign classes —
  member reordering, accessors preserved rather than collapsed, a type alias
  preserved rather than expanded, and JSDoc retained. Full list in the RFC.
- **Speed:** cold full-monorepo `tsc --build` 215.0s → 20.9s (10.3×);
  `tsc -b packages/activerecord` 102.8s → 19.9s (5.2×). Both on a host at load
  41–59, so treat the ratio as the reliable figure.

Note for the `.d.ts` review: TS 7 retains `/** @internal */` in emitted
declarations where TS 5.9.3 drops it. `parity:api:extra` and
`blazetrails/unbacked-internal-needs-receipt` read **source**, not emit, so this
is expected to be inert — confirm it rather than assume it (RFC open question 4).

## Acceptance criteria

- [ ] Root `package.json` pins `typescript` at an exact 7.x; no package declares
      a 5.x dependency or peer range that resolves to 5.x.
- [ ] `pnpm build`, `pnpm typecheck`, `pnpm test:types:virtualized` and
      `pnpm guides:typecheck` are green.
- [ ] The `.d.ts` shape delta versus the last 5.9.3 build is reviewed file by
      file and each entry is either fixed or recorded in the PR body as
      understood and consumer-safe.
- [ ] `parity:api`, `parity:api:calls`, `parity:api:calls:args`,
      `parity:api:extra:gate` and the `rails-comparison` lint rules behave
      identically — in particular the `@internal` emit change is confirmed inert.
- [ ] `scripts/typecheck.mjs`'s "~60s cold" comment is corrected to the real
      measured number for the new compiler.
- [ ] CONTRIBUTING.md / CLAUDE.md build notes reflect the new compiler.

## Definition of done

A flip that leaves any `typescript` 5.x in the lockfile does not close this
story — the whole point is that there is exactly one compiler.

## Verification

```bash
pnpm why typescript            # expect exactly one 7.x entry
pnpm build && pnpm typecheck
pnpm test:types:virtualized
pnpm parity:api:calls && pnpm parity:api:calls:args && pnpm parity:api:extra:gate
```

Re-measure and record cold/warm wall-clock the same way the RFC did, stating the
host load average alongside each number.
