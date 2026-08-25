---
title: "GID.build skips the parse path and duplicates model-id normalization"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails derives a GID's `model_name` / `model_id` from exactly one place — the
path — while trails maintains a second, parallel normalizer for the
build path only.

Rails (`vendor/globalid/lib/global_id/uri/gid.rb`):

- `:89-100` — `URI::GID.build` assembles `parts` (host/path/query) and calls
  `super parts`, i.e. `URI::Generic#initialize`.
- `:115` — that initializer runs
  `set_model_components(path) unless defined?(@model_name) && @model_id`.
- `:156` — `set_model_components(path, validate)` is the single source of
  truth: both `parse` (`:145`, with validation) and `build` land here, so a
  built GID and a parsed GID derive their components identically by
  construction.

trails (`packages/globalid/src/uri/gid.ts`):

- `GID.build` calls `buildGid(...)` to make the URI string, then hands the
  constructor a _pre-computed_ component hash whose `modelId` comes from
  `normalizeModelId(...)` — skipping the parse path entirely.
- `normalizeModelId` (`gid.ts:133-160`) duplicates `parseModelId`
  (`gid.ts:118-131`): same 20-segment cap, same empty-segment filter, same
  collapse-to-scalar rule, same `MissingModelIdError`. Its own docblock states
  it exists so "GID.build so its skip-parse path agrees with the round-trip
  through parseGid(buildGid(...))" — i.e. it is upkeep for a divergence, not a
  ported method.
- The ordering comment at `:146-149` documents a bug class this duplication
  creates: cap-then-filter vs filter-then-cap must be kept in sync by hand
  between the two functions or a 21st segment slips past the cap.

The `components?` second constructor parameter exists only to feed this
skip-parse path; Rails' initializer takes split URI parts, never pre-parsed
GID components.

## Acceptance criteria

- `GID.build` derives its components the way Rails does — through the same
  code path `GID.parse` uses — so built and parsed GIDs cannot disagree.
- `normalizeModelId` is deleted; `parseModelId` is the only model-id
  normalizer.
- The `components?` constructor parameter is removed, or reduced to the
  split-URI-parts shape Rails' `URI::Generic#initialize` actually accepts.
- Existing `uri-gid.test.ts` / `global-id.test.ts` cases pass unchanged
  (composite ids, 20-segment cap, empty-segment handling, CGI escaping) —
  they already cover both paths.
- No new extra surface; `pnpm parity:api && pnpm parity:api:extra` stay clean.
