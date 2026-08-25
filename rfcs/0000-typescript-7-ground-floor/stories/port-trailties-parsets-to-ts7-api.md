---
title: "Reimplement trailties' parseTs() on the TS 7 API"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["trailties"]
deps: []
deps-rfc: []
est-loc: 40
priority: 2
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

`packages/trailties/src/template-builder/testing.ts` is the whole of trailties'
compiler-API usage — one exported function:

```ts
export function parseTs(source: string): { diagnostics: readonly ts.Diagnostic[] } {
  const result = ts.transpileModule(source, { reportDiagnostics: true, ... });
  return { diagnostics: result.diagnostics ?? [] };
}
```

It is **published surface** (`./template-builder/testing` is an `exports`
subpath) — a helper users import to assert generated templates parse. It uses
`transpileModule` purely as a way to get syntactic diagnostics out of a string;
the comment says as much.

`transpileModule` does not exist in TS 7.0.2 (it arrives in the 7.1 nightly).
But the function's actual need — syntactic diagnostics from a source string —
is served on 7.0.2 by a `Project` over `createVirtualFileSystem` plus
`Program.getSyntacticDiagnostics`. RFC `0000-typescript-7-ground-floor`
verified this by reimplementing it:

| input                         | 7.0.2 reimplementation                         |
| ----------------------------- | ---------------------------------------------- |
| `export const x: number = 1;` | 0 diagnostics                                  |
| `export const x: number = ;`  | 1 — `TS1109 Expression expected.`              |
| Ruby source (`def foo … end`) | 2 — `TS1434 Unexpected keyword or identifier.` |

That last case matters: this helper sits next to `assertNoRubySource`, so
"Ruby pasted into a template" is a real input it must keep rejecting.

This is the smallest of the three ground-floor ports and the one that removes
trailties' only reason to require TypeScript 5.

## Acceptance criteria

- [ ] `parseTs()` no longer imports the TypeScript 5.x API.
- [ ] It returns equivalent diagnostics to the 5.9.3 implementation for valid
      TypeScript, syntactically-invalid TypeScript, and Ruby source.
- [ ] The published signature is unchanged, or the change is deliberate and
      noted — this is user-facing surface.
- [ ] The one-time `API` spawn cost (~99ms) is not paid per call in a loop; a
      caller asserting many templates should not pay it repeatedly.

## Definition of done

Waiting for 7.1's `transpileModule` does not close this story — the point is
that 7.0.2 already suffices, and this is what lets `@blazetrails/trailties` drop
its `^5.0.0` peer.

## Verification

```bash
pnpm vitest run packages/trailties/src/template-builder/
grep -rn 'from "typescript"' packages/trailties/src/
```
