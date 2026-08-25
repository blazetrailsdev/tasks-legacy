---
title: "Declare TypeScript 7 as the supported floor in published peer ranges"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["trails-tsc", "trailties", "activerecord"]
deps: []
deps-rfc: []
est-loc: 30
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

The headline change of RFC `0000-typescript-7-ground-floor`. Three of trails'
17 published packages declare a `typescript` peer dependency, and all three are
wrong for a TypeScript 7 user (verified 2026-08-25):

| package                     | declared  | problem                                                                                                      |
| --------------------------- | --------- | ------------------------------------------------------------------------------------------------------------ |
| `@blazetrails/trails-tsc`   | `^5.0.0`  | excludes TS 7 — peer conflict on install                                                                     |
| `@blazetrails/trailties`    | `^5.0.0`  | excludes TS 7, and nothing in it actually needs 5.x                                                          |
| `@blazetrails/activerecord` | `>=5.0.0` | admits TS 7 and then fails — `./type-virtualization/*` needs `ts.createSourceFile`-from-text, absent in TS 7 |

`activerecord`'s is the most harmful: the range invites a TS 7 user in and the
subpath cannot work for them. This is a defect in the published contract right
now, independent of whether trails ever migrates its own build.

This story fixes the **declaration**. The ports that make the declaration true
are separate stories (`port-tsc-wrapper-to-ts7-api`,
`port-type-virtualization-to-ts7-api`, `port-trailties-parsets-to-ts7-api`) —
sequence them ahead of publishing if they are landing anyway, but do not let
them block correcting a range that is wrong today.

## Acceptance criteria

- [ ] No published package declares a `typescript` peer range that admits a
      version it cannot actually work on.
- [ ] `@blazetrails/trailties` and `@blazetrails/activerecord` accept TS 7 —
      either `>=7.0.0` or a range spanning 5.x and 7.x, matching what the code
      supports **after** its port lands.
- [ ] `@blazetrails/trails-tsc` states its real constraint. It keeps a 5.x
      requirement (no programmatic `--build`, no LS plugin hosting in TS 7), and
      that is documented at the declaration rather than left implicit.
- [ ] The supported TypeScript version is stated in `README.md` where a user
      will find it before installing.
- [ ] If a port has not landed for a given package, its range reflects
      **current** reality, not the intended end state.

## Definition of done

Widening `activerecord` to `>=5.0.0 || >=7.0.0` while
`./type-virtualization/*` still cannot run on TS 7 does not close this story —
that reproduces the exact defect being fixed.

## Verification

```bash
pnpm -r exec node -p "JSON.stringify(require('./package.json').peerDependencies||{})"
# expect: no bare ^5.0.0 outside @blazetrails/trails-tsc
```

Then, in a scratch project with only `typescript@7.x` installed, confirm
`npm install @blazetrails/activerecord @blazetrails/trailties` resolves without
a peer conflict.
