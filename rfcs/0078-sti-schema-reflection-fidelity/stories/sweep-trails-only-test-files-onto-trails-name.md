---
title: "sweep-trails-only-test-files-onto-trails-name"
status: done
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6932
claim: "2026-08-23T17:56:07Z"
assignee: "sweep-trails-only-test-files-onto-trails-name"
blocked-by: null
closed-reason: null
---

## Context

`sti-attribute-routing.test.ts` was renamed to `.trails.test.ts` in the
sweep-joins PR (RFC 0078 story
`sti-attribute-routing-test-is-trails-only-misnamed`), whose last acceptance
criterion asked for a sweep of sibling trails-only test files still sitting
under a plain `*.test.ts` name — to be listed, and filed separately if
numerous. They are numerous: **225** candidates in `packages/activerecord/src`
alone.

Method used (reproducible): extract the TS test manifest via
`pnpm parity:test --package activerecord`, then take every
`packages/activerecord/src/**/*.test.ts` in
`scripts/test-compare/output/ts-tests.json` that never appears as a matched TS
counterpart in the `parity:test` per-file report. Those files have no Rails
counterpart file at all, so every test in them lands in the
"extra (TS only)" bucket.

Why it matters: a plain `*.test.ts` name makes the test names look like they
should correspond to Rails tests, which the "NEVER rename or reword test names"
rule then protects. That cost a full check against `pnpm rails:find` in #4985
before a trails-deviation assertion could be corrected. A `.trails.` name makes
the provenance immediate and keeps `parity:test` from trying to match them.

The 225 are heavily clustered, which suggests splitting by directory rather
than one mega-PR:

- `connection-adapters/**` (~55) — adapter internals, oid types, schema
  statements/creation/definitions per adapter.
- `associations/**` (~40) — join-dependency internals, disable-joins matrix,
  preloader scopes.
- `support/**` (~30) — test-infra helpers (canonical schema, ddl profile,
  pg/sqlite templates, run tokens).
- `sqlite/**`, `type/**`, `type-virtualization/**`, `test-fixtures/**`,
  `trailties/**`, `coders/**` (~25 combined).
- `relation/**` (~15) and ~60 loose files at `src/` top level.

Note some entries are genuinely trails-only infrastructure (`support/**`,
`testing/**`, `sqlite/**` driver shims) while others may be _unported_ Rails
files that simply have no counterpart yet — those must NOT be renamed, because
the plain name is what a future port will match on. Classifying each file into
"trails invention" vs "not yet ported" is the substance of this work, not the
rename itself.

## Acceptance criteria

- [ ] Regenerate the candidate list with the method above (it drifts as files
      are ported).
- [ ] Classify each candidate as **trails invention** (rename to
      `*.trails.test.ts`) or **awaiting a Rails port** (leave the plain name,
      and note the Rails file it will match).
- [ ] Rename only the trails-invention set, in per-directory PRs sized under
      the LOC ceiling. Do NOT touch any test name — renames are file-level only.
- [ ] `pnpm parity:test` totals unchanged by each rename PR (verify by running
      the compare before and after; a pure rename of an unmatched file moves
      only the "extra (TS only)" count, never `matched`).
- [ ] Update any vitest include globs / CI filters that enumerate the renamed
      paths.
