---
title: "website-src-tests-run-in-no-ci-job"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 60
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Website `src/` unit tests run in no CI job, and one alias gap makes them uncollectable

## Context

Surfaced while landing #6819 (`retire-the-hyphen-migration-filename-alias`). The
website frontier fixtures at
`packages/website/src/lib/frontiers/tutorials/generator-fixtures.ts` advertised
`db/migrations/*-create-users.ts` while trailties' `MigrationGenerator` has
emitted `<timestamp>_<name>.ts` since PR 1.12c (#2176,
`packages/trailties/src/generators/migration-generator.ts:227`). The file's own
test (`generator-fixtures.test.ts`, "each fixture command produces its
expectedFiles") asserts the glob against real generator output and fails on the
hyphen form — it had simply never run. Two reasons, both still true after #6819:

1. **The `Website` CI job is disabled outright** — `.github/workflows/ci.yml:1320`
   opens its `if:` with a literal `false &&`, with a comment saying the build has
   been failing on main.
2. **Even when enabled it runs only `pnpm --filter @blazetrails/website exec
vitest run scripts/`** (`ci.yml:1344`), never `src/`. The root vitest config
   excludes `packages/website/**` from every project, so nothing anywhere runs
   `packages/website/src/**/*.test.ts`.

On top of that, the website vitest config cannot resolve the subpath
`@blazetrails/activesupport/temporal`: `packages/website/vitest.config.ts:9` maps
the bare `@blazetrails/activesupport` to `../activesupport/src/index.ts`, and
vite alias keys match as a PREFIX, so the subpath rewrites to
`.../activesupport/src/index.ts/temporal`. Any website test that reaches
`packages/trailties/src/generators/generated-attribute.ts:2` therefore fails at
collection with "Cannot find module '@blazetrails/activesupport/temporal'" —
`generator-fixtures.test.ts` among them. Adding the subpath alias ahead of the
bare one makes the file collect and its 18 tests pass (verified locally on
PR #6819's branch).

PR #6819 corrected the fixture strings themselves; it deliberately did not touch
the CI job or the alias, which are infrastructure outside its five stories.

## Acceptance criteria

- [ ] `packages/website/vitest.config.ts` resolves
      `@blazetrails/activesupport/temporal` (subpath entry ordered before the
      bare package entry), so website `src/` tests collect.
- [ ] `packages/website/src/**/*.test.ts` runs in some CI job, or the decision
      not to run it is recorded where the exclusion lives rather than being
      implicit in three separate configs.
- [ ] Re-enabling the `Website` job (dropping the `false &&`) is either done or
      superseded by a narrower job that runs the unit tests without the slow
      SvelteKit + typedoc + VitePress build.
