---
title: "Port activerecord-cli's tsc-wrapper to the TS 7 API"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["activerecord-cli"]
deps: []
deps-rfc: []
est-loc: 250
priority: 3
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

Per RFC `0000-typescript-7-ground-floor`'s API-surface mapping (2026-08-25),
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

- `ts.createSourceFile(text, …)` routes through a `Project` over
  `createVirtualFileSystem` (`typescript/unstable/fs`). **Verified working on
  7.0.2** (RFC § "What the virtual FS closes"): identical AST (5,686 nodes on a
  2,034-line file), guards and `node.forEachChild` intact, and the real-FS +
  in-memory overlay pattern this package needs correctly type-checks synthesized
  files against the on-disk project. Parse is 5.5ms vs 5.9.3's 71.0ms; the dense
  walk is 2.5× slower; a one-time ~99ms `API` spawn applies per process.
- `getPreEmitDiagnostics` composes from `Program.getSyntacticDiagnostics` +
  `getSemanticDiagnostics` + `getDeclarationDiagnostics` +
  `getConfigFileParsingDiagnostics`.
- `readConfigFile` / `parseJsonConfigFileContent` / `parseCommandLine` are on
  `API` in the 7.1 nightly.
- Diagnostics arrive pre-flattened as
  `{ fileName, pos, end, code, category, text }`, so
  `flattenDiagnosticMessageText` is unnecessary and `formatDiagnostics` is a
  short reimplementation over `computeLineStarts` (`unstable/ast/scanner`).
- `ts.sys` → `node:fs`; `ExitStatus` → a 3-member local enum.
- `forEachChild` is now a **method on `Node`**, not a free function.

`tsc-wrapper` drives the virtualized DX type tests
(`pnpm test:types:virtualized`), whose CI job medians 1.4m over 155 runs.

**This story does not need 7.1 and does not need a repo-wide TS 7 decision.**
It is worth landing on its own: it removes one of the four packages that would
otherwise force a split environment.

Note the API is exported under `unstable/` — no semver guarantee. Acceptable for
internal tooling like this; the same is not automatically true of published
surface (see `port-type-virtualization-to-ts7-api`).

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
