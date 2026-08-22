---
title: "FileHandler serves files itself instead of delegating to Rack::Files"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# FileHandler serves files itself instead of delegating to Rack::Files

## Context

Surfaced while converging `middleware/static.ts`'s call-argument rows in
PR #6688 (RFC 0106). The kwarg threading and the constructor now match Rails,
but the body underneath does not delegate the way `static.rb` does:

- `actionpack/lib/action_dispatch/middleware/static.rb:62` — Rails builds
  `@file_server = ::Rack::Files.new(@root, headers)` in
  `FileHandler#initialize` and, at `:66` (`FileHandler#call`) and `:82`
  (`#serve`), hands the request to it, merging `content_headers` onto the
  response. The port has no `@file_server` at all: `serve` reads the file with
  `getFs().readFileSync` and builds the `[status, headers, body]` triple
  itself, so `FileHandler#call` (the `attempt || @file_server.call(env)`
  fallback) does not exist either.
- `static.rb:83-84` — `serve` swaps `request.path_info` to
  `::Rack::Utils.escape_path(filepath).b` around the delegation and restores it
  in an `ensure`; the port's `serve(_env, filepath, contentHeaders)` ignores
  `env` entirely.
- `static.rb:88-90` — headers are merged onto the delegated response only when
  the status is not 304. The port has no 304 arm.
- `static.rb:141` / `:170` — `file_readable?` is
  `File.file? && File.readable?` under `File.join(@root, path.b)`, and
  `each_candidate_filepath` gets its content type from `::Rack::Mime.mime_type`.
  The port carries a bespoke 31-entry `MIME_TYPES` table and its own
  `cleanPath` in place of `::Rack::Utils.unescape_path` / `valid_path?` /
  `clean_path_info` (`static.rb:184-188`).

`packages/rack` already exists in this repo, so `Rack::Files`, `Rack::Mime` and
the `Rack::Utils` path helpers are the convergence target rather than a new
invention. Check what of that surface is ported before sizing.

## Acceptance criteria

- [ ] `FileHandler#initialize` builds a `Rack::Files` file server and `#serve`
      delegates to it, restoring `path_info` in a `finally`, with the 304 arm.
- [ ] `FileHandler#call` exists as `attempt(env) || @file_server.call(env)`
      (`static.rb:66`).
- [ ] The bespoke `MIME_TYPES` table and `cleanPath` give way to the ported
      `Rack::Mime` / `Rack::Utils` helpers, or the gap is filed against `rack`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green, and
      the existing `dispatch/static.test.ts` cases keep passing.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
