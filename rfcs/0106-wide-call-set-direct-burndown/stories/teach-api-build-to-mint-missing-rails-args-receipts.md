---
title: 'parity:api:build cannot migrate kind: "args" rows into @missingRailsArgs receipts'
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 6941
claim: "2026-08-23T19:52:31Z"
assignee: "teach-api-build-to-mint-missing-rails-args-receipts"
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/build.ts` (`pnpm parity:api:build`) mints
`@missingRailsCall` receipts from `call-mismatches-exclude/` rows and drops the
migrated rows from the shard. It is call-SET only: a `kind: "args"` row is left
where it is.

PR #6938 (story `call-set-migrator-dry-run-reports-zero-for-args-kind-rows`)
made that honest rather than silent — the summary now reads
`0 of 2 baseline entr(ies) in scope would migrate` plus a line naming the
args-kind rows as LIVE rows belonging to the call-ARGUMENT dimension. The story
allowed either that or extending the migrator; the reporting half shipped.

The other half is still hand-work: converging an args row means hand-writing a
`@missingRailsArgs <ruby_call> — <reason>` tag at the call site and hand-deleting
the shard row (see `scripts/api-compare/missing-rails-args-tags.ts` for the tag
parser, and PR #6914's activesupport/encrypted-file.ts receipts for the shape).

## Converged shape

Teach `build.ts` a `--kind args` mode (or fold it into the default pass) that,
for `kind: "args"` rows with a curated reason, mints
`@missingRailsArgs <ruby_call> — <reason>` at the declaration and drops the row,
mirroring what the call-SET path already does for `@missingRailsCall`:

- reuse `reconcileFileText` / `parseJsdoc` / `renderJsdoc` with the
  `@missingRailsArgs` tag name and the `rubyArgs` key component;
- honour the seeded-placeholder rule — a row whose reason is still
  `DEFAULT_TAG_REASON` mints nothing (RFC 0083);
- the reason must open with `PERMANENT` or `CONVERGEABLE`, which
  `@missingRailsArgs` enforces and `@missingRailsCall` does not;
- keep `keyOf` collisions apart: it does not carry `kind`, so the args pass must
  filter with `rowsOfKind(baseline, "args")` exactly as the call-set pass now
  filters with `"calls"`.

Then the summary line PR #6938 added becomes a report of rows genuinely out of
scope for the invocation rather than of rows no tool moves.

## Acceptance criteria

- [ ] `parity:api:build` can migrate a curated `kind: "args"` row into a
      `@missingRailsArgs` receipt and drop it from its shard.
- [ ] A seeded-placeholder args row is left baselined and reported, as on the
      call-set path.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
