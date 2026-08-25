---
rfc: "0000-typescript-7-reevaluation"
title: "TypeScript 7 re-evaluation: measured cost of waiting, and the remaining blocker"
status: draft
created: 2026-08-25
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "trails-tsc"
  - "activerecord-cli"
  - "trailties"
  - "activerecord"
  - "activesupport"
  - "activemodel"
clusters:
  - "build-infra"
  - "developer-experience"
---

<!-- Unnumbered until merge: dir stays `0000-typescript-7-reevaluation`,
     `rfc:` stays `0000-...`, H1 stays number-free.
     `scripts/finalize-rfc.mjs` assigns the number at merge. -->

# RFC — TypeScript 7 re-evaluation: measured cost of waiting, and the remaining blocker

## Recommendation

**Wait. Do not migrate yet.** Re-evaluate at **TypeScript 7.1 beta
(2026-09-09)** and decide at **7.1 stable (target 2026-11-10)**.

The named condition is narrower than it was in July, and now has exactly two
items, both owned by `trails-tsc`:

1. A programmatic **`--build` / solution-builder** equivalent
   (`createSolutionBuilder`, `createSolutionBuilderHost`,
   `createEmitAndSemanticDiagnosticsBuilderProgram`).
2. **Language-Service plugin hosting** — a way to inject a host and decorate a
   `LanguageService`, which is what `trails-tsc`'s `./ts-plugin` export is.

Neither exists in TS 7.0.2 or in the 7.1 nightly as of 2026-08-25, and neither
appears as a line item in the TS 7.1 iteration plan. Until they land, adopting
TS 7 still requires pinning TypeScript 5.x alongside it — the **split env** the
maintainer rejected on [tasks PR #59][pr59] (2026-07-22: _"Not interested in the
split env, will wait for native APIs in v7."_). That objection is unchanged in
kind, but it now applies to **one** package rather than two.

This is a decision record, not a migration plan. It ships so the next
re-evaluation starts from measured data instead of redoing this work. The one
piece of work it does schedule is a single unblocking fix
(`fix-anonymous-class-declaration-emit`) that is worth doing on its own merits.

## Motivation — what we actually pay for not upgrading

RFC #59 asserted that typecheck is "the CI long pole" and that the cold
pre-commit path costs "~60s". **Both claims are wrong.** They were never
measured — Phase 0 was the measuring step and never ran. Here are the
measurements.

Every number below records its method so it can be re-run. Numbers marked
_(contended)_ were taken on a host at load average 41–59 on 24 cores, because
the agent fleet was live; they are **upper bounds** on the trails side and the
TS 5 / TS 7 ratio is the trustworthy figure, not the absolute.

### CI is not the case for this migration

Method: `gh api repos/blazetrailsdev/trails/actions/workflows/242258170/runs`
over the 500 most recent completed `pull_request` runs (spanning 15 days,
2026-08-10 → 2026-08-24), then `/jobs` for the 200 most recent
success-or-failure runs (170 resolved). Durations are
`completed_at - started_at` per job.

| Job                                | n   | median                    | p90   |
| ---------------------------------- | --- | ------------------------- | ----- |
| Active Record SQLite Tests         | 155 | **10.9m**                 | 12.1m |
| Active Record PostgreSQL Tests (2) | 140 | 8.6m                      | 9.4m  |
| Active Record MariaDB Tests (1)    | 140 | 8.1m                      | 8.7m  |
| Unit Tests                         | 149 | 4.6m                      | 5.4m  |
| Rails API/Test Comparison          | 168 | 3.8m                      | 4.7m  |
| **Build & Type Check**             | 170 | **1.4m**                  | 1.6m  |
| **Virtualized DX Type Tests**      | 155 | **1.4m**                  | 1.5m  |
| Guides Code Type Check             | 170 | _skipped in 170/170 runs_ | —     |
| **whole PR run, wall-clock**       | 152 | **12.9m**                 | 18.4m |

Two facts kill the CI argument:

- **`Build & Type Check` is not on the critical path.** Every job in
  `.github/workflows/ci.yml` declares `needs: changes` and nothing else — they
  all fan out from `Detect changed paths` in parallel. `Build & Type Check`
  blocks no other job. A 10× speedup on a 1.4m job that runs alongside a 10.9m
  job saves **0 minutes of PR turnaround**.
- **`Guides Code Type Check` never runs.** It was skipped in all 170 sampled
  runs (it is gated on a label/path). It costs nothing today.

The only real CI saving is runner-minutes:

- Total runner time per PR run: **50.1m** across all jobs.
- `Build & Type Check` + `Virtualized DX Type Tests`: **2.55m per run = 5.1%**
  of runner time.
- Merge volume: 500 runs / 15 days = **33 PR runs/day ≈ 233/week**.
- Aggregate: **≈ 9.9 runner-hours/week** on typecheck jobs. A 10× compiler
  would recover **≈ 8.9 runner-hours/week** and zero wall-clock.

At GitHub-hosted `ubuntu-latest` list pricing ($0.008/min for a 2-core runner),
8.9 h/week ≈ **$4/week ≈ $220/year**. _(Unmeasured: whether these jobs run on
hosted or self-hosted runners, and the actual runner SKU. If self-hosted, the
dollar figure is zero and the saving is purely host capacity.)_

### Local compile cost — the "~60s" claim is wrong

Method: in a clean worktree at `origin/main`, `find … -name '*.tsbuildinfo'
-delete && rm -rf packages/*/dist scripts/dist` for cold runs, then
`/usr/bin/time -f '%e %M'`. TS 5 is the repo's pinned `typescript@5.9.3`
(root `package.json`); TS 7 is `typescript@7.0.2` installed in a throwaway dir
outside the tree and invoked by explicit path. _(contended, see above)_

| Scenario                                                                        | TS 5.9.3                      | TS 7.0.2            | speedup   |
| ------------------------------------------------------------------------------- | ----------------------------- | ------------------- | --------- |
| Cold `tsc --build`, full monorepo                                               | **215.0s** (2.88 GB peak RSS) | **20.9s** (2.98 GB) | **10.3×** |
| Cold `tsc -b packages/activerecord` (+4 upstream refs)                          | **102.8s** (2.68 GB)          | **19.9s** (2.34 GB) | **5.2×**  |
| Warm no-op `tsc --build`                                                        | 1.30s                         | not measured        | —         |
| Warm no-op `node scripts/typecheck.mjs`                                         | **0.84s**                     | not measured        | —         |
| Incremental after one real edit to `activerecord/src/relation/query-methods.ts` | **11.3s**                     | not measured        | —         |
| Incremental reverting that edit                                                 | **7.8s**                      | not measured        | —         |

**Correction to RFC #59.** `scripts/typecheck.mjs:16` says the cold path is
"~60s". The measured cold full-monorepo `tsc --build` is **215s** on a
contended host — the comment understates it by 3.5× — but that path is paid
**once per worktree**, not per commit. The path a developer or agent actually
pays on every commit is the incremental one, and that is **8–11s** after
touching an ActiveRecord file and **<1s** when nothing changed. Neither the
comment's 60s nor the RFC's framing of it as "a real tax" survives contact with
a measurement.

`scripts/typecheck.mjs` is a 29-line wrapper that does nothing but
`spawnSync(tsc, ["--build"])`, so its cost is exactly `tsc --build`'s.

### Editor cost — **unmeasured**

`packages/activerecord/src` is 165,308 non-test LOC (`find … -name '*.ts' -not
-name '*.test.ts' | xargs cat | wc -l`, 2026-08-25), dwarfing the next package
(`activesupport`, 40,356). Open-to-ready and "loading…" latency in an editor
were **not measured** — this work ran headless and any number would be a guess.
It is the most plausible remaining benefit and the one we have no data on.
`measure-editor-ls-latency` (below) exists to close that gap.

### The agent-fleet multiplier

Method: `ls -d /home/dean/github/blazetrailsdev/worktrees/*/` and `stat -c %Y`
on each worktree's `.git` file, 2026-08-25.

- **103 live worktrees**, oldest 2026-06-19, newest 2026-08-25 (67-day span).
- **57 of 103** have a populated `packages/activerecord/dist` — i.e. have paid
  at least one cold build.
- Creation rate: **2.1 worktrees/day** over the last 14 days (1.5/day over the
  full span).

At 215s per cold build, ~2.1 new worktrees/day is **≈7.5 min/day ≈ 0.9
hours/week** of cold-build CPU, serial-equivalent. The number is small; the
problem is that it is not serial. These builds land concurrently on a 24-core
host that CLAUDE.md already warns is saturated by parallel agents — the load
average during this RFC's own measurements was **41–59**, which is why every
local number here is a contended upper bound. TS 7 would cut peak cold-build
CPU-seconds by ~10× and shrink that contention window from ~3.5 minutes to ~20
seconds per worktree bootstrap.

**Unmeasured:** aggregate incremental-typecheck cost across the fleet. Commits
to `origin/main` run ~55/day over the last 30 days (`git log --oneline
--since='30 days ago' origin/main | wc -l` = 1658), and each pre-commit hook
pays the incremental path, but there is no instrumentation recording hook
invocations (including the ones that never reach a commit), so multiplying
55 × 8s would be a fabricated number rather than a measured one.

### Summary of the cost of waiting

| Axis              | Cost of staying on TS 5.9.3                                 |
| ----------------- | ----------------------------------------------------------- |
| PR wall-clock     | **zero** — typecheck is off the critical path               |
| CI runner time    | ≈ 8.9 runner-hours/week recoverable (5.1% of total)         |
| Local cold build  | 215s → 20.9s per worktree bootstrap, ~2.1/day               |
| Local incremental | 8–11s per commit touching AR; no TS 7 measurement yet       |
| Host contention   | real but small: ~0.9 CPU-hours/week, concentrated in bursts |
| Editor latency    | **unmeasured** — the likeliest real win, and the open gap   |

This is a genuine but modest cost. It does not justify a split environment. It
would comfortably justify a clean single-compiler migration once one is
possible.

## Design — what shipped since 2026-07-08

Every claim below is dated and sourced. Assume it has a shelf life; the last
version of this RFC had a 13-month one and its central fact went stale.

### Release state (verified 2026-08-25 via `npm view typescript`)

| Fact                    | Value                                        | Verified                 |
| ----------------------- | -------------------------------------------- | ------------------------ |
| `typescript@latest`     | **7.0.2**                                    | 2026-08-25, npm registry |
| 7.0.2 publish date      | **2026-07-08**                               | npm `time` field         |
| Patch releases since GA | **none** — 7.0.2 is still latest, 48 days on | 2026-08-25               |
| `typescript@next`       | `7.1.0-dev.20260825.1`                       | 2026-08-25               |
| 7.0.1-rc                | 2026-06-18                                   | npm `time` field         |

7.1 has **not** shipped. It exists only as nightlies.

### The 7.1 schedule is published and dated

Source: [TypeScript 7.1 Iteration Plan, microsoft/TypeScript#63703][iter]
(fetched 2026-08-25).

| Milestone      | Date           |
| -------------- | -------------- |
| Beta prep      | 2026-09-04     |
| **7.1 Beta**   | **2026-09-09** |
| RC prep        | 2026-10-16     |
| 7.1 RC         | 2026-10-20     |
| Stable prep    | 2026-11-06     |
| **7.1 Stable** | **2026-11-10** |

The plan's "Language and Compiler" section names three APIs for stabilization:
**Content Mapper API**, **Emit API**, and **Language Service API**. It does
**not** list plugin hosting, solution-builder/`--build` driving, or config
parsing as distinct objectives.

### TS 7.0.2 already ships an `unstable/` API — RFC #59's central fact is stale

The [TypeScript 7.0 GA post][ga] (2026-07-08) says TS 7.0 "does not yet ship
with an API." That is true of the _stable_ API and of the `import ts from
"typescript"` entry point — 7.0.2's `"."` export resolves to `lib/version.cjs`
and nothing else. But the package **does** ship a real, substantial programmatic
surface under `unstable/` subpaths. Verified 2026-08-25 by installing
`typescript@7.0.2` and reading `package.json#exports` plus the shipped `.d.ts`:

```jsonc
"./unstable/sync"          → dist/api/sync/api.js      (API, Snapshot, Project,
                                                        Program, Checker, Emitter)
"./unstable/async"         → dist/api/async/api.js
"./unstable/fs"            → dist/api/fs.js            (createVirtualFileSystem)
"./unstable/proto"         → dist/api/proto.js
"./unstable/ast"           → dist/ast/index.js         (SyntaxKind, ScriptTarget,
                                                        all node type guards)
"./unstable/ast/{is,factory,utils,scanner,visitor,clone}"
```

Shape: it is an **out-of-process RPC client**. `new API()` spawns/connects to
the Go server; `updateSnapshot()` returns a `Snapshot` of `Project`s, each with
a `Program` and a `Checker`. `unstable/fs`'s `createVirtualFileSystem(files)`
delegates `readFile`/`fileExists`/`directoryExists`/`getAccessibleEntries`/
`realpath` back to JS — this is the replacement for `ts.CompilerHost`'s
in-memory-overlay role, and it is what the virtualized DX type tests need.

The 7.1 nightly extends this materially. Diffing the exported-name sets of
`7.0.2` and `7.1.0-dev.20260825.1` (`dist/api/**/*.d.ts` + `dist/ast/**/*.d.ts`)
shows **115 new exported names**, including a `LanguageService` class,
`Program.emit(emitOnly?)`, `EmitResult`/`EmitOutput`, `API.transpileModule` /
`transpileDeclaration` (+ `*FromFile`), `API.parseCommandLine`,
`API.readConfigFile`, `API.parseJsonConfigFileContent`, `ParsedCommandLine`,
`TextEdit`, and an import-adder. The API is under active build-out and is
tracking exactly the gaps below.

### API-surface mapping for our two consumers

Method: `grep -rhoE '\bts\.[A-Za-z][A-Za-z0-9_]*'` across
`packages/trails-tsc/src` and `packages/activerecord-cli/src` (2026-08-25),
then each distinct symbol checked against the exported-name index extracted
from the shipped `.d.ts` of `typescript@7.0.2` and `typescript@7.1.0-dev.20260825.1`,
with runtime spot-checks via `require("typescript/unstable/ast")`.

`✓` available · `~` reachable with rework · `✗` no equivalent

| TS 5.x symbol                                                                                                                | 7.0.2 | 7.1-dev       | TS 7 replacement / note                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------- | ----- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| node type guards (`isIdentifier`, `isCallExpression`, `isClassDeclaration`, … ×24)                                           | ✓     | ✓             | `typescript/unstable/ast` (runtime-verified)                                                                                                                                                                                                            |
| `SyntaxKind`, `ScriptTarget`, `ScriptKind`                                                                                   | ✓     | ✓             | `typescript/unstable/ast`                                                                                                                                                                                                                               |
| `ModuleKind`                                                                                                                 | ✓     | ✓             | `typescript/unstable/sync`                                                                                                                                                                                                                              |
| `forEachChild`                                                                                                               | ✓     | ✓             | now a **method on `Node`**, not a free function                                                                                                                                                                                                         |
| `Diagnostic`, `CompilerOptions`, `Node`, `SourceFile`, `Program` (types)                                                     | ✓     | ✓             | `typescript/unstable/sync` / `…/ast`                                                                                                                                                                                                                    |
| `getPreEmitDiagnostics`                                                                                                      | ✗     | ~             | compose `Program.getSyntactic/Semantic/Declaration/ConfigFileParsingDiagnostics`                                                                                                                                                                        |
| `createProgram`                                                                                                              | ✗     | ~             | `new API().updateSnapshot()` → `Project` → `Program`                                                                                                                                                                                                    |
| `createCompilerHost` / `createIncrementalCompilerHost`                                                                       | ✗     | ~             | `unstable/fs` `createVirtualFileSystem` — a delegating FS, not a `CompilerHost`                                                                                                                                                                         |
| `readConfigFile`, `parseJsonConfigFileContent`, `findConfigFile`                                                             | ✗     | ✓ (first two) | `API.readConfigFile` / `parseJsonConfigFileContent` / `parseCommandLine`; `findConfigFile` is a trivial upward walk                                                                                                                                     |
| `formatDiagnostics`, `formatDiagnosticsWithColorAndContext`, `flattenDiagnosticMessageText`, `sortAndDeduplicateDiagnostics` | ✗     | ✗             | pure formatting helpers; ~100 LOC to reimplement                                                                                                                                                                                                        |
| `ts.sys`, `getDefaultLibFilePath`, `ExitStatus`                                                                              | ✗     | ✗             | trivial (`node:fs`, a path join, a 3-member enum)                                                                                                                                                                                                       |
| **`createSourceFile` (parse from text)**                                                                                     | ✗     | ✗             | **no standalone text parser.** `ast/factory`'s `createSourceFile` builds from statements; `ast/scanner` is token-level only. Parsing requires routing through a `Project` over a virtual FS                                                             |
| **`createSolutionBuilder` / `createSolutionBuilderHost` / `createEmitAndSemanticDiagnosticsBuilderProgram`**                 | ✗     | ✗             | **no programmatic `--build`.** Not in the 7.1 iteration plan                                                                                                                                                                                            |
| **`LanguageServiceHost` / `ScriptSnapshot` / decorating a `LanguageService`**                                                | ✗     | ✗             | 7.1's `LanguageService` class is **server-owned with 5 methods** (`getImportAdderEdits`, `getImportEditsForSymbols`, `getReferencedSymbolsForNode`, `getSignatureUsage`, `getCompletionsAtPosition`). There is no host injection and no plugin protocol |

#### Verdict: `activerecord-cli`'s `tsc-wrapper` — **plausibly migratable on 7.1**

Its work is parse-and-diagnose. `schema-ts-parser.ts` and
`schema-ts-model-parser.ts` are pure AST walks (guards + `forEachChild` +
`SyntaxKind`) — **fully covered by 7.0.2 already**, modulo `createSourceFile`'s
missing text-parse, which a virtual-FS `Project` covers.
`auto-import.ts` is the same shape. `cli.ts`'s `getPreEmitDiagnostics` +
formatting helpers are compose-or-reimplement. No solution builder, no LS
plugin. **This package is not the blocker.**

#### Verdict: `trails-tsc` — **not migratable, and not scheduled to become so**

Two of its five compiler-touching modules have no path forward:

- `src/build.ts` drives `createSolutionBuilder` / `createSolutionBuilderHost` /
  `createEmitAndSemanticDiagnosticsBuilderProgram` — programmatic `--build`.
  No TS 7 equivalent exists and none is on the 7.1 plan.
- `src/lsp-plugin.ts` (the `./ts-plugin` export) implements the
  `LanguageServiceHost`/`ScriptSnapshot` plugin contract to decorate a
  `LanguageService`. TS 7.1's `LanguageService` is a five-method read-only
  client of the Go server; there is nothing to decorate and no host to supply.

`src/host.ts`, `src/program.ts`, `src/build-views.ts` and
`src/plugins/tse-diagnose.ts` are all `createProgram`/`CompilerHost` shaped and
would be reworkable onto the snapshot API. The two above are not.

**This is the whole remaining blocker, and it is one package.**

### Compatibility survey, re-verified 2026-08-25

RFC #59's survey is 13 months stale. Current numbers, from this worktree at
`origin/main`:

| Item                                                         | RFC #59 (2026-07-08)                      | Today (2026-08-25)   |
| ------------------------------------------------------------ | ----------------------------------------- | -------------------- |
| Root project references                                      | 15                                        | **18**               |
| `.ts` files under `packages/` (excl. `node_modules`, `dist`) | ~2,900                                    | **3,472**            |
| `activerecord` non-test src LOC                              | ~170k                                     | **165,308**          |
| Pinned `typescript`                                          | `^5.9.3`                                  | `^5.9.3` (unchanged) |
| Root `tsconfig.json`                                         | composite, Node16, strict, `.d.ts` + maps | **unchanged**        |

Root `tsconfig.json` compiler options are unchanged: `target: ES2022`,
`module`/`moduleResolution: Node16`, `strict`, `declaration`, `declarationMap`,
`sourceMap`, `isolatedModules`, `composite`, `rootDir: "."`, `skipLibCheck`,
`esModuleInterop`, `resolveJsonModule`, `forceConsistentCasingInFileNames`.

Packages declaring a `typescript` dependency: root (`^5.9.3` devDep),
`activerecord` (`>=5.0.0` peer), `trails-tsc` (`^5.0.0`),
`activerecord-cli` (`^5.9.3`), `trailties` (`^5.0.0` + peer meta).

### The spike: TS 7.0.2 actually run over this repo

This is the finding RFC #59 deferred to a Phase 0 that never happened. Method:
`typescript@7.0.2` installed in a scratch directory outside the tree, its
`bin/tsc` invoked by absolute path against the repo's own `tsconfig.json`
graph, with all `dist/` and `*.tsbuildinfo` wiped first. Nothing was committed
and the worktree was restored to a TS 5.9.3 build afterward.

**Result: it works, essentially out of the box.**

- `tsc -b packages/activerecord`: **19.9s, zero diagnostics**, 1,626 `.d.ts`
  emitted.
- `tsc --build` (full monorepo, 18 projects): **20.9s, exactly two
  diagnostics**, both the same error in the same file:

  ```text
  packages/trailties/src/application.ts(41,12): error TS4094: Property '#private'
    of exported anonymous class type may not be private or protected.
  packages/trailties/src/application.ts(45,12): error TS4094: …
  ```

  Both are `readonly executor = class extends Executor {}` and
  `readonly reloader = class extends Reloader {}` — anonymous class expressions
  whose base classes carry `#private` fields. TS 5.9.3 emits these; TS 7 refuses
  to. Fixable in a handful of lines (name the classes, or annotate the
  properties). Story below.

- **`types: []` did not bite.** RFC #59 flagged TS 7's changed `types` default
  as the likeliest breaking change. Across all 18 projects it produced **zero**
  diagnostics. No package in this repo relies on ambient `@types/*`
  auto-inclusion.

#### Declaration-emit shape diff — far smaller than feared

Method: build the full graph under each compiler, snapshot every `packages/*/
dist/**/*.d.ts`, `diff -rq`, then classify the hunks.

**14 of 3,338 `.d.ts` files differ (0.42%), plus 1 missing** (the TS4094
failure). The differences fall into four benign classes:

| Class                                                      | Files                                                                                                                                     | Example                                                                                                      |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Member **reordering** (alphabetized vs source order)       | `activemodel/validations/helper-methods.d.ts`, `activemodel/validations.d.ts`, `activerecord/relation/{query-methods,spawn-methods}.d.ts` | the 12 `validates*Of` signatures, same text, different order                                                 |
| Accessor **preserved** rather than collapsed to a property | 6 `activesupport/*-adapter.d.ts`                                                                                                          | TS 5: `adapter: string \| null;` → TS 7: `get adapter(): string \| null; set adapter(name: string \| null);` |
| Type **alias preserved** rather than expanded              | `activerecord/test-helpers/fixtures-registry.d.ts`                                                                                        | TS 5: `readonly addOn: () => Promise<void>;` → TS 7: `readonly addOn: typeof bootstrapEncryptionAddOn;`      |
| **JSDoc retained** where TS 5 dropped it                   | `activerecord/ar-config.d.ts`                                                                                                             | TS 7 emits `/** @internal */` into the `.d.ts`                                                               |
| Emit **failed**                                            | `trailties/application.d.ts`                                                                                                              | the TS4094 above                                                                                             |

Full differing set: `activemodel/validations/helper-methods.d.ts`,
`activemodel/validations.d.ts`, `activerecord/ar-config.d.ts`,
`activerecord/relation/query-methods.d.ts`,
`activerecord/relation/spawn-methods.d.ts`,
`activerecord/test-helpers/fixtures-registry.d.ts`,
`activesupport/{async-context,child-process,crypto,fs,os}-adapter.d.ts`,
`activesupport/index.d.ts`, `activesupport/xml-mini.d.ts`,
`trailties/package-manager.d.ts`, and the missing `trailties/application.d.ts`.

None of these is a consumer-visible type-compatibility break. The last class is
worth flagging for a different reason: TS 7 retaining `@internal` in `.d.ts`
interacts with the `@internal` machinery in
`eslint/rails-private-methods.json` and `parity:api:extra`, which reads the
source rather than the emit — so it is a note, not a hazard, but it should be
checked before any flip.

**"Declaration emit differs greatly, intentionally" is not what we measured.**
On this codebase it differs in 0.42% of files, cosmetically. RFC #59's plan to
build a whole sampled-`.d.ts`-diff gate around this risk was sized against a
fear, not a measurement; the real thing is a 15-file allowlist.

## Non-goals

- **Migrating `trails-tsc` to a TS 7 API.** Impossible today: no programmatic
  `--build`, no LS plugin hosting. This is the blocker, not a task.
- **Shipping a split TS 5.x + TS 7 environment.** Explicitly rejected by the
  maintainer on [PR #59][pr59]. This RFC does not re-propose it.
- **Building the dual-run diagnostic-parity gate from RFC #59.** The measured
  diagnostic delta is 2 errors in 1 file and the `.d.ts` delta is 14 files. A
  parity harness is disproportionate to a 15-row allowlist; if we migrate, the
  spike script plus a one-shot review is enough.
- **Flipping the editor Language Service.** Out of scope until 7.1 ships an API
  a plugin can attach to.
- **Adopting TS 7-only language features.** A compiler swap at behavioural
  parity, if it happens at all.

## Alternatives considered

- **Migrate now, keep pinned TS 5.x for `trails-tsc` only.** This is RFC #59's
  proposal, narrowed from two packages to one. Rejected: it is still a split
  env, which is the stated objection, and the measured benefit (zero PR
  wall-clock, ~8.9 runner-hours/week) does not buy off two compilers in one
  repo.
- **Rewrite `trails-tsc` to not need the compiler API** — drop the LS plugin,
  shell out to `tsc --build` instead of driving the solution builder.
  Rejected as premature: it discards working DX tooling to chase a benefit
  we have now measured as modest, and 7.1 may make the rewrite unnecessary or
  much smaller. Worth reconsidering only if 7.1 ships without the two gaps
  closed.
- **Adopt TS 7 for CI only, keep TS 5.x locally.** Rejected: drift between what
  CI checks and what developers check, in exchange for the one axis (runner
  minutes) with the smallest payoff.
- **Do nothing and do not write this down.** Rejected: that is what produced a
  13-month-stale RFC whose central claim had to be re-derived from scratch. The
  decision record _is_ the deliverable.

## Rollout

Only one story is `ready`. The rest are `blocked` on a dated external event and
exist so the re-evaluation is cheap rather than a from-scratch redo.

1. **Now (unblocked, worth doing regardless of TS 7).**
   - `fix-anonymous-class-declaration-emit` — fix the two TS4094 sites in
     `trailties/src/application.ts`.

2. **At TS 7.1 beta, 2026-09-09 (the API surface freezes).**
   - `recheck-ts7-api-surface` — re-run this RFC's API-surface mapping against
     the 7.1 beta and record the verdict for `trails-tsc`.

3. **At TS 7.1 stable, 2026-11-10 — gated on the two gaps closing.**
   - `port-tsc-wrapper-to-ts7-api` — move `activerecord-cli`'s `tsc-wrapper`
     onto the TS 7 API.
   - `port-trails-tsc-to-ts7-api` — the blocker; only actionable if the
     solution-builder and LS-plugin gaps close.
   - `flip-build-to-ts7` — swap the pinned `typescript` and drop TS 5.x
     entirely. Single-compiler, no split env.

4. **Independent of the above.**
   - `measure-editor-ls-latency` — close the one unmeasured axis.

## Verification

This RFC is a decision record; its verification is that the decision is
re-checkable, not that code changed.

- **Unblocked work:** `pnpm build` under `typescript@7.0.2` produces **zero**
  diagnostics across all 18 projects (today: 2). Verified by re-running the
  spike after `fix-anonymous-class-declaration-emit`.
- **Re-evaluation trigger:** `recheck-ts7-api-surface` closes with an explicit
  yes/no on each of the two gaps, dated and sourced, by 2026-09-16.
- **If we ever migrate:** exactly one `typescript` version resolves in the
  workspace (`pnpm why typescript` shows a single 7.x), and no
  `Build & Type Check` regression in the CI medians recorded above.
- **Cost baseline for the next re-evaluation:** the numbers in Motivation are
  reproducible by the recorded methods, on a host whose load average is stated
  alongside the result.

## Open questions

1. **Does TS 7.1 ship a programmatic `--build`?** Not on the iteration plan.
   _Recommendation:_ ask upstream (`microsoft/TypeScript` discussion) before
   7.1 beta prep on 2026-09-04, so the answer is known while it can still
   influence 7.1 rather than 7.2.
2. **Does TS 7.1's Language Service API admit plugins?** The plan lists
   "Language Service API" with no plugin item, and the nightly's
   `LanguageService` is a five-method server client. _Recommendation:_ same —
   ask upstream, and treat a "no" as the trigger to seriously cost the
   `trails-tsc` rewrite alternative.
3. **What does TS 7 cost/save in the editor?** Unmeasured, and the likeliest
   real win. _Recommendation:_ `measure-editor-ls-latency`, independent of
   everything else here.
4. **Does TS 7 retaining `@internal` in `.d.ts` affect our tooling?**
   `parity:api:extra` and `blazetrails/unbacked-internal-needs-receipt` read
   source, not emit, so probably not. _Recommendation:_ confirm during
   `flip-build-to-ts7`; not a blocker.
5. **Are the CI runners hosted or self-hosted?** The $220/year figure assumes
   GitHub-hosted list pricing. _Recommendation:_ confirm before quoting the
   dollar figure anywhere; the runner-hours number stands either way.

## Sources

- **tasks PR #59** — "docs(rfc): migrate to TypeScript 7 (native Go compiler)",
  opened 2026-07-08, closed unmerged 2026-07-22:
  <https://github.com/blazetrailsdev/tasks/pull/59>
- **Announcing TypeScript 7.0** — devblogs.microsoft.com, published 2026-07-08,
  fetched 2026-08-25:
  <https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/>
- **TypeScript 7.1 Iteration Plan** — microsoft/TypeScript#63703, fetched
  2026-08-25: <https://github.com/microsoft/TypeScript/issues/63703>
- **microsoft/typescript-go** — repo README, fetched 2026-08-25 (repo now
  closed; development moved to `microsoft/TypeScript`):
  <https://github.com/microsoft/typescript-go>
- **npm registry `typescript` metadata** — `npm view typescript version
dist-tags time`, queried 2026-08-25.
- **Shipped `.d.ts` of `typescript@7.0.2` and `typescript@7.1.0-dev.20260825.1`**
  — installed outside the tree and read 2026-08-25.
- **CI job durations** — `gh api` over `blazetrailsdev/trails` workflow
  242258170, 500 `pull_request` runs spanning 2026-08-10 → 2026-08-24, jobs
  resolved for the 170 most recent; sampled 2026-08-25.

[pr59]: https://github.com/blazetrailsdev/tasks/pull/59
[ga]: https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/
[iter]: https://github.com/microsoft/TypeScript/issues/63703

## Changelog

- 2026-08-25: initial RFC. Supersedes the closed
  `0000-typescript-7-native-compiler` (tasks PR #59). Re-verified every dated
  claim; corrected the "CI long pole" and "~60s cold pre-commit" premises with
  measurements; narrowed the split-env blocker from two packages to one
  (`trails-tsc`); recorded the TS 7.0.2 `unstable/` API, which did not exist as
  a known fact when #59 was written.
