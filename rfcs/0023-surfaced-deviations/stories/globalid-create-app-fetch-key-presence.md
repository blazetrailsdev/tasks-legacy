---
title: "GlobalID.create treats an explicit app: null as omitted instead of raising"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails resolves the app for a new GlobalID with
`options.fetch(:app) { GlobalID.app }`
(`vendor/globalid/lib/global_id/global_id.rb:11`). `fetch` with a block only
falls back when the `:app` **key is absent** — an explicit `app: nil` keeps
`nil` and raises the ArgumentError on the next line.

Trails uses `opts.app ?? getApp()`
(`packages/globalid/src/global-id.ts`, in `GlobalID.create`), which treats an
explicit `app: null` / `app: undefined` as "omitted" and silently falls back to
the configured default. The same key-presence distinction is already handled
faithfully a few methods away — `SignedGlobalID.pickPurpose` uses
`options.for !== undefined ? options.for : DEFAULT_PURPOSE`, and
`pickExpiration` uses `options.expiresAt !== undefined` — so `create` is the
odd one out.

The existing test `:app option`
(`packages/globalid/src/global-id.test.ts`, `GlobalIDTest`) appears to cover
this but does not: it calls `_resetApp()` first, so the throw comes from there
being no default app at all, not from the explicit `app: null`. A regression
test must set an app first and still expect the raise.

Surfaced while converging `SignedGlobalID` onto real inheritance (PR #5951).

## Acceptance criteria

- `GlobalID.create` distinguishes an absent `app` key from an explicitly
  null/undefined one, matching Ruby's `fetch(:app) { ... }`.
- With a default app configured via `setApp`, `GlobalID.create(model, { app: null })`
  raises; `GlobalID.create(model)` still falls back to the default.
- `SignedGlobalID.create` inherits the same behaviour (it runs the same body).
- Existing globalid tests keep passing; test names unchanged.
