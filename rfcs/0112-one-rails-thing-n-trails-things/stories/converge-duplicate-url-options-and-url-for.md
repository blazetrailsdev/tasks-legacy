---
title: "converge-duplicate-url-options-and-url-for"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Actionpack declares `UrlOptions` twice, with divergent key casing:

- `action-dispatch/http/url.ts:18` — camelCase keys (`scriptName`, `onlyPath`,
  `tldLength`), the port of `ActionDispatch::Http::URL`
  (`actionpack/lib/action_dispatch/http/url.rb`).
- `action-dispatch/url-for.ts:7` — snake_case keys (`script_name`,
  `only_path`, `tld_length`) plus its own `urlFor`. `parity:api:extra` scores this
  file with `rubyFile === null`: no Rails file maps onto it, and its header
  claims to mirror `ActionDispatch::Http::URL` / `ActionController::UrlFor`,
  both of which have their own TS files.

So one option hash is spelled two ways and `urlFor` is implemented twice, with
the second copy in a file Rails has no counterpart for.

Surfaced by the RFC 0080 audit of `moved` interface declaration names
(`audit-moved-interface-declaration-names`, PR 5675), which tagged both
declarations `@noRailsEquivalent PERMANENT` for the _name_ collision with
`ActionDispatch::Integration::UrlOptions`. The duplication is a separate
problem the tags do not address.

## Acceptance criteria

- One `UrlOptions` remains, with camelCase keys, in the Rails-layout file
  (`action-dispatch/http/url.ts`).
- `url-for.ts` either folds into its Rails-layout counterpart or is deleted,
  with call sites moved; the now-redundant `@noRailsEquivalent` tag goes with
  it.
- `pnpm parity:api:extra` exits 0 (no stale tag).

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
