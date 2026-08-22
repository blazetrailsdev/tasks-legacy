---
title: "parity:api:build silently drops pre-existing @missingRailsCall tags it did not migrate"
status: done
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6881
claim: "2026-08-22T20:35:01Z"
assignee: "parity-api-build-must-not-drop-harvested-tags"
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:api:build --package <pkg> --file <tsFile>` (`scripts/api-compare/build.ts`,
tag writer in `scripts/api-compare/missing-rails-call-tags.ts`) rewrites the
JSDoc of every declaration in the target file, not only the ones whose baseline
rows it is migrating. In the process it **harvests** pre-existing
`@missingRailsCall` tags on untouched declarations and drops them from the file.

Observed on PR #6873 (RFC 0106 wave 5c). Migrating the nine
`connection-adapters/postgresql-adapter.json` rows printed:

    harvested (connection-adapters/postgresql-adapter.ts quote check_int_in_range): PERMANENT: …
    harvested (connection-adapters/postgresql-adapter.ts quoteString with_raw_connection): PERMANENT: …

and silently deleted both tags. Neither had a baseline row anywhere, so nothing
put them back and no gate went red — `parity:api:calls` stayed OK because the
flags they suppressed are re-derived from the tags themselves. The two receipts
documented still-live divergences (`quoteString` escapes inline via
`pgQuoteString` rather than routing through `with_raw_connection`,
`postgresql/quoting.rb:127-131`; `quote` dispatches through the
`checkIntInRange` alias) and were only caught by a human reviewer diffing for
removed lines. Restored in 406b1d409.

Silent, gate-invisible loss of reviewed deviation receipts is the failure mode
the whole `@missingRailsCall` register exists to prevent.

## Acceptance criteria

- [ ] `parity:api:build` re-emits every harvested `@missingRailsCall` tag it did
      not migrate, so a run on a file that already has tags is tag-preserving.
- [ ] If a harvested tag genuinely must be dropped (its flag no longer fires),
      the run says so explicitly and names the declaration — it never disappears
      on a `harvested …` line that reads like a successful migration.
- [ ] A regression test over a fixture file carrying both a baseline row to
      migrate AND an unrelated pre-existing tag asserts the second survives.
