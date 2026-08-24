---
title: "website-vitest-missing-activesupport-subpath-alias"
status: claimed
updated: 2026-08-24
rfc: "0061-ci-failures"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T14:19:54Z"
assignee: "website-vitest-missing-activesupport-subpath-alias"
blocked-by: null
closed-reason: null
---

## Context

`packages/website/src/lib/frontiers/sandbox-sw.test.ts`'s
`"trail-cli accepts generate model command"` case fails on `origin/main`
(verified at 20cf3f521 in a detached worktree sharing this checkout's
`node_modules`):

```text
Error: Test timed out in 5000ms.
 ❯ src/lib/frontiers/sandbox-sw.test.ts:180:5
```

The CLI never answers, because the generator import chain
(`VfsModelGenerator` -> `ModelGenerator` -> `generated-attribute`) reaches
`packages/trailties/src/generators/generated-attribute.ts:2`:

```ts
import { Temporal } from "@blazetrails/activesupport/temporal";
```

`packages/website/vitest.config.ts:9` aliases only the package root
(`@blazetrails/activesupport` -> `../activesupport/src/index.ts`); the
`/temporal` subpath has no alias. With an incomplete workspace link set the
failure surfaces as the underlying
`Cannot find module '@blazetrails/activesupport/temporal'` instead of the
timeout, which is the same root cause seen more directly.
`packages/website/vite.config.ts:33,55` keeps its own `pkgAlias` list plus a
`startsWith("@blazetrails/activesupport/")` external rule; the two configs
disagree.

`runtime.test.ts` fails to collect in the same way when the workspace links
are incomplete, so treat it as part of the same fix.

Reproduce (a fresh worktree needs `npx svelte-kit sync` first, or every
website test fails earlier with a TSConfckParseError on
`./.svelte-kit/tsconfig.json`):

```bash
cd packages/website && npx svelte-kit sync
npx vitest run src/lib/frontiers/sandbox-sw.test.ts src/lib/frontiers/runtime.test.ts
```

Surfaced while verifying an unrelated rename in PR #6982, whose diff touches
neither `generated-attribute.ts`, `activesupport`, nor either website config.
See the memory note `project_new_package_subpath_needs_four_registrations.md`:
a new cross-package subpath needs the vitest alias plus both dx-test
tsconfigs, and the website project is a further registration site that was
missed.

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
