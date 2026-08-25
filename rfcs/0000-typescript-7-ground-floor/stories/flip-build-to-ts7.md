---
title: "Flip the pinned typescript to 7.x and drop TypeScript 5.x"
status: draft
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
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
blocked-by: null
---

## Context

The terminal story of RFC `0000-typescript-7-ground-floor`. Once no package
needs the TypeScript 5.x programmatic API, the migration is a **single-compiler
swap with no split env** — which is the only form the maintainer has not
rejected.

The spike in the RFC (2026-08-25, `typescript@7.0.2` run over this repo's real
18-project graph) already established what this costs:

- **Diagnostics on the 7.1 target:** 10, from **2 root causes** — the `TS2883`
  in `activesupport/src/yaml.ts` (which cascades into 7) and the `TS4094` pair in
  `trailties/src/application.ts`. Both retired by this story's deps. On 7.0.2
  only the TS4094 pair appears; `TS2883` is a 7.1-line check.
- **`types: []`** (TS 7's changed default, RFC #59's headline risk): **zero**
  hits. No package here relies on ambient `@types/*` auto-inclusion.
- **`.d.ts` shape:** 14 of 3,338 files differ (0.42%), in four benign classes —
  member reordering, accessors preserved rather than collapsed, a type alias
  preserved rather than expanded, and JSDoc retained. Full list in the RFC.
- **Speed** (quiet host, load 3–5): cold full-monorepo `tsc --build`
  **91.75s → 9.73s (7.0.2) → 8.47s (7.1-dev)**; `tsc -b packages/activerecord`
  **69.92s → 8.34s**. Warm no-op 0.47s; incremental after one AR edit 6.36s.

Note for the `.d.ts` review: TS 7 retains `/** @internal */` in emitted
declarations where TS 5.9.3 drops it. `parity:api:extra` and
`blazetrails/unbacked-internal-needs-receipt` read **source**, not emit, so this
is expected to be inert — confirm it rather than assume it (RFC open question 4).

## Acceptance criteria

- [ ] Root `package.json` pins `typescript` at an **exact** version on the 7.1
      line — a specific `7.1.0-dev.*` nightly, never the `next` tag — with a note
      that it moves to 7.1 stable on 2026-11-10.
- [ ] The only remaining 5.x resolution is `@blazetrails/trails-tsc`'s, via an
      explicit alias (e.g. `typescript-5@npm:typescript@5.9.3`), scoped to its
      views pipeline and documented at the declaration.
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

A flip that leaves 5.x resolving for any package **other than**
`@blazetrails/trails-tsc` does not close this story. `trails-tsc`'s aliased 5.x
is expected and scoped (RFC § Non-goals); anything beyond it is the split this
RFC is avoiding.

## Verification

```bash
pnpm why typescript            # expect 7.x everywhere except trails-tsc's alias
pnpm build && pnpm typecheck
pnpm test:types:virtualized
pnpm parity:api:calls && pnpm parity:api:calls:args && pnpm parity:api:extra:gate
```

Re-measure and record cold/warm wall-clock the same way the RFC did, stating the
host load average alongside each number.
