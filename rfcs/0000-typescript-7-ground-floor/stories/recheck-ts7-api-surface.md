---
title: "Move the pin to 7.1 stable and re-check the trails-tsc gaps"
status: blocked
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["trails-tsc", "activerecord-cli"]
deps: []
deps-rfc: []
est-loc: 0
priority: null
pr: null
claim: null
assignee: null
blocked-by: "TypeScript 7.1 beta — scheduled 2026-09-09 (microsoft/TypeScript#63703). Not yet released as of 2026-08-25; latest is 7.0.2 and 7.1 exists only as nightlies."
---

## Context

**Two jobs now that the RFC targets the 7.1 line:** move the pinned compiler off
the `7.1.0-dev` nightly onto 7.1 stable when it ships (2026-11-10), and re-check
whether the two `trails-tsc` gaps closed. Beta is 2026-09-09 — the point at
which the API surface stops moving and the re-check becomes meaningful.

Re-confirmed 2026-08-25 against `7.1.0-dev.20260825.1`: there is still **no
solution-builder, build, or watch API of any kind**, verified by both a content
grep of `dist/` and the exported-name index. So the expected answer is "still
blocked" — this story exists so that is a lookup rather than a re-derivation.

## Original context

RFC `0000-typescript-7-ground-floor` recommends waiting on TS 7.1 and names
exactly two gaps that keep `trails-tsc` on TypeScript 5.x — and therefore keep
any TS 7 adoption a "split env", which the maintainer rejected on
[tasks PR #59](https://github.com/blazetrailsdev/tasks/pull/59) (2026-07-22):

1. **Programmatic `--build`** — `src/build.ts` drives `createSolutionBuilder`,
   `createSolutionBuilderHost`, `createEmitAndSemanticDiagnosticsBuilderProgram`.
2. **LS plugin hosting** — `src/lsp-plugin.ts` (the `./ts-plugin` export)
   implements `LanguageServiceHost` / `ScriptSnapshot` to decorate a
   `LanguageService`.

Neither exists in `typescript@7.0.2` nor in `typescript@7.1.0-dev.20260825.1`,
and neither is a line item in the [7.1 iteration
plan](https://github.com/microsoft/TypeScript/issues/63703), which lists only
Content Mapper API, Emit API, and Language Service API.

7.1 beta is scheduled for **2026-09-09**, which is when the API surface stops
moving. This story re-runs the RFC's mapping against it so the decision is a
lookup rather than a re-derivation.

## Acceptance criteria

- [ ] The RFC's API-surface mapping table is re-run against the 7.1 beta and
      updated in place, with the version string and verification date recorded.
- [ ] An explicit **yes/no** is recorded for each of the two gaps, with a
      source link (release notes, `.d.ts`, or an upstream issue reply).
- [ ] The pinned `typescript` moves from the `7.1.0-dev` nightly to 7.1 stable,
      with `pnpm build` / `pnpm typecheck` green and no new diagnostics.
- [ ] The RFC records whether the two `trails-tsc` gaps closed; if they did,
      `port-trails-tsc-to-ts7-api` moves to `ready` and the aliased 5.x is
      scheduled for removal.
- [ ] If either gap is a "no", the `trails-tsc` rewrite alternative (RFC
      § Alternatives considered) is costed rather than left as a sentence.

## Definition of done

Re-reading the iteration plan does not close this story. The mapping must be
re-run against the **installed 7.1 beta package's shipped `.d.ts`**, the way
the original was.

## Verification

```bash
npm view typescript dist-tags                 # confirm a 7.1 beta exists
npm i typescript@beta --prefix /tmp/ts71      # outside the tree
# then re-run the RFC's method: extract exported names from
# dist/api/**/*.d.ts + dist/ast/**/*.d.ts and diff against the
# ts.* symbol set grepped from packages/trails-tsc/src.
```

## Notes

Open questions 1 and 2 in the RFC recommend asking upstream **before 7.1 beta
prep on 2026-09-04**, while the answer can still influence 7.1. If that
happened, link the thread here.
