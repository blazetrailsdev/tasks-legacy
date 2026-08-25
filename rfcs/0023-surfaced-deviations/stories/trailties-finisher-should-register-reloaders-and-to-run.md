---
title: "finisher.ts should push onto Application#reloaders and register the routes reload via Reloader#to_run"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "trailties"
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

Surfaced by the RFC 0106 `@missingRailsCall` permanence audit (PR #6855). Two
CONVERGEABLE tags on `packages/trailties/src/application/finisher.ts` could NOT
be returned to `call-mismatches-exclude/` as baseline rows, because the call-set
gate does not compare that pair — a row for a call that never flags is a STALE
row by construction. So they stay as call-site tags with nothing tracking the
work. This story is that tracker.

`railties/lib/rails/application/finisher.rb:162` (`set_routes_reloader_hook`):

    reloaders << reloader

`Rails::Application#reloaders` is not ported, so the routes reloader is never
registered on the reloaders collection.

`railties/lib/rails/application/finisher.rb:164-177`:

    app.reloader.to_run do
      require_unload_lock!
      reloader.execute
      ActiveSupport.run_load_hooks(:after_routes_loaded, self)
    end

`ActiveSupport::Reloader#to_run` is not ported, so trails executes the block once
at boot instead of re-running it on every reload.

## Converged shape

Port `Application#reloaders` (the collection the initializer pushes onto) and
`ActiveSupport::Reloader#to_run`, then spell `finisher.ts`'s initializer exactly
as `finisher.rb:159-179` does — `reloaders << reloader` and an `app.reloader.to_run`
block carrying the same three calls in the same order. Drop the two
`@missingRailsCall` tags from `finisher.ts` once the calls are made.

## Acceptance criteria

- [ ] `Application#reloaders` and `ActiveSupport::Reloader#to_run` are ported.
- [ ] `finisher.ts`'s `set_routes_reloader_hook` makes both calls at Rails' call
      sites, and the block re-runs on reload rather than executing once.
- [ ] Both `@missingRailsCall` tags are deleted from `finisher.ts` — converged,
      not rewritten.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
