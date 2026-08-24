---
title: "website-vitest-missing-activesupport-subpath-alias"
status: ready
updated: 2026-08-24
rfc: "0061-ci-failures"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two frontiers tests in `packages/website` fail at collection time with:

```text
Error: Cannot find module '@blazetrails/activesupport/temporal' imported from
  packages/trailties/src/generators/generated-attribute.ts
```

- `packages/website/src/lib/frontiers/runtime.test.ts` (whole suite fails to collect)
- `packages/website/src/lib/frontiers/sandbox-sw.test.ts` — the
  `"trail-cli accepts generate model command"` case

`packages/website/vitest.config.ts:9` aliases only the package root
(`@blazetrails/activesupport` -> `../activesupport/src/index.ts`); the
`/temporal` subpath has no alias, so the import from
`packages/trailties/src/generators/generated-attribute.ts:2` — pulled in
through `VfsModelGenerator` -> `ModelGenerator` -> `generated-attribute` —
never resolves. `packages/website/vite.config.ts:33,55` has its own
`pkgAlias` list plus a `startsWith("@blazetrails/activesupport/")` external
rule; the two configs disagree.

Reproduce (a fresh worktree needs `npx svelte-kit sync` first, or every
website test fails earlier with a TSConfckParseError on
`./.svelte-kit/tsconfig.json`):

```bash
cd packages/website && npx svelte-kit sync
npx vitest run src/lib/frontiers/runtime.test.ts src/lib/frontiers/sandbox-sw.test.ts
```

Pre-existing on `origin/main` — surfaced while verifying an unrelated rename in
PR #6982, whose diff touches neither `generated-attribute.ts`, `activesupport`,
nor either website config. See the memory note
`project_new_package_subpath_needs_four_registrations.md`: a new cross-package
subpath needs the vitest alias plus both dx-test tsconfigs, and the website
project is a fourth registration site that was missed.

## Acceptance criteria

- [ ] `packages/website/vitest.config.ts` resolves
      `@blazetrails/activesupport/temporal` (and any other subpath the
      frontiers import graph reaches), so `runtime.test.ts` and
      `sandbox-sw.test.ts` collect and pass.
- [ ] The alias list is derived from, or kept consistent with, the one in
      `packages/website/vite.config.ts` rather than duplicated by hand.
- [ ] Document whether `npx svelte-kit sync` is a prerequisite for running the
      website tests in a fresh worktree, and if so wire it into the test
      script (or `scripts/start-worktree.sh`) so the TSConfckParseError does
      not look like a real failure.
