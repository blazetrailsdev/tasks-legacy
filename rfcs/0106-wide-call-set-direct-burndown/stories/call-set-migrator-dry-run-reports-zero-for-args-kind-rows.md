---
title: "parity:api:build --dry-run reports a bare 0 for args-kind rows, so live rows read as stale"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6938
claim: "2026-08-23T19:22:37Z"
assignee: "call-set-migrator-dry-run-reports-zero-for-args-kind-rows"
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/build.ts`'s `--dry-run` (`pnpm parity:api:build --package
<pkg> --file <f> --dry-run`) reports how many exclude rows would migrate to
`@missingRailsCall` receipts. It only considers call-**SET** rows: for a shard
whose remaining rows carry `kind: "args"` it reports `0 rows would migrate` and
emits no warning — which reads exactly like "these rows are stale and exclude
nothing".

That misreading cost a full story cycle. The story
`encrypted-file-call-rows-are-stale-and-exclude-nothing` (RFC 0106, PR #6914)
was written on that evidence and asserted both remaining rows in
`call-mismatches-exclude/activesupport/encrypted-file.json` were dead. They were
live `kind: "args"` rows: deleting the shard alone reds
`pnpm parity:api:calls:args` with 2 NEW rows
(`encryptor -> new`, `writing -> chomp`). The first review round repeated the
same false premise. PR #6914 shipped the correct convergence
(`@missingRailsArgs` receipts) but the tool never said the rows were args-kind.

## Converged shape

Make the dry-run honest about the rows it does not handle. When a shard the
migrator is asked about still holds rows of a kind this migrator does not
migrate, say so instead of reporting a bare `0`:

    0 of 2 rows would migrate (2 rows are kind: "args" — the call-ARGUMENT
    migrator handles those; see `pnpm parity:api:calls:args`).

Either extend the migrator to also emit `@missingRailsArgs` receipts for
`kind: "args"` rows, or keep it call-set-only and make the unhandled-kind count
explicit. Either is fine; silently reporting `0` is not.

Reference: `pnpm parity:api:calls:args` reads the same
`call-mismatches-exclude/` shards, filtering on `kind: "args"`, and writes
`scripts/api-compare/output/call-arg-mismatches.json` — the artifact the story
should have been checked against.

## Acceptance criteria

- [ ] `parity:api:build --dry-run` on a shard holding only `kind: "args"` rows
      reports those rows rather than a bare `0 rows would migrate`.
- [ ] A test in `scripts/api-compare/` covers the args-only-shard case and fails
      on the current behaviour.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
