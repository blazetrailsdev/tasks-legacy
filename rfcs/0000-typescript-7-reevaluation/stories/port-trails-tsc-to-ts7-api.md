---
title: "Port trails-tsc to the TS 7 API (the split-env blocker)"
status: blocked
updated: 2026-08-25
rfc: "0000-typescript-7-reevaluation"
cluster: build-infra
packages: ["trails-tsc"]
deps: ["recheck-ts7-api-surface"]
deps-rfc: []
est-loc: 600
priority: null
pr: null
claim: null
assignee: null
blocked-by: "No TS 7 equivalent exists for programmatic --build (createSolutionBuilder) or for LanguageService plugin hosting, as of typescript@7.0.2 and 7.1.0-dev.20260825.1 (verified 2026-08-25). Neither is a line item in the TS 7.1 iteration plan (microsoft/TypeScript#63703). This story is not actionable until both gaps close."
---

## Context

This is **the** blocker described in RFC `0000-typescript-7-reevaluation`.
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

The four reworkable modules are real work but ordinary. The two bolded ones
have no upstream story.

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
