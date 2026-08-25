---
title: "Port trails-tsc to the TS 7 API (the split-env blocker)"
status: blocked
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["trails-tsc"]
deps: ["recheck-ts7-api-surface"]
deps-rfc: []
est-loc: 600
priority: null
pr: null
claim: null
assignee: null
blocked-by: "No TS 7 equivalent for programmatic --build (createSolutionBuilder) or LanguageService plugin hosting, as of typescript@7.0.2 and 7.1.0-dev.20260825.1 (verified 2026-08-25); neither is a line item in the 7.1 iteration plan (microsoft/TypeScript#63703). Two escapes were measured and ruled out: shelling out to tsc --build cannot carry the virtualizing host, and rebuilding the build on the API loses reference redirection (712 diagnostics vs 2), up-to-date checking, ordering, and emit."
---

## Context

**Not on the ground-floor path (2026-08-25).** `@blazetrails/trails-tsc`
publishes only the `trails-tsc-views` bin, serving the TSE views pipeline, which
is roadmap-stage (ActionView 8.2% of API surface, P3). The user-facing
`trails-tsc` typecheck bin belongs to `activerecord-cli`, not this package —
`src/cli.ts:6-10` says otherwise and is stale. So this package keeps a pinned
5.x and does **not** block declaring TypeScript 7 as trails' floor.

## Original context

This is **the** blocker described in RFC `0000-typescript-7-ground-floor`.
`trails-tsc` is the single package that forces a split TS 5.x + TS 7
environment, which is the shape the maintainer rejected on
[tasks PR #59](https://github.com/blazetrailsdev/tasks/pull/59) (2026-07-22:
_"Not interested in the split env, will wait for native APIs in v7."_).

Per-module status from the RFC's API-surface mapping (verified 2026-08-25
against `typescript@7.0.2` and `typescript@7.1.0-dev.20260825.1`):

| Module                                  | TS 5.x API used                                                                                                          | TS 7 path                                                                                                          |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `src/host.ts`                           | `CompilerHost`, `createIncrementalCompilerHost`, `createSourceFile`                                                      | reworkable onto `unstable/fs` `createVirtualFileSystem` + a `Project`                                              |
| `src/program.ts`                        | `createProgram`, `findConfigFile`, `readConfigFile`, `parseJsonConfigFileContent`, `ts.sys`                              | reworkable onto `API.updateSnapshot()` + 7.1's config methods                                                      |
| `src/build-views.ts`                    | `createProgram`, `createCompilerHost`, `getPreEmitDiagnostics`, formatting helpers                                       | reworkable                                                                                                         |
| `src/plugins/tse-diagnose.ts`           | `createProgram`, `createCompilerHost`, `createSourceFile`, `flattenDiagnosticMessageText`                                | reworkable                                                                                                         |
| **`src/build.ts`**                      | **`createSolutionBuilder`, `createSolutionBuilderHost`, `createEmitAndSemanticDiagnosticsBuilderProgram`, `ExitStatus`** | **none — no programmatic `--build` in TS 7**                                                                       |
| **`src/lsp-plugin.ts`** (`./ts-plugin`) | **`LanguageServiceHost`, `ScriptSnapshot`, decorating `LanguageService`**                                                | **none — 7.1's `LanguageService` is a 5-method server-owned client with no host injection and no plugin protocol** |

The four reworkable modules are real work but ordinary — and the virtual FS
makes them more tractable than the table suggests: `unstable/fs`'s `readFile`
hook is a direct analogue of `buildCompilerHost`'s source-text interception
(measured: opening all 18 project configs fired 16,233 `readFile` callbacks into
JS). The two bolded ones have no upstream story, and two candidate escapes were
measured and ruled out (2026-08-25):

- **Shell out to `tsc --build`.** Structurally impossible. `build.ts` is a
  _virtualizing_ build — plugins rewrite source text before the compiler sees it
  and `remapDiagnostics` maps coordinates back. The `tsc` CLI compiles what is on
  disk and exposes no filesystem hook.
- **Rebuild the solution builder on the API.** The API returns projects, not a
  build. Opening the root `tsconfig.json` yields 1 project with 0 root files;
  opening all 18 explicitly works but loses reference redirection —
  `activerecord`'s semantic check reports **712 diagnostics against
  `tsc --build`'s 2**, resolving `node_modules/@blazetrails/arel/src/*.ts`
  instead of `../arel/dist/*.d.ts`. No up-to-date checking, no ordering, no emit
  before 7.1.

## Acceptance criteria

- [ ] `packages/trails-tsc` imports no `typescript` 5.x API.
- [ ] `trails-tsc build` drives a project-references build with per-project
      incremental behaviour equivalent to today's solution-builder path.
- [ ] The `./ts-plugin` LSP export still decorates the editor language service
      (TSE files resolve, diagnostics remap), or the RFC's Non-goals are amended
      to drop it with a stated reason.
- [ ] `pnpm vitest run packages/trails-tsc/` passes.
- [ ] `pnpm why typescript` resolves exactly one `typescript` version repo-wide.

## Definition of done

Keeping a pinned TypeScript 5.x for this package does **not** close this story —
that is the split env, and it is the thing being rejected. If the two gaps have
not closed, `pnpm tasks block` this story with the specific missing API; do not
close it by narrowing the acceptance criteria.

## Verification

```bash
pnpm vitest run packages/trails-tsc/
pnpm why typescript          # expect exactly one 7.x, no 5.x
node -e "require('typescript/unstable/sync')"   # the API this package now uses
```
