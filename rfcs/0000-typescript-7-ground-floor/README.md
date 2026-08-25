---
rfc: "0000-typescript-7-ground-floor"
title: "Make TypeScript 7 the ground-floor version for trails"
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

<!-- Unnumbered until merge: dir stays `0000-typescript-7-ground-floor`,
     `rfc:` stays `0000-...`, H1 stays number-free.
     `scripts/finalize-rfc.mjs` assigns the number at merge. -->

# RFC — Make TypeScript 7 the ground-floor version for trails

## Recommendation

**Proceed. Make TypeScript 7 the ground-floor version for trails, targeting
the 7.1 line, now.**

trails is pre-release with no external dependents, so this is not a
compatibility migration with a cost to weigh — it is a choice of baseline while
that choice is still free. The RFC therefore recommends the **newest** target
rather than the safest one: pin an exact `7.1.0-dev` nightly now and move to
7.1 stable when it ships (2026-11-10). Writing against 7.1's API today means
the code we write is the code we keep.

Measured on this repo, cold full-monorepo `tsc --build` on a quiet host:

| compiler                          | wall      | vs 5.9.3  | diagnostics                |
| --------------------------------- | --------- | --------- | -------------------------- |
| `typescript@5.9.3` (today)        | 91.75s    | —         | 0                          |
| `typescript@7.0.2`                | **9.73s** | **9.4×**  | 2                          |
| `typescript@7.1.0-dev.20260825.1` | **8.47s** | **10.8×** | 10, from **2 root causes** |

### Why 7.1 rather than 7.0.2

7.1 is strictly more API for the same build, and two of its additions land
directly on our code:

- **`transpileModule` returns.** `trailties`' `parseTs()` needs no rewrite at
  all on 7.1 — an import-path change, not a port. On 7.0.2 it needs
  reimplementing on `getSyntacticDiagnostics`.
- **`Program.emit`, `EmitResult`, `parseCommandLine`, `readConfigFile`,
  `parseJsonConfigFileContent`, `LanguageService`** — 115 new exported names
  over 7.0.2. `activerecord-cli`'s `tsc-wrapper` has emit and config-parsing
  needs that 7.0.2 cannot serve.

Targeting 7.0.2 means writing shims for things 7.1 gives us in eleven weeks.
Given no external dependents, that trade is not worth making.

### What 7.1 still does not give us

`trails-tsc`'s two gaps are **unchanged on the newest code** — verified against
`7.1.0-dev.20260825.1` by both a content grep of `dist/` and the exported-name
index: there is **no solution-builder, build, or watch API at all**. So
`@blazetrails/trails-tsc` keeps a pinned 5.x for its `trails-tsc-views` binary.
That package serves the TSE views pipeline, which is roadmap-stage (ActionView
8.2% of API surface, P3) — it is not on the ground-floor path and does not
block this RFC.

### The two root causes in the 7.1 build

Ten diagnostics, two defects, both ours and both narrow:

1. **`activesupport/src/yaml.ts:23` — `TS2883`.** The inferred type of
   `{ parse, stringify }` cannot be named portably without referencing a
   `node_modules/yaml/dist/parse/cst` path. This is a _stricter portability
   check new in the 7.1 line_, and its consequence is that
   **`activesupport/dist/yaml.d.ts` never emits** (verified: only `yaml.js` and
   `yaml.js.map` are written). The other **7** diagnostics — 5× `TS7016`
   "could not find a declaration file for `@blazetrails/activesupport/yaml`"
   across `actionview`, `activemodel`, `activerecord`, plus 2× `TS7006` implicit
   `any` flowing from that untyped import — are all downstream of it. One type
   annotation clears eight diagnostics.
2. **`trailties/src/application.ts:41,45` — `TS4094` ×2.** The anonymous
   `executor` / `reloader` class expressions over `#private`-bearing bases.
   Present on 7.0.2 as well.

Neither is a TypeScript regression. `TS2883` is a real portability defect that
5.9.3 was silently tolerating while emitting a declaration file — arguably worth
fixing on its own merits.

### The published contract

Three published packages declare `typescript` peers, and all three are wrong for
a TS 7 user: `@blazetrails/trails-tsc` and `@blazetrails/trailties` at `^5.0.0`
(which excludes TS 7 outright), and `@blazetrails/activerecord` at `>=5.0.0`
(which admits TS 7 and then fails, because `./type-virtualization/*` needs
`ts.createSourceFile`-from-text). With no external dependents this harms nobody
today — it is a correctness item, not an urgency — but it is the literal
declaration of the floor, so it is the first story.

### A correction worth surfacing

`packages/trails-tsc/src/cli.ts:6-10` says `activerecord` publishes the
user-facing `trails-tsc` bin. It does not — **`activerecord-cli` does**
(`bin/trails-tsc.js`). The example app's `typecheck` script resolves to
`activerecord-cli`'s tsc-wrapper, which is the migratable one. The blocked
package (`@blazetrails/trails-tsc`) publishes only `trails-tsc-views`. This
distinction is load-bearing for the whole recommendation, and the stale comment
is what obscured it.

## Motivation — pick the baseline while it is still free

trails is pre-release software with no external dependents. That is the whole
motivation, and it cuts against the way both this RFC and [#59][pr59] were
originally framed: there is no compatibility cost to amortise, no users to
break, and no migration window to negotiate. There is only a question of which
TypeScript we build the next several years of this codebase on, asked at the one
moment when the answer costs nothing.

The measured facts say the newest answer is also the cheapest one. TS 7.1 builds
this repo **10.8× faster** than 5.9.3, its declaration output differs from
5.9.3's in **14 of 3,338 files, all benign**, and its entire diagnostic delta is
**two defects of our own** — one of which (`TS2883`) is a real portability bug
5.9.3 was hiding while emitting a `.d.ts` that should not have emitted.

The secondary case — what staying on 5.x costs us in CI and local build time —
follows below. It is included because #59 asserted it without measuring and a
future reader deserves the real numbers. It is **not** the argument for this
RFC, and on its own it would not carry one.

### What we actually pay for not upgrading### What we actually pay for not upgrading

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
-delete && rm -rf packages/*/dist scripts/dist` before each cold run, then
`/usr/bin/time -f '%e %M'`. TS 5 is the repo's pinned `typescript@5.9.3`; the
TS 7 builds use `typescript@7.0.2` and `typescript@7.1.0-dev.20260825.1`
installed in throwaway directories outside the tree and invoked by absolute
path.

**All numbers below were taken on a quiet host (load average 3–5 on 24 cores).**
An earlier revision of this RFC measured the same things at load 41–59 with the
agent fleet live and reported cold full-monorepo `tsc --build` as 215s; the true
uncontended figure is **91.75s**. Contention inflated the absolute by ~2.3×
while leaving the compiler ratio broadly intact (10.3× contended vs 9.4× clean),
which is why the ratio was flagged as the trustworthy figure at the time. Quote
the clean numbers.

| Scenario                                                                        | 5.9.3                    | 7.0.2               | 7.1.0-dev           |
| ------------------------------------------------------------------------------- | ------------------------ | ------------------- | ------------------- |
| Cold `tsc --build`, full monorepo (18 projects)                                 | **91.75s** (3.07 GB RSS) | **9.73s** (2.90 GB) | **8.47s** (2.82 GB) |
| Cold `tsc -b packages/activerecord` (+4 refs)                                   | **69.92s**               | **8.34s**           | not measured        |
| Warm no-op `node scripts/typecheck.mjs`                                         | **0.47s**                | not measured        | not measured        |
| Incremental after one real edit to `activerecord/src/relation/query-methods.ts` | **6.36s**                | not measured        | not measured        |

**Correction to #59.** `scripts/typecheck.mjs:16` calls the cold path "~60s".
The real cold full-monorepo build is **91.75s** — the comment understates it by
~1.5× — but that path is paid **once per worktree**, not per commit. What a
developer or agent pays on every commit is the incremental path: **6.36s** after
touching an ActiveRecord file, **0.47s** when nothing changed. Neither the
comment's number nor #59's framing of it as "a real tax" survives measurement.

`scripts/typecheck.mjs` is a 29-line wrapper around `spawnSync(tsc,
["--build"])`, so its cost is exactly `tsc --build`'s.

### Editor cost — **unmeasured, and deliberately postponed**

`packages/activerecord/src` is 165,308 non-test LOC (`find … -name '*.ts' -not
-name '*.test.ts' | xargs cat | wc -l`, 2026-08-25), dwarfing the next package
(`activesupport`, 40,356). Open-to-ready and "loading…" latency in an editor
were **not measured** — this work ran headless and any number would be a guess.
No `tsconfig` in this repo declares a `plugins` array and there is no
`.vscode/settings.json`, so the TSE language-service plugin is not wired into
any editor today — there is no integration to regress or improve yet. The
measurement is parked with the views pipeline (`measure-editor-ls-latency`,
status `draft`) and is not part of the ground-floor case.

### The agent-fleet multiplier

Method: `ls -d /home/dean/github/blazetrailsdev/worktrees/*/` and `stat -c %Y`
on each worktree's `.git` file, 2026-08-25.

- **103 live worktrees**, oldest 2026-06-19, newest 2026-08-25 (67-day span).
- **57 of 103** have a populated `packages/activerecord/dist` — i.e. have paid
  at least one cold build.
- Creation rate: **2.1 worktrees/day** over the last 14 days (1.5/day over the
  full span).

At 91.75s per cold build, ~2.1 new worktrees/day is **≈3.2 min/day ≈ 0.4
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

### Summary: the secondary case

| Axis              | Cost of staying on TS 5.9.3                                 |
| ----------------- | ----------------------------------------------------------- |
| PR wall-clock     | **zero** — typecheck is off the critical path               |
| CI runner time    | ≈ 8.9 runner-hours/week recoverable (5.1% of total)         |
| Local cold build  | 91.75s → 8.47s per worktree bootstrap, ~2.1/day             |
| Local incremental | 6.36s per commit touching AR; no TS 7 measurement yet       |
| Host contention   | real but small: ~0.9 CPU-hours/week, concentrated in bursts |
| Editor latency    | **unmeasured** — the likeliest real win, and the open gap   |

This is a genuine but modest cost, and on its own it would not justify much of
anything. It is not why this RFC recommends proceeding — the published contract
being wrong is (§ Motivation). Read this section as the answer to "what do we
also get", not "why do this".

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

### API-surface mapping for our compiler-API consumers

**Correction to the framing inherited from #59:** this repo has **four**
compiler-API consumers, not two. `grep -rn 'from "typescript"'` across
`packages/` (2026-08-25) finds **34 import sites**:

| package                                   | sites | uses                                                           | status                  |
| ----------------------------------------- | ----- | -------------------------------------------------------------- | ----------------------- |
| `trails-tsc`                              | 16    | solution builder, LS plugin                                    | **blocked**             |
| `activerecord` `src/type-virtualization/` | 8     | `createSourceFile`, `createScanner`, `getLeadingCommentRanges` | **migratable on 7.0.2** |
| `activerecord-cli` `tsc-wrapper`          | 8     | `createSourceFile` + diagnostics                               | **migratable on 7.0.2** |
| `trailties` `template-builder/testing.ts` | 2     | `transpileModule`                                              | migratable on 7.1       |

`activerecord`'s is published surface — `./type-virtualization/*.js` is an
`exports` subpath backed by the `typescript: ">=5.0.0"` peer dependency.

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

#### Verdict: `activerecord-cli`'s `tsc-wrapper` — **migratable today on 7.0.2**

Its work is parse-and-diagnose. `schema-ts-parser.ts` and
`schema-ts-model-parser.ts` are pure AST walks (guards + `forEachChild` +
`SyntaxKind`) — **fully covered by 7.0.2 already**, modulo `createSourceFile`'s
missing text-parse, which a virtual-FS `Project` covers.
`auto-import.ts` is the same shape. `cli.ts`'s `getPreEmitDiagnostics` +
formatting helpers are compose-or-reimplement. No solution builder, no LS
plugin. **This package is not the blocker, and it does not need to wait for
7.1** — the virtual-FS measurements below close its only real gap.

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

### What the virtual FS closes (measured 2026-08-25, TS 7.0.2)

`typescript/unstable/fs`'s `createVirtualFileSystem` / `FileSystem` delegation
turned out to close the largest gap in the table above. This was verified by
building working probes against `typescript@7.0.2`, not by reading `.d.ts`.

**Text parsing works.** A virtual FS holding a tsconfig + one source file, opened
via `new API({ fs })` → `updateSnapshot({ openProjects })` → `Program.getSourceFile()`,
returns a real AST. The `unstable/ast` guards (`isCallExpression`, `isIdentifier`,
`isPropertyAccessExpression`) and `node.forEachChild` all work on it, and a walk
over a schema-shaped file extracted exactly the call names
`schema-ts-parser.ts` extracts. On
`packages/activerecord/src/test-helpers/test-schema.ts` (2,034 lines) both
compilers produce **identical node counts (5,686)**.

| same file, same dense walk | TS 5.9.3                    | TS 7.0.2 via virtual FS     |
| -------------------------- | --------------------------- | --------------------------- |
| parse                      | 71.0ms (`createSourceFile`) | **5.5ms** (`getSourceFile`) |
| dense walk, 5,686 nodes    | 1.4ms                       | 3.5ms                       |
| total, warm process        | 72.4ms                      | **9.0ms**                   |
| one-time `API` spawn       | —                           | ~99ms                       |

The AST is shipped over the wire **once** and materialized locally — the walk is
not per-node RPC. The walk is 2.5× slower (remote node materialization) but the
parse win dominates from the second file onward. For a one-shot CLI parsing a
single file, TS 7 is roughly a wash (108ms vs 72ms).

**The real-FS + in-memory overlay pattern works** — the `tsc-wrapper` case. An
in-memory tsconfig and source, type-checked against the real on-disk
`activerecord` project, correctly reported
`TS2322: Type 'number' is not assignable to type 'string'`, with 2 overlay hits
and 64 real-FS fallthroughs (`readFile` returning `undefined` falls through to
disk, per the documented contract).

**Diagnostics arrive pre-flattened** as
`{ fileName, pos, end, code, category, text }`. This removes the need for
`flattenDiagnosticMessageText` entirely, and `computeLineStarts`
(`unstable/ast/scanner`) converts `pos` to line/column — so `formatDiagnostics`
is a short reimplementation rather than a gap.

#### What it does **not** close: `--build`

`trails-tsc/src/build.ts` is not merely a `--build` driver — it is a
**virtualizing** build. `buildCompilerHost` (`src/host.ts`) intercepts `readFile`
so plugins rewrite source text before the compiler sees it, and
`remapDiagnostics` maps diagnostics from the virtualized text back to the
user's original coordinates via cached line deltas. Two consequences:

- **Shelling out to `tsc --build` cannot replace it.** The CLI compiles what is
  on disk and exposes no filesystem hook. This was the cheapest hoped-for escape
  and it is structurally unavailable.
- **The API gets halfway.** `unstable/fs`'s `readFile` hook is a direct analogue
  of `buildCompilerHost` — opening all 18 project configs fired **16,233**
  `readFile` callbacks into JS, with a targeted file intercepted 4×. So the
  virtualization mechanism ports cleanly.

But the API hands back _projects_, not a _build_. Opening the root
`tsconfig.json` yields **1** project with 0 root files — the reference graph does
not expand. Opening all 18 configs explicitly works (18 projects, 6.2s), but
without `--build` semantics: `activerecord`'s full semantic check then reports
**712 diagnostics against `tsc --build`'s 2**, because module resolution follows
`node_modules/@blazetrails/arel/src/*.ts` instead of redirecting to
`../arel/dist/*.d.ts` the way `references` does. There is also no up-to-date
checking, no build ordering, and no emit at all in 7.0.2 (`Program.emit` arrives
in 7.1). Rebuilding those **is** reimplementing the solution builder.

`trails-tsc` therefore stays blocked on both counts, and this is now a measured
conclusion rather than an inference from the missing exports.

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

- `tsc -b packages/activerecord`: **8.34s, zero diagnostics**, 1,626 `.d.ts`
  emitted (re-measured on a quiet host; 19.9s under load).
- `tsc --build` (full monorepo, 18 projects): **9.73s, exactly two
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

- **Migrating `@blazetrails/trails-tsc` to a TS 7 API.** Impossible today: no
  programmatic `--build`, no LS plugin hosting (both measured, § "What it does
  not close"). It keeps a pinned 5.x. This is **not** on the ground-floor path —
  it publishes only `trails-tsc-views`, serving the roadmap-stage TSE views
  pipeline.
- **Re-proposing #59's split.** That split was permanent, load-bearing, and sat
  under the shipped DX. This RFC puts the entire shipped DX on TS 7 and leaves a
  5.x dependency only in a package whose blocked feature is not yet a product.
- **Measuring or optimising editor / language-service latency.** Deliberately
  postponed: the TSE language-service plugin is not wired into any `tsconfig` or
  editor config in this repo, so there is no editor integration to regress or
  improve yet. Revisit when the views pipeline ships.
- **Building the dual-run diagnostic-parity gate from RFC #59.** The measured
  diagnostic delta is 2 errors in 1 file and the `.d.ts` delta is 14 files. A
  parity harness is disproportionate to a 15-row allowlist; if we migrate, the
  spike script plus a one-shot review is enough.
- **Replacing `trails-tsc`'s solution builder by shelling out to `tsc --build`.**
  Measured as structurally impossible: the build virtualizes source text through
  plugin `readFile` interception, and the `tsc` CLI has no filesystem hook.
- **Flipping the editor Language Service.** Out of scope until 7.1 ships an API
  a plugin can attach to.
- **Adopting TS 7-only language features.** A compiler swap at behavioural
  parity, if it happens at all.

## Alternatives considered

- **Wait for TS 7.1 (2026-11-10) and migrate everything at once.** This was
  this RFC's own recommendation until the user-facing surface was examined.
  Rejected: it leaves three published peer ranges wrong for eleven more weeks,
  and nothing on the ground-floor path actually needs 7.1 — `parseTs()`, the
  last apparent 7.1 dependency, was reimplemented on 7.0.2 and verified.
- **#59's proposal: flip the build, pin 5.x for the API consumers.** Rejected
  for the reason it was rejected the first time — that split was permanent,
  load-bearing, and sat under the shipped DX. It also inverts the priority: it
  optimises our build while leaving the published contract wrong.
- **Fix only the peer ranges, port nothing.** Narrow `activerecord` to `^5.0.0`
  and admit trails is a TS 5 framework. Honest and nearly free, and it remains
  the fallback if the reviewer rejects the residual 5.x in `trails-tsc`. Not
  preferred: it makes a temporary tooling gap into a stated product limit, when
  three of the four consumers can move today.
- **Rewrite `trails-tsc` to not need the compiler API** — drop the LS plugin,
  shell out to `tsc --build`. Rejected, and now on measured grounds rather than
  as premature: the shell-out is structurally impossible (§ "What it does not
  close"), and the package is off the ground-floor path anyway, so the rewrite
  buys nothing a pinned 5.x does not.
- **Adopt TS 7 for CI only, keep TS 5.x locally.** Rejected: drift between what
  CI checks and what developers check, for the axis with the smallest payoff.

## Rollout

Target is the **7.1 line**: pin an exact `7.1.0-dev` nightly now, move to 7.1
stable on 2026-11-10. Every story branches from `main` and stands alone.

1. **Clear the two defects the 7.1 build finds.** Both are ours, both are
   narrow, and both are worth fixing regardless of TypeScript version.
   - `fix-yaml-inferred-type-portability` — the `TS2883` at
     `activesupport/src/yaml.ts:23` that silently suppresses `yaml.d.ts` emit
     and cascades into 7 downstream diagnostics.
   - `fix-anonymous-class-declaration-emit` — the `TS4094` pair in
     `trailties/src/application.ts`.

2. **Declare the floor.**
   - `declare-typescript-7-peer-ranges` — the three `typescript` peer ranges,
     plus the supported version in the README.

3. **Move the compiler-API consumers onto the 7.1 API.**
   - `port-tsc-wrapper-to-ts7-api` — the user-facing `trails-tsc` typecheck bin.
   - `port-type-virtualization-to-ts7-api` — `activerecord`'s published subpath.
   - `port-trailties-parsets-to-ts7-api` — on 7.1 this is an import-path change,
     not a port: `transpileModule` is back.

4. **Flip the build.**
   - `flip-build-to-ts7` — pin `typescript` at the 7.1 line;
     `@blazetrails/trails-tsc` keeps an aliased 5.x for `trails-tsc-views`.

5. **Deferred.**
   - `port-trails-tsc-to-ts7-api` — blocked upstream; no solution-builder or
     watch API exists in 7.1 either.
   - `recheck-ts7-api-surface` — at 7.1 stable (2026-11-10), confirm the pin
     moves cleanly off the nightly and re-check the two gaps.
   - `measure-editor-ls-latency` — postponed with the views pipeline.

## Verification

- **Clean build on the target compiler.** `tsc --build` under the pinned 7.1
  produces **zero** diagnostics across all 18 projects (today: 10, from 2 root
  causes), and `activesupport/dist/yaml.d.ts` exists.
- **The floor is stated correctly.** No published package declares a
  `typescript` peer range admitting a version it cannot work on:
  `pnpm -r exec node -p "require('./package.json').peerDependencies"` shows no
  bare `^5.0.0` outside `@blazetrails/trails-tsc`.
- **The shipped DX runs on TS 7.** In a scratch project with only the 7.1 line
  installed: `trails-tsc --noEmit -p tsconfig.json` succeeds against
  `examples/twitter-clone`, and `parseTs()` from
  `@blazetrails/trailties/template-builder/testing` returns diagnostics
  equivalent to 5.9.3's for valid, syntactically-invalid, and Ruby input.
- **The 5.x residue is contained.** `pnpm why typescript` resolves 5.x only
  under `@blazetrails/trails-tsc`; no batch `tsc --build` on 5.x remains in CI
  or hooks.
- **No unexplained type regression.** The `.d.ts` delta versus the last 5.9.3
  build stays within the 14 files enumerated in § spike, each reviewed — plus
  `yaml.d.ts`, which should newly _appear_.
- **Speed, re-measured the same way.** Cold full-monorepo `tsc --build` lands
  near the 8.47s measured here, with the host load average recorded alongside.

## Open questions

1. **Does one roadmap-stage package keeping a 5.x dependency count as the split
   that was rejected?** This is the reviewer's call and the RFC's main risk.
   _Recommendation:_ it does not — the rejected split sat under the shipped DX,
   this residue sits under an unshipped pipeline. If the reviewer disagrees, the
   fallback is Rollout phase 1 alone (fix the peer ranges), which is correct
   either way.
2. **Should `activerecord`'s `type-virtualization` consume the TS 7 API at all?**
   It is published surface, the API is out-of-process (~99ms spawn, where today's
   parse is in-process), and it is exported under `unstable/` with no semver
   guarantee. _Recommendation:_ decide inside
   `port-type-virtualization-to-ts7-api`; "keep 5.x internally and narrow the
   peer range" is a legitimate outcome that still fixes the floor.
3. **Is pinning a `7.1.0-dev` nightly acceptable in CI and across ~103
   worktrees?** An exact dev version is deterministic, so reproducibility is not
   the issue; being on unreleased software is. _Recommendation:_ yes, given
   pre-release status and an eleven-week runway to 7.1 stable — but pin an exact
   nightly, never `next`, and treat the 2026-11-10 move to stable as scheduled
   work (`recheck-ts7-api-surface`). If that is unpalatable, 7.0.2 is the
   fallback at the cost of shimming `transpileModule` and emit.
4. **Does TS 7 retaining `@internal` in `.d.ts` affect our tooling?**
   `parity:api:extra` and `blazetrails/unbacked-internal-needs-receipt` read
   source, not emit, so probably not. _Recommendation:_ confirm during
   `flip-build-to-ts7`; not a blocker.
5. **Are the CI runners hosted or self-hosted?** Only affects the dollar figure
   in § Motivation, which is secondary evidence. _Recommendation:_ confirm before
   quoting it; the runner-hours number stands either way.

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

- 2026-08-25: retargeted from 7.0.2 to the **7.1 line** — trails is
  pre-release with no external dependents, so the baseline choice is free and
  the newest target is also the cheapest (7.1 restores `transpileModule` and
  adds emit/config APIs, removing shims 7.0.2 would need). Re-measured every
  local number on a quiet host: cold full-monorepo `tsc --build` is **91.75s**
  on 5.9.3, not the 215s reported under load — 9.73s on 7.0.2, 8.47s on 7.1-dev.
  Found that the 7.1 build's 10 diagnostics reduce to **2 root causes**, the
  larger being a `TS2883` portability defect in `activesupport/src/yaml.ts` that
  silently suppresses `yaml.d.ts` emit and cascades into 7. Re-confirmed against
  the nightly that no solution-builder, build, or watch API exists in 7.1
  either, so `trails-tsc` stays deferred.
- 2026-08-25: reframed from "should we migrate the build" to "make TS 7 the
  ground floor". Renamed from `0000-typescript-7-reevaluation`. Recommendation
  flipped from **wait** to **proceed on 7.0.2**: the three published
  `typescript` peer ranges are wrong for TS 7 today, and every user-facing
  surface is either already TS 7-clean or migratable on 7.0.2 (`parseTs()`
  reimplemented and verified). Established that the `trails-tsc` blocker serves
  `trails-tsc-views` / the roadmap-stage TSE pipeline, not shipped DX — and that
  the user-facing `trails-tsc` bin belongs to `activerecord-cli`, not
  `@blazetrails/trails-tsc` as `cli.ts:6-10` claims. Editor/LS work postponed
  with the views pipeline.
- 2026-08-25: revised after probing `typescript/unstable/fs`. Corrected the
  consumer count (four packages, not one — `activerecord`'s published
  `type-virtualization` and `trailties` were missed); corrected the
  `createSourceFile` row from a hard gap to a verified rework, which moves two
  packages to "migratable today on 7.0.2"; added measured parse/walk numbers and
  the overlay result; recorded that shelling out to `tsc --build` cannot replace
  the virtualizing solution builder, so `trails-tsc` is blocked by measurement
  rather than by inference.
- 2026-08-25: initial RFC. Supersedes the closed
  `0000-typescript-7-native-compiler` (tasks PR #59). Re-verified every dated
  claim; corrected the "CI long pole" and "~60s cold pre-commit" premises with
  measurements; narrowed the split-env blocker from two packages to one
  (`trails-tsc`); recorded the TS 7.0.2 `unstable/` API, which did not exist as
  a known fact when #59 was written.
