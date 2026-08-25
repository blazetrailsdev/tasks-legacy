---
title: "Port activerecord-cli's tsc-wrapper to the TS 7 API"
status: blocked
updated: 2026-08-25
rfc: "0000-typescript-7-reevaluation"
cluster: build-infra
packages: ["activerecord-cli"]
deps: ["recheck-ts7-api-surface"]
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: "TypeScript 7.1 stable — scheduled 2026-11-10 (microsoft/TypeScript#63703). Also gated on the repo actually deciding to migrate, which requires the trails-tsc blocker to clear."
---

## Context

Per RFC `0000-typescript-7-reevaluation`'s API-surface mapping (2026-08-25),
`activerecord-cli`'s `tsc-wrapper` is **not** the blocker — it is plausibly
migratable on TS 7.1. Its compiler use, by file:

- `schema-ts-parser.ts` (182 LOC) and `schema-ts-model-parser.ts` (212 LOC) —
  pure AST walks: node type guards, `forEachChild`, `SyntaxKind`,
  `ScriptTarget`. **All available in `typescript/unstable/ast` today (7.0.2).**
- `auto-import.ts` (123 LOC) — same shape.
- `cli.ts` (409 LOC) — `findConfigFile`, `getPreEmitDiagnostics`,
  `formatDiagnostics`, `formatDiagnosticsWithColorAndContext`,
  `flattenDiagnosticMessageText`, `sortAndDeduplicateDiagnostics`, `ts.sys`.

Translation notes from the RFC's mapping:

- `ts.createSourceFile(text, …)` has **no TS 7 equivalent** — `ast/factory`'s
  `createSourceFile` builds from statements and `ast/scanner` is token-level.
  Parsing a standalone file means routing through a `Project` over
  `createVirtualFileSystem` (`typescript/unstable/fs`).
- `getPreEmitDiagnostics` composes from `Program.getSyntacticDiagnostics` +
  `getSemanticDiagnostics` + `getDeclarationDiagnostics` +
  `getConfigFileParsingDiagnostics`.
- `readConfigFile` / `parseJsonConfigFileContent` / `parseCommandLine` are on
  `API` in the 7.1 nightly.
- The four diagnostic **formatting** helpers have no TS 7 equivalent at all and
  must be reimplemented (~100 LOC).
- `ts.sys` → `node:fs`; `ExitStatus` → a 3-member local enum.
- `forEachChild` is now a **method on `Node`**, not a free function.

`tsc-wrapper` drives the virtualized DX type tests
(`pnpm test:types:virtualized`), whose CI job medians 1.4m over 155 runs.

## Acceptance criteria

- [ ] `packages/activerecord-cli` imports no `typescript` 5.x API; its only
      `typescript` dependency is the 7.x line.
- [ ] `pnpm test:types:virtualized` passes, producing the same pass/fail verdict
      per fixture as it does on 5.9.3.
- [ ] `schema-ts-parser.test.ts`, `schema-ts-model-parser.test.ts` and
      `cli.test.ts` pass unchanged — the parsers' outputs are unchanged.
- [ ] Diagnostic output is human-legible: file, line, column, code, message.
      Exact byte-for-byte match with `tsc`'s formatting is not required, but any
      deliberate difference is noted in the PR body.

## Definition of done

Shelling out to the `tsc` binary and scraping stdout does not close this story —
the point is to use the API, so the wrapper can keep injecting virtual files.

## Verification

```bash
pnpm test:types:virtualized
pnpm vitest run packages/activerecord-cli/src/tsc-wrapper/
pnpm why typescript   # expect: no 5.x resolved for activerecord-cli
```
