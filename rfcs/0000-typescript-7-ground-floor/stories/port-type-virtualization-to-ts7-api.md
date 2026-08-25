---
title: "Port activerecord's type-virtualization to the TS 7 API"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

`packages/activerecord/src/type-virtualization/` imports `typescript` at 8 sites
(`walker.ts`, `virtualize.ts`, `synthesize.ts`, `transitive-extends-walker.ts`,
plus `scripts/materialize-model-declares.ts` and a test). It uses
`createSourceFile`, `createScanner`, `getLeadingCommentRanges`, `getModifiers`,
`canHaveModifiers`, `forEachChild` and ~20 node type guards.

RFC `0000-typescript-7-ground-floor` found this package while re-checking the
consumer count — #59's framing named only `trails-tsc` and `activerecord-cli`,
and this one was missed. It is one of four packages that would otherwise force a
split environment, and per the RFC's § "What the virtual FS closes" it is
**migratable today on `typescript@7.0.2`**: everything it uses is available in
`typescript/unstable/ast`, and text parsing works through a `Project` over
`createVirtualFileSystem` (verified — identical AST, 5,686 nodes on a
2,034-line file).

**What makes this one different from `port-tsc-wrapper-to-ts7-api`:** this is
**published surface.** `./type-virtualization/*.js` is an `exports` subpath of
`@blazetrails/activerecord`, backed by the peer dependency
`typescript: ">=5.0.0"`. Two consequences that need a decision, not just a port:

1. **The TS 7 API is out-of-process.** It spawns a Go server (~99ms) and talks
   over IPC, where today's `createSourceFile` parses in-process. For a package
   our users install, that is a deployment change, not a pure win.
2. **The API is exported under `unstable/`** — no semver guarantee. Building
   published surface on an unstable subpath is a materially bigger bet than
   building internal tooling on it.

Also worth settling regardless of the outcome: the peer range `>=5.0.0` is
already wrong. A user on TS 7 cannot use this subpath today, because
`ts.createSourceFile`-from-text does not exist there. The range should say what
is actually supported.

## Acceptance criteria

- [ ] A decision is recorded (in this story or the RFC) on whether
      `type-virtualization` should consume the TS 7 API directly, keep a pinned
      5.x, or stop being published surface — with the out-of-process and
      `unstable/` considerations weighed explicitly.
- [ ] If the port proceeds: no `typescript` 5.x API import remains under
      `src/type-virtualization/`, and `virtualize.trails.test.ts` passes with
      unchanged output.
- [ ] `activerecord`'s `typescript` peer range states what is actually
      supported, whatever the outcome.
- [ ] `packages/activerecord-cli`'s consumers of
      `@blazetrails/activerecord/type-virtualization/*` still work
      (`ar-program.ts`, `cli.ts`, `ar-models-plugin.ts`, `auto-import.ts`).

## Definition of done

Porting the code while leaving the peer range at `>=5.0.0` does not close this
story — the range is part of the defect.

Deciding "keep 5.x" is a legitimate close, provided the reasoning is recorded.
This story is about resolving the question, not about forcing a port.

## Verification

```bash
pnpm vitest run packages/activerecord/src/type-virtualization/
pnpm test:types:virtualized
grep -rn 'from "typescript"' packages/activerecord/src/type-virtualization/
```
