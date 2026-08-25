---
title: "Relocate SKIP-mirror methods (dup/clone/inspect/toArray) into their Rails-layout files"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
  - "activerecord"
  - "activesupport"
  - "arel"
  - "rack"
  - "trailties"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #5370 deleted `TS_ALWAYS_ALLOWED`, a blanket in-file allow-set in
`scripts/api-compare/extra-surface.ts` that hid ~25 TS names in EVERY file.
SKIP-mirror names now resolve through the candidate mapping
(`skipMirrorCandidates` → `conventions.rubyMethodToTsIgnoringSkip`), which is
file-scoped: a TS `freeze` is allowed in `core.ts` because `core.rb` defines
`freeze`, and flagged anywhere Ruby doesn't.

That un-hid 44 real location divergences, now reported as "moved" extras — the
method exists in Rails, just in a different `.rb` than the TS file we declared
it in. They are correctly NOT `@noRailsEquivalent` (Rails has them), so the only
way to clear them is to move the declaration into its Rails-layout file:

- `dup`/`clone` in `activerecord/src/base.ts`, `persistence.ts`,
  `relation/where-clause.ts` — Rails' copy surface lives in `core.rb`.
- `inspect` in 9 files: `activerecord/src/attribute-methods.ts`,
  `activesupport/src/ordered-hash.ts`, `activesupport/src/values/time-zone.ts`,
  `rack/src/headers.ts`, and the actionpack metal / http-response /
  http-upload / routing-inspector / testing-integration set.
- `toArray`/`toH`/`toHash` on the collection-ish types (`relation/delegation.ts`,
  `association-relation.ts`, `disable-joins-association-relation.ts`,
  `associations/collection-proxy.ts`, `relation/batches/batch-enumerator.ts`,
  `result.ts`, `rack/src/headers.ts`, `activesupport/src/ordered-hash.ts`, …).
- `eql`, `equals`, `then`, `toF`, and `lookupCastType` on the
  abstract-mysql/sqlite3 adapters.

See #5370's description for the full per-package table. This is a burndown, not
one change: expect it to split into per-package stories once triaged. File the
first slice against whichever package's list is smallest (arel/trailties) to
establish the pattern.

## Acceptance criteria

- Pick ONE cluster (suggest the `dup`/`clone` copy surface, or the
  `toArray`/`toH`/`toHash` collection protocol) and relocate those declarations
  into the TS file matching Rails' `.rb`, per
  `project_api_compare_method_must_stay_in_rails_layout_file`.
- `pnpm parity:api:extra` moved-count for the touched packages drops by the cluster
  size; no new novel extras and no new `@noRailsEquivalent` tags.
- Remaining clusters registered as follow-up stories rather than absorbed.
