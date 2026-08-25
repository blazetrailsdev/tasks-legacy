---
title: "Converge Rack::ShowExceptions#pretty onto show_exceptions.rb:76-91 and drop the bespoke template helper"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

`Rack::ShowExceptions#pretty` builds `Rack::Request.new(env)` and a `Frame.new`
per backtrace line before rendering the template
(`vendor/rack/lib/rack/show_exceptions.rb:76-91`+). Neither is ported, so
`packages/rack/src/show-exceptions.ts:55-57` makes `pretty` a one-line delegate
to a `template` method that renders straight from `env` and the Error — a
helper Rack does not have. PR #6431 added the `pretty / new` call-set baseline
row for it.

## Converged shape

`pretty` builds the request and the per-frame objects itself, as
show_exceptions.rb:76-91 does, and renders through Rack's own template path;
the bespoke `template` helper is deleted or reduced to Rack's own spelling. The
baseline row is then deleted by hand (only-shrink).

## Acceptance criteria

- [ ] `pretty` matches show_exceptions.rb:76-91's structure, including the
      per-frame objects.
- [ ] No `template` helper without a Rack counterpart survives.
- [ ] The `pretty / new` row is deleted; `pnpm parity:api:calls` green.
